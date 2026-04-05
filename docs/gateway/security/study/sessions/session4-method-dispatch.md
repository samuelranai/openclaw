---
title: "Session 4 — Method Dispatch and Role Authorization"
summary: "Step-by-step walkthrough of gateway method dispatch, handler table, role and scope authorization, control-plane rate limiting, and the plugin request scope — with VS Code debugger breakpoints"
read_when:
  - Following the Gateway Source Walkthrough, Session 4
  - Learning how a WS RPC call is authorized and routed to a handler
  - Investigating method security classification or adding a new gateway method
---

# Session 4 — Method Dispatch and Role Authorization

This session covers what happens after a WS message passes authentication (Session 3)
and before the business-logic handler runs. It is a short but critical layer: every
single RPC call in the gateway passes through `handleGatewayRequest` and
`authorizeGatewayMethod` — the two functions that enforce role and scope policy
at call time.

**Prerequisite:** Complete Sessions 1–3. You should know the `GatewayWsClient`
structure and how `setClient()` stores the authenticated role and scopes on it.

---

## 1. What this session covers

| File                                      | Lines | Role                                                                       |
| ----------------------------------------- | ----- | -------------------------------------------------------------------------- |
| `src/gateway/server-methods.ts`           | 157   | Dispatch table, `authorizeGatewayMethod`, `handleGatewayRequest`           |
| `src/gateway/server-methods-list.ts`      | 149   | `BASE_METHODS` registry, `GATEWAY_EVENTS`                                  |
| `src/gateway/server-methods/types.ts`     | 127   | `GatewayRequestContext`, `GatewayRequestHandlers`, `GatewayRequestOptions` |
| `src/gateway/control-plane-rate-limit.ts` | 87    | Fixed-window rate limit for `config.apply`, `config.patch`, `update.run`   |
| `src/gateway/control-plane-audit.ts`      | 41    | Audit identity for control-plane log lines                                 |
| `src/gateway/role-policy.ts`              | 24    | (already read in Session 3)                                                |
| `src/gateway/method-scopes.ts`            | 236   | (already read in Session 3)                                                |
| `src/security/dangerous-tools.ts`         | 40    | Static deny lists for HTTP and ACP surfaces                                |

---

## 2. How dispatch connects to Session 2

From Session 2, the post-handshake path in `message-handler.ts` ends with:

```typescript
// message-handler.ts (post-auth state)
await handleGatewayRequest({
  req,
  respond,
  client,
  isWebchatConnect,
  extraHandlers,
  context: buildRequestContext(),
});
```

`handleGatewayRequest` lives in `server-methods.ts`. Everything in this session
explains what happens inside that one call.

---

## 3. The method registry (`server-methods-list.ts`)

**File:** `src/gateway/server-methods-list.ts`

`BASE_METHODS` is the static list of ~110 core gateway method names. It is the
authoritative declaration of what methods exist — independent of the handler
implementations. It serves two purposes:

1. **Advertised to clients in `hello-ok`:** `listGatewayMethods()` returns
   `BASE_METHODS` merged with channel-plugin methods. This array is sent to every
   client in the `features.methods` field of the connect response. Clients use it
   to discover what the gateway supports.

2. **Method name surface:** The list is the source of truth for what names are
   expected — it does not enforce that a handler exists. A name in `BASE_METHODS`
   with no handler in `coreGatewayHandlers` will return `"unknown method"` at
   runtime.

```typescript
export function listGatewayMethods(): string[] {
  const channelMethods = listChannelPlugins().flatMap((plugin) => plugin.gatewayMethods ?? []);
  return Array.from(new Set([...BASE_METHODS, ...channelMethods]));
}
```

Channel plugin methods are merged with a `Set` to deduplicate. Plugin methods
added here are advertised to clients but only dispatched if the plugin also
registered a handler in `pluginRegistry.gatewayHandlers`.

`GATEWAY_EVENTS` (line 124) is the complementary list of push-event names the
gateway can emit. Clients use this to know which event names to subscribe to.

**Breakpoint at `server-methods-list.ts` line 119** (`listGatewayMethods` entry).
Called once during startup (Session 1 Phase 5). Inspect the return value to see
the full combined method list.

---

## 4. The handler table (`server-methods.ts`)

**File:** `src/gateway/server-methods.ts`

### `coreGatewayHandlers` (line 68)

A single flat object built from ~28 handler group imports via spread-merge:

```typescript
export const coreGatewayHandlers: GatewayRequestHandlers = {
  ...connectHandlers,
  ...logsHandlers,
  ...voicewakeHandlers,
  ...healthHandlers,
  ...channelsHandlers,
  ...chatHandlers,
  ...cronHandlers,
  ...deviceHandlers,
  ...doctorHandlers,
  ...execApprovalsHandlers,
  ...webHandlers,
  ...modelsHandlers,
  ...configHandlers,
  ...wizardHandlers,
  ...talkHandlers,
  ...toolsCatalogHandlers,
  ...toolsEffectiveHandlers,
  ...ttsHandlers,
  ...skillsHandlers,
  ...sessionsHandlers,
  ...systemHandlers,
  ...updateHandlers,
  ...nodeHandlers,
  ...nodePendingHandlers,
  ...pushHandlers,
  ...sendHandlers,
  ...usageHandlers,
  ...agentHandlers,
  ...agentsHandlers,
};
```

**Why spread-merge instead of a class hierarchy:** Each handler group is an
independent `Record<string, GatewayRequestHandler>`. Spread-merge means TypeScript
can fully type the combined object without any prototype manipulation. The
trade-off: if two groups export the same method name, the last spread wins silently.
This is a known risk — no deduplication check exists at runtime.

### `extraHandlers` override (line 135)

```typescript
const handler = opts.extraHandlers?.[req.method] ?? coreGatewayHandlers[req.method];
```

`extraHandlers` is checked first. It is assembled in `server.impl.ts` Phase 10:

```typescript
extraHandlers: {
  ...pluginRegistry.gatewayHandlers,   // plugin-provided handlers
  ...execApprovalHandlers,             // exec approval state machine
  ...pluginApprovalHandlers,           // plugin approval state machine
  ...secretsHandlers,                  // secrets.reload, secrets.resolve
}
```

**Spread order matters for collision:** Plugin handlers are first. `execApprovalHandlers`,
`pluginApprovalHandlers`, and `secretsHandlers` follow — meaning those three groups
can override plugin handlers if there is a name collision. This is intentional:
core approval and secrets operations must not be hijackable by a plugin.

`extraHandlers` then overrides `coreGatewayHandlers` at the `??` operator level,
meaning `extraHandlers` takes priority over `coreGatewayHandlers` for the same
method name. This allows the runtime handlers (secrets, approvals) to shadow core
handlers when needed.

---

## 5. Authorization: `authorizeGatewayMethod` (line 39)

This is the per-call gate. It runs for every inbound WS RPC, returns `null` on
success and an `ErrorShape` on failure. The full decision tree:

```
authorizeGatewayMethod(method, client)
  │
  ├── !client?.connect
  │     → null  (unauthenticated path, but in practice this never reaches here
  │               because message-handler.ts only dispatches post-handshake)
  │
  ├── method === "health"
  │     → null  (health is always allowed for authenticated clients regardless of role/scope)
  │
  ├── parseGatewayRole(client.connect.role ?? "operator")
  │     → null role → errorShape(INVALID_REQUEST, "unauthorized role: <raw>")
  │
  ├── isRoleAuthorizedForMethod(role, method)
  │     false → errorShape(INVALID_REQUEST, "unauthorized role: <role>")
  │     │ operator calling node-only method (e.g. node.pending.pull) → rejected
  │     │ node calling any non-node method (e.g. agent) → rejected
  │
  ├── role === "node"
  │     → null  (node-role methods have no scope requirements beyond role)
  │
  ├── scopes.includes(ADMIN_SCOPE)
  │     → null  (admin bypasses all further scope checks)
  │
  └── authorizeOperatorScopesForMethod(method, scopes)
        → { allowed: false, missingScope } → errorShape(INVALID_REQUEST, "missing scope: <scope>")
        → { allowed: true }               → null
```

**`health` special case:** `health` is the only method that passes
`authorizeGatewayMethod` even with no scopes. It is also the only method allowed
before the handshake completes (see Session 2, `message-handler.ts` pre-auth path
— `health` is explicitly passed through in the unauthenticated state check). This
enables health probes from reverse proxies that do not hold a token.

**Errors are `INVALID_REQUEST` not `FORBIDDEN`:** Unauthorized role/scope errors use
`ErrorCodes.INVALID_REQUEST`, not a distinct 403 code. The `UnauthorizedFloodGuard`
(Session 3) detects these by matching the message prefix `"unauthorized role:"`.

**Breakpoint at `server-methods.ts` line 39** (`authorizeGatewayMethod` entry).
This fires for every post-handshake RPC. Inspect:

- `method` — the method being called
- `client.connect.role` — the authenticated role
- `client.connect.scopes` — the authenticated scope list
- Return value: `null` (allowed) or an `ErrorShape` (blocked)

---

## 6. Control-plane write rate limit (`control-plane-rate-limit.ts`)

**File:** `src/gateway/control-plane-rate-limit.ts`

Three methods are in `CONTROL_PLANE_WRITE_METHODS`:

```typescript
const CONTROL_PLANE_WRITE_METHODS = new Set([
  "config.apply", // writes the full config
  "config.patch", // patches specific config fields
  "update.run", // triggers a self-update
]);
```

After `authorizeGatewayMethod` returns `null`, these methods hit a second rate
limiter — a simple fixed-window counter keyed by `deviceId|clientIp`:

```
consumeControlPlaneWriteBudget({ client })
  │
  ├── key = resolveControlPlaneRateLimitKey(client)
  │     = "deviceId|clientIp"  (or "unknown-device|unknown-ip|conn=..." as last resort)
  │
  ├── if no bucket or window expired:
  │     → new bucket { count: 1, windowStartMs: now }
  │     → { allowed: true, remaining: 2, retryAfterMs: 0 }
  │
  ├── if bucket.count >= 3:
  │     → retryAfterMs = windowStartMs + 60_000 - now
  │     → { allowed: false, remaining: 0, retryAfterMs }
  │     → gateway logs: "control-plane write rate-limited method=... actor=... device=... ip=..."
  │
  └── else:
        → bucket.count += 1
        → { allowed: true, remaining: 3 - count }
```

Constants:

- `CONTROL_PLANE_RATE_LIMIT_MAX_REQUESTS = 3` — 3 writes per window
- `CONTROL_PLANE_RATE_LIMIT_WINDOW_MS = 60_000` — 60-second fixed window

**Why a separate limiter for these three methods:** Config writes and self-updates
are high-impact mutations. A client that can call `config.patch` 100 times/second
could thrash gateway state or cause rapid restart loops. The 3/min limit forces
human-paced interaction while not blocking any legitimate use case.

**Note:** `controlPlaneBuckets` is a module-level `Map` (not per-connection). It
persists across all connections for the lifetime of the process. The key includes
`deviceId|clientIp` so a single attacker cannot bypass it by opening multiple WS
connections.

**Breakpoint at `control-plane-rate-limit.ts` line 34** (`consumeControlPlaneWriteBudget`
entry). Only fires for the three methods listed. Inspect:

- `params.client.connect.device?.id` — the authenticated device
- `params.client.clientIp` — the connection IP
- `key` — the composite rate-limit key
- Return value `allowed` and `remaining`

---

## 7. Handler invocation and plugin request scope (`server-methods.ts` line 144)

After auth and rate-limiting pass, the handler is invoked:

```typescript
const invokeHandler = () =>
  handler({
    req,
    params: (req.params ?? {}) as Record<string, unknown>,
    client,
    isWebchatConnect,
    respond,
    context,
  });

await withPluginRuntimeGatewayRequestScope({ context, client, isWebchatConnect }, invokeHandler);
```

**`withPluginRuntimeGatewayRequestScope`** wraps the handler in an
`AsyncLocalStorage` scope (`src/plugins/runtime/gateway-request-scope.ts`).
Any code running inside the handler — including deeply nested plugin calls, tool
executions, and sub-agent dispatches — can call
`getPluginRuntimeGatewayRequestScope()` to retrieve the current request context
without it being passed explicitly through every call frame.

This is the mechanism that allows a plugin's subagent to dispatch back into the
gateway (e.g. to call `sessions_spawn` or `chat.send` on behalf of the current
request) without needing a direct reference to the WS handler or the `context`
object.

**Breakpoint at `server-methods.ts` line 144** (`const invokeHandler = ...`).
When this fires:

- `handler` — the function that was looked up for `req.method`
- `req.method` — the method name
- `req.params` — the raw params object from the WS frame
- `context` — the full `GatewayRequestContext` (inspect `context.cron`,
  `context.execApprovalManager`, `context.nodeRegistry` etc.)

---

## 8. Handler structure (`types.ts`)

**File:** `src/gateway/server-methods/types.ts`

Every handler has the same signature:

```typescript
type GatewayRequestHandler = (opts: GatewayRequestHandlerOptions) => Promise<void> | void;

type GatewayRequestHandlerOptions = {
  req: RequestFrame; // { id, method, params }
  params: Record<string, unknown>; // req.params ?? {} (convenience alias)
  client: GatewayClient | null; // authenticated client (role, scopes, connId)
  isWebchatConnect: (p) => boolean; // helper to detect webchat client type
  respond: RespondFn; // send { type:"res", id, ok, payload, error }
  context: GatewayRequestContext; // full gateway runtime state
};
```

`GatewayRequestContext` is the runtime dependency bundle threaded through every
handler. It contains:

| Field                    | Type                       | Purpose                                |
| ------------------------ | -------------------------- | -------------------------------------- |
| `deps`                   | `createDefaultDeps` return | CLI-level service dependencies         |
| `cron`                   | `CronService`              | Cron scheduler for `cron.*` methods    |
| `execApprovalManager`    | `ExecApprovalManager`      | Shell command approval state           |
| `broadcast`              | fn                         | Push event to all connected clients    |
| `broadcastToConnIds`     | fn                         | Push event to specific connection IDs  |
| `nodeRegistry`           | `NodeRegistry`             | Connected mobile/remote node registry  |
| `agentRunSeq`            | `Map<string, number>`      | Per-session run sequence counter       |
| `chatAbortControllers`   | `Map<...>`                 | Active chat abort handles              |
| `logGateway`             | `SubsystemLogger`          | Structured gateway logger              |
| `subscribeSessionEvents` | fn                         | Register connId for session broadcasts |
| `dedupe`                 | `Map<string, DedupeEntry>` | Message deduplication map              |

**`GatewayClient`** carries the per-connection authenticated state:

```typescript
type GatewayClient = {
  connect: ConnectParams; // role, scopes, device, client identity
  connId?: string; // UUID for this WS connection
  clientIp?: string; // resolved client IP (may be undefined for loopback)
  canvasHostUrl?: string; // canvas host URL for this connection
  internal?: {
    allowModelOverride?: boolean; // internal flag, not settable via RPC params
  };
};
```

The `internal` field is the mechanism for granting server-side privileges that
cannot be set by clients through the RPC protocol. Code that needs to allow a
model override (e.g. an internal admin tool) sets `client.internal.allowModelOverride`
programmatically. A client cannot inject this through `ConnectParams`.

---

## 9. `sessions_spawn` and the HTTP deny list (`dangerous-tools.ts`)

**File:** `src/security/dangerous-tools.ts`

`sessions_spawn` is the agent's ability to spawn a child agent session. It is a
gateway WS method (not an RPC method like `agent`) and a tool exposed to the
LLM through the tool catalog. Two deny lists govern where it can run:

```typescript
// Blocked on POST /tools/invoke (HTTP surface)
export const DEFAULT_GATEWAY_HTTP_TOOL_DENY = [
  "sessions_spawn", // session orchestration = RCE over HTTP
  "sessions_send", // cross-session injection
  "cron", // persistent automation control plane
  "gateway", // gateway reconfiguration
  "whatsapp_login", // interactive QR scan, hangs on HTTP
];

// Always require explicit user approval in ACP (automation)
export const DANGEROUS_ACP_TOOL_NAMES = [
  "exec",
  "spawn",
  "shell",
  "sessions_spawn",
  "sessions_send",
  "gateway",
  "fs_write",
  "fs_delete",
  "fs_move",
  "apply_patch",
];
```

These lists exist at the boundary between WS and HTTP. When the LLM calls a tool
via WS (`handleGatewayRequest` → `sessionsHandlers`), scopes govern access.
When a tool arrives via `POST /tools/invoke` (HTTP, no WS session), the tool name
is checked against `DEFAULT_GATEWAY_HTTP_TOOL_DENY` before dispatch — this list
is the HTTP surface's equivalent of scope authorization.

`DANGEROUS_ACP_TOOLS` (`new Set(DANGEROUS_ACP_TOOL_NAMES)`) is checked in the
ACP (automation) pipeline to require explicit human approval even when the tool
is otherwise permitted.

**Security implication:** A client that can call `sessions_spawn` over WS with
admin scope has the ability to spawn a new agent session that inherits the parent's
tool policy. The spawned session can call tools the parent was allowed to call.
This is the scope-leakage risk noted in the STRIDE model.

---

## 10. Complete dispatch flow

```
handleGatewayRequest(opts)  (server-methods.ts:100)
  │
  ├── authorizeGatewayMethod(req.method, client)
  │     ├── health → null (always allowed)
  │     ├── bad role → errorShape INVALID_REQUEST
  │     ├── role not authorized for method → errorShape INVALID_REQUEST
  │     ├── role=node → null (no scope check)
  │     ├── ADMIN_SCOPE → null (bypass)
  │     └── authorizeOperatorScopesForMethod → null or errorShape INVALID_REQUEST
  │
  ├── [if authError] respond(false, undefined, authError)  ← return early
  │
  ├── CONTROL_PLANE_WRITE_METHODS check
  │     config.apply | config.patch | update.run
  │       → consumeControlPlaneWriteBudget()
  │         if !allowed → respond(false, ..., errorShape UNAVAILABLE rate limited)  ← return early
  │         log "control-plane write rate-limited"
  │
  ├── handler = opts.extraHandlers?.[method] ?? coreGatewayHandlers[method]
  │     if !handler → respond(false, ..., "unknown method")  ← return early
  │
  └── withPluginRuntimeGatewayRequestScope({ context, client }, invokeHandler)
        → handler({ req, params, client, isWebchatConnect, respond, context })
              ← Sessions 5–7
```

---

## 11. Exercises

### Exercise 1 — Trace a full authorized dispatch

1. Set a breakpoint at `server-methods.ts` line 39 (`authorizeGatewayMethod` entry).
2. Set a breakpoint at `server-methods.ts` line 104 (`const authError = ...`).
3. Set a breakpoint at `server-methods.ts` line 144 (`const invokeHandler = ...`).
4. In Terminal 2:
   ```bash
   OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js agent --agent dev \
     --message "Say: ping" --thinking low
   ```
5. At line 39: `method` = one of the gateway methods the CLI calls internally.
   Confirm `client.connect.role` = `"operator"` and scopes include `ADMIN_SCOPE`.
6. At line 104: `authError` should be `null` (all checks passed).
7. At line 144: `handler` is the function; `req.method` matches the method.

### Exercise 2 — Observe a scope rejection

1. This requires a client with reduced scopes. Edit the test scenario:
   set a breakpoint at line 61 (`authorizeOperatorScopesForMethod(method, scopes)`).
2. In the debugger, manually modify `scopes` to `[]` and step through.
3. Confirm `authorizeOperatorScopesForMethod` returns `{ allowed: false, missingScope }`.
4. Confirm `respond(false, undefined, authError)` is called and the handler is skipped.

### Exercise 3 — Observe the handler table lookup

1. Set a breakpoint at `server-methods.ts` line 135
   (`const handler = opts.extraHandlers?.[req.method] ?? coreGatewayHandlers[req.method]`).
2. Trigger a message via the CLI.
3. Inspect `opts.extraHandlers` — note the keys (secrets, approvals handlers).
4. Inspect `coreGatewayHandlers` — it is the large flat merged object.
5. Confirm the looked-up `handler` is a function (not `undefined`).

### Exercise 4 — Control-plane rate limit

1. Set a breakpoint at `control-plane-rate-limit.ts` line 34 (`consumeControlPlaneWriteBudget`).
2. From the CLI, run:
   ```bash
   OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js config set \
     agents.defaults.model.primary openai/gpt-5.4
   ```
   This triggers `config.apply` or `config.patch` on the gateway.
3. At the breakpoint: inspect `key` and `bucket`. On first call, `bucket` is
   `undefined` (not yet in the map). Confirm `allowed=true` and `remaining=2`.

### Exercise 5 — Inspect `GatewayRequestContext`

1. Set a breakpoint at `server-methods.ts` line 144.
2. Inspect `context`. Expand `context.nodeRegistry` — should show an empty registry
   with no connected nodes in dev.
3. Expand `context.cron` — inspect the cron service state.
4. Expand `context.execApprovalManager` — note it may be `undefined` if exec
   approvals are not configured in dev.

---

## 12. Useful breakpoint summary

| What to catch            | File                          | Line | Inspect                                                  |
| ------------------------ | ----------------------------- | ---- | -------------------------------------------------------- |
| Method registry built    | `server-methods-list.ts`      | 119  | Return value — all method names                          |
| Auth gate entry          | `server-methods.ts`           | 39   | `method`, `client.connect.role`, `client.connect.scopes` |
| Auth gate result         | `server-methods.ts`           | 104  | `authError` (null = allowed)                             |
| Control-plane rate check | `control-plane-rate-limit.ts` | 34   | `key`, `bucket`, `allowed`, `remaining`                  |
| Handler lookup           | `server-methods.ts`           | 135  | `handler` (function or undefined)                        |
| Handler invocation       | `server-methods.ts`           | 144  | `req.method`, `req.params`, `context`                    |
| Scope authorization      | `method-scopes.ts`            | 214  | `method`, `scopes`, `requiredScope`                      |
| Role → method check      | `role-policy.ts`              | 18   | `role`, `method`, return value                           |

---

## See Also

- [Session 3 — Authentication](session3-authentication.md)
- [Session 5 — Agent Loop and Chat](gateway-source-walkthrough.md)
- [Gateway Source Walkthrough](gateway-source-walkthrough.md)
- [Attack Surface Map](attack-surface-map.md)
- [STRIDE Threat Model](stride-threat-model.md)
