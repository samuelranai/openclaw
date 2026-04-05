---
title: "Session 6 — Tool Dispatch"
summary: "Step-by-step walkthrough of gateway tool dispatch: HTTP POST /tools/invoke, the tool policy pipeline, node.invoke with APNS wake + command allowlist, exec approval gating, and the system.run approval sanitization layer — with VS Code debugger breakpoints"
read_when:
  - Following the Gateway Source Walkthrough, Session 6
  - Learning how the gateway decides which tools an agent (or HTTP caller) may invoke
  - Investigating tool policy, exec approval, or node command allowlist behavior
---

# Session 6 — Tool Dispatch

This session covers three distinct tool dispatch surfaces that all exist in the
same gateway process: direct HTTP `POST /tools/invoke`, node-dispatched tool
invocations via `node.invoke`, and the exec approval system that gates shell
command execution. All three share the same tool policy pipeline but have
different outer authorization and transport layers.

**Prerequisite:** Complete Sessions 1–5. You should know `handleGatewayRequest`
(Session 4), how `GatewayRequestContext` is built, and how `ResolvedGatewayAuth`
is computed at startup.

---

## 1. What this session covers

| File                                             | Lines | Role                                                                     |
| ------------------------------------------------ | ----- | ------------------------------------------------------------------------ |
| `src/gateway/tools-invoke-http.ts`               | 365   | HTTP `POST /tools/invoke` handler                                        |
| `src/agents/tool-policy-pipeline.ts`             | 186   | Ordered policy reduction — `applyToolPolicyPipeline`                     |
| `src/gateway/server-methods/nodes.ts`            | 1194  | `node.invoke`, APNS wake, pending action queue, node pairing             |
| `src/gateway/node-command-policy.ts`             | 217   | `resolveNodeCommandAllowlist`, `isNodeCommandAllowed`                    |
| `src/gateway/exec-approval-manager.ts`           | 210   | In-flight exec approval records with expiry and consume                  |
| `src/gateway/node-invoke-system-run-approval.ts` | 303   | `sanitizeSystemRunParamsForForwarding` — approval guard for `system.run` |
| `src/security/dangerous-tools.ts`                | 40    | `DEFAULT_GATEWAY_HTTP_TOOL_DENY`, `DANGEROUS_ACP_TOOL_NAMES`             |

---

## 2. How the three surfaces connect

```
                    ┌─────────────────────────────────────────────┐
                    │         Tool Policy Pipeline                 │
                    │  applyToolPolicyPipeline(tools, steps)       │
                    │  (tool-policy-pipeline.ts)                   │
                    └──────────────┬──────────────────────────────┘
                                   │ same function, different callers
              ┌────────────────────┼───────────────────────┐
              ▼                    ▼                       ▼
  HTTP POST /tools/invoke     WS node.invoke          LLM tool call
  (tools-invoke-http.ts)      (nodes.ts)             (pi agent runtime)
  bearer token auth            WS auth + role          in-process, same policy
  + DEFAULT_GATEWAY_HTTP_TOOL_DENY + node command allowlist
```

All three paths call `resolveEffectiveToolPolicy` from `pi-tools.policy.ts` to
build the same layered policy input, then pass it through
`applyToolPolicyPipeline`. The surfaces differ in: outer auth mechanism,
additional deny lists, and whether a human approval step can be required.

---

## 3. The tool policy pipeline (`tool-policy-pipeline.ts`)

**File:** `src/agents/tool-policy-pipeline.ts`

`applyToolPolicyPipeline` reduces a list of policy steps — ordered from most
specific to least specific — into a filtered tool list:

```typescript
for (const step of steps) {
  if (!step.policy) continue;
  let policy = maybeStripPluginOnlyAllowlist(step.policy, pluginGroups, coreToolNames);
  const expanded = expandPolicyWithPluginGroups(policy, pluginGroups);
  filtered = expanded ? filterToolsByPolicy(filtered, expanded) : filtered;
}
```

**Evaluation order** (from `buildDefaultToolPolicyPipelineSteps`, lines 36–89):

| Step | Config key                           | Description                                              |
| ---- | ------------------------------------ | -------------------------------------------------------- |
| 1    | `tools.profile`                      | Active tool profile (e.g. `"minimal"`)                   |
| 2    | `tools.byProvider.profile`           | Provider-specific profile override                       |
| 3    | `tools.allow`                        | Global allow/deny list                                   |
| 4    | `tools.byProvider.allow`             | Global by-provider allow/deny                            |
| 5    | `agents.<id>.tools.allow`            | Per-agent allow/deny                                     |
| 6    | `agents.<id>.tools.byProvider.allow` | Per-agent by-provider allow/deny                         |
| 7    | Group policy                         | Resolved from `messageChannel` / `accountId` header      |
| 8    | Subagent policy                      | `subagentToolPolicy` (only if session key is a subagent) |

**`stripPluginOnlyAllowlist`:** If a step has `stripPluginOnlyAllowlist: true`
and the allowlist contains only names for unavailable plugins (none of the
core tool names match), the allowlist is treated as empty rather than blocking
all core tools. This prevents a misconfigured plugin name from accidentally
locking out all tools.

**Each step's `policy` structure:**

```typescript
type ToolPolicyLike = {
  allow?: string[]; // if set, only these tools pass through
  deny?: string[]; // if set, these tools are removed
};
```

Steps are sequential: each step further restricts the passing set. A deny at any
step removes permanently; an allow at a step restricts to that subset.

---

## 4. HTTP tool dispatch (`tools-invoke-http.ts`)

**File:** `src/gateway/tools-invoke-http.ts`

This file handles the `POST /tools/invoke` REST endpoint — a non-WebSocket
surface that lets bearer-token callers invoke a single tool by name.

### 4a. Auth (lines 159–168)

```typescript
const ok = await authorizeGatewayBearerRequestOrReply({ req, res, auth, rateLimiter });
if (!ok) return true;
```

`authorizeGatewayBearerRequestOrReply` validates the `Authorization: Bearer ...`
header against `ResolvedGatewayAuth`. This is the same `auth` object computed
once at startup (Session 1). There is no session/role context — any bearer that
passes auth can call any non-denied tool for the resolved `sessionKey`.

### 4b. Session key and channel hints (lines 208–217)

```typescript
const sessionKey =
  !rawSessionKey || rawSessionKey === "main" ? resolveMainSessionKey(cfg) : rawSessionKey;
const messageChannel = normalizeMessageChannel(getHeader(req, "x-openclaw-message-channel") ?? "");
```

The `x-openclaw-message-channel` header influences `resolveGroupToolPolicy`.
**Security risk (from walkthrough):** an untrusted caller that controls this
header can shift into a different policy group, potentially accessing a less
restrictive tool set. The header is not validated against any allowed list.

### 4c. Policy construction (lines 220–297)

```
resolveEffectiveToolPolicy({ config, sessionKey })
  → returns: agentId, globalPolicy, agentPolicy, profile, groupPolicy, ...

applyToolPolicyPipeline({ tools: allTools, steps: [
  ...buildDefaultToolPolicyPipelineSteps(...),
  { policy: subagentPolicy, label: "subagent tools.allow" },
]})
```

### 4d. HTTP-specific deny list (lines 300–308)

```typescript
const defaultGatewayDeny = DEFAULT_GATEWAY_HTTP_TOOL_DENY.filter(
  (name) => !gatewayToolsCfg?.allow?.includes(name),
);
const gatewayDenySet = new Set([...defaultGatewayDeny, ...(gatewayToolsCfg?.deny ?? [])]);
const gatewayFiltered = subagentFiltered.filter((t) => !gatewayDenySet.has(t.name));
```

After the policy pipeline, this additional deny list removes tools that are
specifically dangerous over HTTP (see `dangerous-tools.ts`):

| Denied tool      | Reason                                                  |
| ---------------- | ------------------------------------------------------- |
| `sessions_spawn` | Session orchestration — spawning agents remotely is RCE |
| `sessions_send`  | Cross-session injection                                 |
| `cron`           | Persistent automation control plane                     |
| `gateway`        | Gateway reconfiguration                                 |
| `whatsapp_login` | Requires interactive terminal, hangs on HTTP            |

Each can be individually un-denied via `gateway.tools.allow` in config.

### 4e. Before-tool-call hook and execution (lines 327–358)

```typescript
const hookResult = await runBeforeToolCallHook({ toolName, params: toolArgs, toolCallId, ctx });
if (hookResult.blocked) {
  sendJson(res, 403, {
    ok: false,
    error: { type: "tool_call_blocked", message: hookResult.reason },
  });
  return true;
}
const result = await tool.execute?.(toolCallId, hookResult.params);
sendJson(res, 200, { ok: true, result });
```

`runBeforeToolCallHook` runs any registered pre-call hooks (loop detection, etc.).
The `toolCallId` is `"http-" + Date.now()` — it is not cryptographically unique
and could collide under high throughput, but it is only used for logging context.

---

## 5. Node command dispatch (`server-methods/nodes.ts`)

**File:** `src/gateway/server-methods/nodes.ts`

Nodes are remote execution extensions (mobile device, second machine). The
gateway acts as an orchestrator: it sends commands to nodes, waits for results,
and handles node unavailability via APNs wake.

### 5a. `node.invoke` entry (lines 900–1128)

```
node.invoke({ nodeId, command, params, timeoutMs, idempotencyKey })
  ├── reject system.execApprovals.* commands (hardcoded deny, line 926)
  ├── context.nodeRegistry.get(nodeId)        ← is node currently connected?
  ├── if not connected → APNs wake flow (see §5b)
  ├── resolveNodeCommandAllowlist(cfg, nodeSession)
  ├── isNodeCommandAllowed({ command, declaredCommands, allowlist })
  ├── sanitizeNodeInvokeParamsForForwarding(...)   ← sanitize/approval-gate params
  └── context.nodeRegistry.invoke(...)        ← forward to node over WS
```

### 5b. APNs wake flow (lines 940–1020)

When the node is not connected, the gateway attempts to wake it via Apple Push
Notification Service (APNs) before failing:

```
Stage 1: maybeWakeNodeWithApns(nodeId)     ← throttle: 15s between wake attempts
  └── wait for reconnect (NODE_WAKE_RECONNECT_WAIT_MS = 3s)
Stage 2: if still disconnected and wake was available →
         maybeWakeNodeWithApns(nodeId, { force: true })  ← bypass throttle
  └── wait for reconnect (NODE_WAKE_RECONNECT_RETRY_WAIT_MS = 12s)
Stage 3: if still disconnected →
         maybeSendNodeWakeNudge(nodeId)   ← alert (throttle: 10 min)
         → respond UNAVAILABLE: NOT_CONNECTED
```

**Transport:** Wake can be direct (gateway has APNs credentials) or via a relay
service. Both paths are resolved from environment variables.

### 5c. iOS foreground queue (lines 1064–1108)

Canvas, camera, screen, and talk commands fail with `NODE_BACKGROUND_UNAVAILABLE`
when iOS is in the background. The gateway queues them as pending actions and
resends an APNs wake:

```typescript
if (shouldQueueAsPendingForegroundAction({ platform, command, error })) {
  enqueuePendingNodeAction({ nodeId, command, paramsJSON, idempotencyKey });
  await maybeWakeNodeWithApns(nodeId);
  respond(false, undefined, errorShape(QUEUED_UNTIL_FOREGROUND, ...));
}
```

Node clients poll for queued actions via `node.pending.pull`, execute them when
in the foreground, and acknowledge via `node.pending.ack`.

### 5d. Node command allowlist (`node-command-policy.ts`)

**File:** `src/gateway/node-command-policy.ts`

`resolveNodeCommandAllowlist` builds the set of allowed commands for a node
based on its declared platform:

```
platform    → base command set
ios         → canvas, camera, location, device, contacts, calendar, reminders, photos, motion, system.notify
android     → canvas, camera, location, notifications, device, contacts, calendar, callLog, reminders, sms, photos, motion
macos       → canvas, camera, location, device, contacts, calendar, reminders, photos, motion + system.run + system.notify
linux       → system.run + system.notify + browser.proxy
windows     → system.run + system.notify + browser.proxy
unknown     → canvas, camera, location, system.notify (fail-safe minimal)
```

`DEFAULT_DANGEROUS_NODE_COMMANDS` (line 68) — high-risk commands
(`camera.snap`, `screen.record`, `contacts.add`, `sms.send`, etc.) — are **not**
in the base sets. They must be explicitly added via `gateway.nodes.allowCommands`.

`isNodeCommandAllowed` performs two checks (line 196):

1. Command is in the computed allowlist
2. Command is in the node's `declaredCommands` list (what the node itself said it supports)

Both must pass. A command in the allowlist that the node never declared fails with
`"command not declared by node"`.

---

## 6. Exec approval system (`exec-approval-manager.ts`)

**File:** `src/gateway/exec-approval-manager.ts`

`ExecApprovalManager<TPayload>` is a generic in-memory registry for approval
records. The same class is used for both local exec approvals and node-side
`system.run` approvals.

### 6a. Approval lifecycle

```
1. agent proposes exec command
2. exec tool creates approval record:
   manager.create(request, timeoutMs)  →  ExecApprovalRecord { id, request, expiresAtMs }
3. manager.register(record, timeoutMs) →  Promise<ExecApprovalDecision | null>
   └── stores entry; setTimeout to auto-expire → resolve(null)
4. UI receives broadcast, user decides
5. exec.approval.resolve RPC →  manager.resolve(recordId, "allow-once" | "allow-always", resolvedBy)
   └── clears timer, sets decision, resolves promise, 15s grace window before cleanup
6. exec tool awaits the promise:
   if decision === "allow-once"  → consumeAllowOnce() (atomic one-shot)
   if decision === "allow-always" → proceed
   if null (timeout / expired)  → deny
```

**`consumeAllowOnce` (line 154):** Atomically clears the `"allow-once"` decision
so the same approval record cannot be replayed during the 15-second resolved-entry
grace window.

**`lookupPendingId` (line 178):** Supports prefix matching of approval IDs — the
same mechanism used by the CLI to resolve partial UUIDs. Returns `exact`,
`prefix`, `ambiguous`, or `none`.

**Grace period:** After `resolve()` or `expire()`, the entry stays in the map
for `RESOLVED_ENTRY_GRACE_MS = 15,000 ms`. This allows in-flight `awaitDecision`
calls on reconnect to find the already-resolved entry.

### 6b. Identity binding (in `ExecApprovalRecord`)

```typescript
requestedByConnId?: string | null;
requestedByDeviceId?: string | null;
requestedByClientId?: string | null;
```

These are set when the approval record is created and checked by
`sanitizeSystemRunParamsForForwarding` to ensure the approval can only be used by
the same client (or device) that requested it. Device ID is preferred over connId
because it is stable across reconnects.

---

## 7. `system.run` approval gate (`node-invoke-system-run-approval.ts`)

**File:** `src/gateway/node-invoke-system-run-approval.ts`

`sanitizeSystemRunParamsForForwarding` is called inside `node.invoke` when the
command is `system.run`. It strips or validates the `approved` and
`approvalDecision` fields that a node host uses to execute shell commands without
showing an approval dialog.

### 7a. Why this guard exists (line 91–95)

> Gate `system.run` approval flags (`approved`, `approvalDecision`) behind a real
> `exec.approval.*` record. This prevents users with only `operator.write` from
> bypassing node-host approvals by injecting control fields into `node.invoke`.

Without this guard, any `operator.write` caller could send:

```json
{ "command": "system.run", "params": { "command": "rm -rf /", "approved": true } }
```

and the node host would execute it without prompting.

### 7b. Sanitization flow

```
sanitizeSystemRunParamsForForwarding({ nodeId, rawParams, client, execApprovalManager })
  ├── if no approval override (approved/approvalDecision absent) →
  │     pickSystemRunParams(obj)  ← allowlist-only field copy; return
  └── if approval override requested:
        ├── require params.runId
        ├── manager.getSnapshot(runId) → must exist and not be expired
        ├── verify nodeId matches approval.request.nodeId
        ├── verify client device/connId matches requestedByDeviceId/requestedByConnId
        ├── evaluateSystemRunApprovalMatch(...)  ← command/cwd/agentId must match approval
        └── based on decision:
              "allow-once"  → consumeAllowOnce() + re-add fields
              "allow-always" → re-add fields
              timedOut + allow-once + clientHasApprovals → allow (fallback)
              else → systemRunApprovalRequired error
```

**`pickSystemRunParams` (line 67):** Allowlist-only field copy — only the fields
listed explicitly are forwarded. Future internal control fields cannot be smuggled
through the gateway even if a new field is added to the node protocol.

**`evaluateSystemRunApprovalMatch` (from `node-invoke-system-run-approval-match.ts`):**
Verifies that the actual command/cwd/agentId/sessionKey at execution time matches
what was recorded in the approval request. This prevents a race where an approval
for command A is replayed for command B.

---

## 8. Full flow: `POST /tools/invoke`

```
HTTP POST /tools/invoke
  │  Headers: Authorization: Bearer <token>
  │  Body: { tool: "exec", args: { command: "ls" }, sessionKey: "main:..." }
  │
  ▼
handleToolsInvokeHttpRequest (tools-invoke-http.ts)
  ├── authorizeGatewayBearerRequestOrReply → 401 if invalid bearer
  ├── readJsonBodyOrError → 400 if > 2 MB or invalid JSON
  ├── resolveSessionKeyFromBody → "main:..." or resolveMainSessionKey(cfg)
  ├── resolveEffectiveToolPolicy({ config, sessionKey })
  ├── createOpenClawTools({ sessionKey, ... })   ← full tool list with policy hints
  ├── applyToolPolicyPipeline(allTools, steps)    ← ordered policy reduction
  ├── gatewayFiltered = subagentFiltered.filter(t => !gatewayDenySet.has(t.name))
  │     ← remove sessions_spawn, sessions_send, cron, gateway, whatsapp_login
  ├── find tool by name → 404 if not found after filtering
  ├── runBeforeToolCallHook → 403 if blocked
  └── tool.execute(toolCallId, args) → 200 { ok: true, result }
                                     → 400/403 if ToolInputError/ToolAuthorizationError
                                     → 500 if unexpected throw
```

---

## 9. Full flow: `node.invoke system.run` with approval

```
WS client sends: node.invoke { nodeId: "iphone-1", command: "system.run",
                                params: { command: "open -a Safari", approved: true,
                                          approvalDecision: "allow-once", runId: "uuid-abc" } }
  │
  ▼
nodeHandlers["node.invoke"]
  ├── validate params
  ├── resolve node session (wake via APNs if needed)
  ├── resolveNodeCommandAllowlist(cfg, nodeSession)
  ├── isNodeCommandAllowed("system.run", nodeSession.commands, allowlist)
  ├── sanitizeNodeInvokeParamsForForwarding({ nodeId, command, rawParams, client, execApprovalManager })
  │     └── sanitizeSystemRunParamsForForwarding(...)
  │           ├── detects approved=true / approvalDecision present
  │           ├── getSnapshot("uuid-abc")   ← must exist in execApprovalManager
  │           ├── verify nodeId + device identity match
  │           ├── evaluateSystemRunApprovalMatch → command/cwd must match snapshot
  │           ├── consumeAllowOnce("uuid-abc")   ← atomic one-shot
  │           └── returns { ok: true, params: { command: [...], approved: true, approvalDecision: "allow-once" } }
  └── context.nodeRegistry.invoke({ nodeId, command, params, idempotencyKey })
        └── sends to node over WS; node executes and returns result
```

---

## 10. Key constants

| Constant                     | Value       | Location                     |
| ---------------------------- | ----------- | ---------------------------- |
| HTTP body size limit         | 2 MB        | `tools-invoke-http.ts:40`    |
| Node wake throttle           | 15,000 ms   | `nodes.ts:57`                |
| Node wake reconnect wait 1   | 3,000 ms    | `nodes.ts:54`                |
| Node wake reconnect wait 2   | 12,000 ms   | `nodes.ts:55`                |
| Node wake poll interval      | 150 ms      | `nodes.ts:56`                |
| Node wake nudge throttle     | 10 min      | `nodes.ts:58`                |
| Pending action TTL           | 10 min      | `nodes.ts:59`                |
| Pending action max per node  | 64          | `nodes.ts:60`                |
| Approval grace after resolve | 15,000 ms   | `exec-approval-manager.ts:8` |
| Tool policy warning cache    | 256 entries | `tool-policy-pipeline.ts:12` |

---

## 11. Breakpoint summary

| Breakpoint file + line                               | What you will see                                                    |
| ---------------------------------------------------- | -------------------------------------------------------------------- |
| `src/gateway/tools-invoke-http.ts:159`               | Bearer auth gate — inspect `auth` object and `ok` result             |
| `src/gateway/tools-invoke-http.ts:230`               | `resolveEffectiveToolPolicy` call — inspect resolved policy layers   |
| `src/gateway/tools-invoke-http.ts:274`               | `applyToolPolicyPipeline` call — inspect `allTools` before filtering |
| `src/gateway/tools-invoke-http.ts:308`               | `gatewayFiltered` — final tool list after HTTP deny list             |
| `src/gateway/tools-invoke-http.ts:327`               | `runBeforeToolCallHook` — inspect hook result / blocked reason       |
| `src/agents/tool-policy-pipeline.ts:109`             | Pipeline loop — inspect each step as policy is applied               |
| `src/gateway/server-methods/nodes.ts:940`            | `node.invoke` node lookup — is node in registry?                     |
| `src/gateway/server-methods/nodes.ts:1022`           | `isNodeCommandAllowed` — inspect command vs allowlist                |
| `src/gateway/server-methods/nodes.ts:1039`           | `sanitizeNodeInvokeParamsForForwarding` call                         |
| `src/gateway/node-command-policy.ts:178`             | `resolveNodeCommandAllowlist` — inspect platform → base commands     |
| `src/gateway/exec-approval-manager.ts:59`            | `register` — approval record registered, promise created             |
| `src/gateway/exec-approval-manager.ts:103`           | `resolve` — decision recorded, promise resolved                      |
| `src/gateway/exec-approval-manager.ts:154`           | `consumeAllowOnce` — atomic one-shot consume                         |
| `src/gateway/node-invoke-system-run-approval.ts:150` | Snapshot lookup — inspect runId → record                             |
| `src/gateway/node-invoke-system-run-approval.ts:255` | `evaluateSystemRunApprovalMatch` — command/cwd verification          |

---

## 12. Exercises

### Exercise 1 — Trace HTTP tool filtering

Set a breakpoint at `tools-invoke-http.ts:308` (`gatewayFiltered`). Invoke
`POST /tools/invoke` with body `{ "tool": "sessions_spawn" }`. Observe that
`sessions_spawn` is absent from `gatewayFiltered` even if the policy pipeline
left it in, because `DEFAULT_GATEWAY_HTTP_TOOL_DENY` removes it afterward.
Then try `{ "tool": "exec" }`. Observe it survives the gateway deny list but
may be filtered by the policy pipeline depending on config.

### Exercise 2 — Policy pipeline step-through

Set a breakpoint at `tool-policy-pipeline.ts:109` (the `for` loop). Send an
HTTP request that has a non-default tool profile configured. Step through each
iteration: observe which steps have a `policy` object, when `stripPluginOnlyAllowlist`
fires, and how the `filtered` array shrinks (or doesn't) at each step.

### Exercise 3 — Node command allowlist

Set a breakpoint at `node-command-policy.ts:178` (`resolveNodeCommandAllowlist`).
Trigger a `node.invoke` call from a connected iOS node. Inspect `platformId`,
the `base` set, and the final `allow` set. Verify `camera.snap` is absent.
Then add `camera.snap` to `gateway.nodes.allowCommands` in config and repeat —
observe it appears in `allow`.

### Exercise 4 — Approval record lifecycle

Set breakpoints at `exec-approval-manager.ts:59` (`register`) and line 103
(`resolve`). Trigger a shell command that requires approval. Observe the record
created with `expiresAtMs = now + timeoutMs`. Accept the approval in the UI.
Observe `resolve()` called, timer cleared, and promise resolved. Then check
`consumeAllowOnce` at line 154: verify that calling it a second time returns `false`.

### Exercise 5 — `system.run` approval guard

Set a breakpoint at `node-invoke-system-run-approval.ts:150` (snapshot lookup).
Send a `node.invoke` for `system.run` with `approved: true` and an invented
`runId: "fake-id"`. Observe the guard returns `UNKNOWN_APPROVAL_ID` before
reaching the node. Then use a real pending runId from an in-progress approval —
observe it passes identity checks and proceeds.

---

## 13. Security observations

**HTTP channel header is untrusted (`tools-invoke-http.ts:213`):**
`x-openclaw-message-channel` influences `resolveGroupToolPolicy`. A caller that
controls this header can influence which group policy is applied. The header is
not validated against any configured list of allowed channels — any string is
accepted and normalized.

**`pickSystemRunParams` prevents field smuggling (`node-invoke-system-run-approval.ts:67`):**
The allowlist-only field copy in `pickSystemRunParams` means that even if the
node protocol gains a new internal control field in the future, it cannot be
injected by a gateway client unless the allowlist is explicitly updated.

**`system.execApprovals.*` hardcoded deny (`nodes.ts:926`):**
`node.invoke` rejects `system.execApprovals.get` and `system.execApprovals.set`
with a hard error before any policy check runs. These methods exist on the node
host as internal protocol — they must only be called via the `exec.approvals.node.*`
gateway methods which have separate authorization.

**`DEFAULT_DANGEROUS_NODE_COMMANDS` are opt-in (`node-command-policy.ts:68`):**
`camera.snap`, `screen.record`, `sms.send`, `contacts.add` and similar
high-sensitivity commands are absent from all platform base sets. They require
explicit `gateway.nodes.allowCommands` addition by the operator — they are never
available by default even for a fully paired node.

**Approval record is device-bound, not session-bound:**
The `requestedByDeviceId` check in `sanitizeSystemRunParamsForForwarding` (line 194)
means an approval created by device A cannot be used by device B even if both
are operator-scope clients on the same gateway. Approval grants are per-device,
not per-role.
