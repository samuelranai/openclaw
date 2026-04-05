---
title: "Gateway Source Code Walkthrough"
summary: "Layer-by-layer reading plan for the OpenClaw Gateway — startup, HTTP, WebSocket, auth, method dispatch, agent loop, tool dispatch, sessions, and channels"
read_when:
  - Starting a deep-dive into the gateway codebase
  - Orienting to a specific gateway layer before making changes
  - Building a mental model of the full request lifecycle
---

# Gateway Source Code Walkthrough

This document is a structured reading plan for the OpenClaw Gateway source code.
It is organized as seven reading sessions, each covering one architectural layer.
Each session identifies the files to read, what to look for, and how the layer
connects to the next. Sessions build on each other — read them in order the first time.

The gateway lives entirely under `src/gateway/`. The total reading surface is roughly
**12,000 lines** across ~30 key files. At a focused pace, one session per sitting.

---

## Architecture Overview (Read First)

Before opening any file, hold this mental model:

```
External world
    │
    ▼
[HTTP + WebSocket server]          ← src/gateway/server/http-listen.ts
    │                                  src/gateway/server/ws-connection.ts
    ├── HTTP surfaces
    │     ├── POST /tools/invoke    ← src/gateway/tools-invoke-http.ts
    │     ├── POST /v1/...          ← src/gateway/openai-http.ts
    │     └── Control UI            ← src/gateway/control-ui.ts
    │
    └── WebSocket
          ├── Handshake + auth      ← src/gateway/server/ws-connection/
          └── Method dispatch       ← src/gateway/server-methods.ts
                └── Handlers        ← src/gateway/server-methods/*.ts
                      ├── agent     → runs embedded agent loop
                      ├── chat      → streams agent events to client
                      ├── sessions  → reads/writes JSONL transcripts
                      └── send      → message injection

[Channel adapters] ─────────────────────────────────────────────────────────►
    Telegram / Discord / Slack / Signal / iMessage / WhatsApp / extensions
    src/telegram/, src/discord/, src/slack/, extensions/
    All deliver through the same agent.command() path
```

**One process, three entry types:** WebSocket RPC, HTTP REST, and channel-adapter
callbacks — all converging on the same `agentCommand` / tool execution core.

---

## Session 1 — Startup and Wiring (`server.impl.ts`)

**Files:** `src/gateway/server.impl.ts` (1496 lines)

This is the gateway's `main()`. Read it top-to-bottom once, not line by line — you are
mapping the startup sequence, not reading the implementation.

**What to track as you scan:**

| Step | What happens            | Where                                                             |
| ---- | ----------------------- | ----------------------------------------------------------------- |
| 1    | Config load + migration | `loadConfig`, `migrateLegacyConfig`                               |
| 2    | Auth resolution         | `ensureGatewayStartupAuth`, `mergeGatewayAuthConfig`              |
| 3    | Secrets snapshot        | `prepareSecretsRuntimeSnapshot`, `activateSecretsRuntimeSnapshot` |
| 4    | Plugin bootstrap        | `loadGatewayStartupPlugins`, `setActivePluginRegistry`            |
| 5    | HTTP server start       | `startGatewaySidecars` → `listenGatewayHttpServer`                |
| 6    | WebSocket handlers      | `attachGatewayWsHandlers`                                         |
| 7    | Channel adapters        | `createChannelManager` → per-channel `start()`                    |
| 8    | Maintenance timers      | `startGatewayMaintenanceTimers`                                   |
| 9    | Tailscale exposure      | `startGatewayTailscaleExposure`                                   |

**Key observations to note:**

- `resolvedAuth` is computed once at startup and passed down to every auth check.
  There is no per-request auth re-resolution from config — a config reload cycles
  the auth object explicitly (`startGatewayConfigReloader`).
- Two rate limiters are created: `rateLimiter` (general, loopback-exempt by default)
  and `browserRateLimiter` (browser-origin WS, loopback exemption disabled).
- Plugin load is two-phase: `loadGatewayStartupPlugins` at boot, then
  `reloadDeferredGatewayPlugins` after channel manager is ready.
- `ExecApprovalManager` is instantiated here and threaded through to all places
  that execute shell commands.

**Connection to next session:** `startGatewaySidecars` calls `listenGatewayHttpServer`
and sets up the HTTP server object that also hosts the WebSocket upgrade.

---

## Session 2 — HTTP and WebSocket Transport

**Files:**

- `src/gateway/server/http-listen.ts` (~30 lines)
- `src/gateway/server-http.ts` (surveyable via imports)
- `src/gateway/server/ws-connection.ts` (344 lines)
- `src/gateway/server/ws-connection/message-handler.ts` (1248 lines)

### 2a. HTTP listen (`http-listen.ts`)

Minimal: binds `httpServer.listen(port, bindHost)` with retry on `EADDRINUSE`.
The retry loop (4 attempts, 500ms interval) is the only availability concern here —
if another process holds the port, the gateway delays startup silently.

### 2b. WebSocket connection lifecycle (`ws-connection.ts`)

This file owns the per-connection lifecycle. Read the `attachGatewayWsConnectionHandler`
function. The sequence is:

```
ws 'connection' event
  ├── sanitize headers (replaceControlChars, truncate)
  ├── preauthConnectionBudget.tryAcquire()   ← flood guard
  ├── get client IP (resolveClientIp)
  ├── upsertPresence()                        ← presence tracking
  └── attachGatewayWsMessageHandler()         ← hands off to message-handler.ts
```

**Key security detail:** `preauthConnectionBudget` limits simultaneous
pre-authenticated connections. This is the DoS guard against connection floods
before auth completes.

Header sanitization (`replaceControlChars` + Unicode format-character stripping +
`truncateUtf16Safe(LOG_HEADER_MAX_LEN)`) happens only for logging — headers are
not otherwise filtered at this layer.

### 2c. Message handler (`ws-connection/message-handler.ts`, 1248 lines)

The most complex file in this session. It implements the full WS message lifecycle:
**pre-auth handshake → `connect` RPC → authenticated method dispatch**.

Read in three passes:

**Pass 1 — The handshake state machine:**

```
ws 'message' event
  └── if not yet connected:
        ├── expect { method: "connect", params: { token?, password?, ... } }
        ├── verifyHandshake()  ← auth-context.ts + handshake-auth-helpers.ts
        ├── if ok → store ConnectAuthState on the ws client object
        └── if fail → send error, enforce unauthorized-flood-guard
```

**Pass 2 — Post-connect dispatch:**

```
if connected:
  └── parse JSON message
        ├── validate method name
        ├── authorizeGatewayMethod()   ← role + scope check
        └── invoke handler from gatewayMethods[method]
```

**Pass 3 — Origin check:**
`checkBrowserOrigin` in `connect-policy.ts` is called for browser-origin
connections (detected by `isBrowserOperatorUiClient`). It validates the
`Origin` header against the configured allowed origins list.

**Connection to next session:** `authorizeGatewayMethod` is defined in
`server-methods.ts` and calls into `role-policy.ts` and `method-scopes.ts`.

---

## Session 3 — Authentication

**Files:**

- `src/gateway/auth.ts` (493 lines)
- `src/gateway/server/ws-connection/auth-context.ts` (262 lines)
- `src/gateway/server/ws-connection/handshake-auth-helpers.ts` (199 lines)
- `src/gateway/server/ws-connection/connect-policy.ts` (126 lines)
- `src/gateway/auth-rate-limit.ts` (surveyable)

### 3a. Auth resolution (`auth.ts`)

`ResolvedGatewayAuth` is the runtime auth configuration object — mode, token, password,
whether Tailscale auth is allowed. It is computed once at startup from config + env.

`authorizeHttpGatewayConnect` and `authorizeWsControlUiGatewayConnect` are the
two primary entry points. Both call `resolveGatewayAuthResult` internally.

The `GatewayAuthResult` type is the per-request outcome:

```typescript
type GatewayAuthResult = {
  ok: boolean;
  method?:
    | "none"
    | "token"
    | "password"
    | "tailscale"
    | "device-token"
    | "bootstrap-token"
    | "trusted-proxy";
  user?: string;
  reason?: string;
  rateLimited?: boolean;
  retryAfterMs?: number;
};
```

**Credential comparison:** `safeEqualSecret` from `src/security/secret-equal.ts`
— constant-time comparison to prevent timing attacks on bearer tokens.

**Tailscale auth path:** When `allowTailscale` is true and the request arrives
from a Tailscale node, `readTailscaleWhoisIdentity` resolves the Tailscale
identity. The user field in `GatewayAuthResult` is populated with the Tailscale
identity — this is the only auth path that provides a named user identity.

### 3b. WS handshake auth (`auth-context.ts`, `handshake-auth-helpers.ts`)

`ConnectAuthState` is built during the handshake:

- `authOk` — shared secret (token/password) verified
- `sharedAuthOk` — same but also tracks if auth was provided at all
- `deviceTokenCandidate` — may fall back to using the shared token as a device token

Device pairing is a separate auth path from the shared secret:

- `verifyDeviceToken` — verifies a pre-paired device's token
- `verifyBootstrapToken` — one-time token for new device pairing

**Roles and scopes (from `method-scopes.ts`):**

```
Roles:    "operator" | "node"
Scopes:   operator.admin | operator.read | operator.write
          operator.approvals | operator.pairing
```

`CLI_DEFAULT_OPERATOR_SCOPES` = all five scopes (full access).
`NODE_ROLE_METHODS` = 7 node-specific methods; `node` role can only call these.
`ADMIN_SCOPE` bypasses per-method scope checks — it is effectively superuser.

### 3c. Connect policy (`connect-policy.ts`)

`resolveControlUiAuthPolicy` governs the web UI auth bypass options:

- `allowInsecureAuth` — allows non-device-paired connections (but does not bypass
  device auth requirement)
- `dangerouslyDisableDeviceAuth` — full bypass of device pairing requirement

**Study checkpoint:** After this session you should be able to trace the auth
path for any of the four connection types:

1. CLI client (full scopes, device-paired or loopback-exempt)
2. Web UI / browser (origin check + device auth)
3. Mobile node (node role, limited method set)
4. HTTP REST (`/tools/invoke` — bearer token, no WebSocket)

---

## Session 4 — Method Dispatch and Role Authorization

**Files:**

- `src/gateway/server-methods.ts` (157 lines) — the dispatch table
- `src/gateway/server-methods-list.ts` (149 lines) — the method registry
- `src/gateway/role-policy.ts` (surveyable)
- `src/gateway/method-scopes.ts` (surveyable)
- `src/gateway/server-methods/types.ts` (surveyable)

### 4a. The dispatch table (`server-methods.ts`)

`authorizeGatewayMethod` is the per-call gate:

```
authorizeGatewayMethod(method, client)
  1. if no client.connect → null (unauthenticated path, "health" is the only allowed method)
  2. parseGatewayRole(client.connect.role)
  3. isRoleAuthorizedForMethod(role, method)  ← operator vs node segregation
  4. if role === "node" → allowed (node methods already gated by step 3)
  5. if scopes includes ADMIN_SCOPE → allowed
  6. authorizeOperatorScopesForMethod(method, scopes)  ← fine-grained scope check
```

`coreGatewayHandlers` is the merged handler map: 30+ handler groups spread-merged
into one flat object, keyed by method name string.

`CONTROL_PLANE_WRITE_METHODS` = `config.apply`, `config.patch`, `update.run` —
these go through an extra `consumeControlPlaneWriteBudget` rate limit.

### 4b. Handler structure

Every handler group (`agentHandlers`, `sessionsHandlers`, etc.) exports an object:

```typescript
type GatewayRequestHandlers = {
  [method: string]: (params: unknown, opts: GatewayRequestOptions) => Promise<unknown> | unknown;
};
```

`GatewayRequestOptions` carries the authenticated `client` (including the role,
scopes, and connection metadata from the handshake), plus the `context` (runtime
state: config, channel manager, sessions, exec approval manager, etc.).

**Implication for security review:** Adding a new method means adding an entry to
the dispatch table and correctly setting its scope requirements in
`METHOD_SCOPE_GROUPS`. A method not listed in any scope group requires
`ADMIN_SCOPE` to call — this is the safe default. A method incorrectly placed in
`READ_SCOPE` when it should be `WRITE_SCOPE` is a privilege escalation.

---

## Session 5 — Agent Loop and Chat

**Files:**

- `src/gateway/server-methods/agent.ts` (907 lines)
- `src/gateway/server-chat.ts` (883 lines)
- `src/gateway/server-methods/chat.ts` (1875 lines)

This session covers the most complex part of the gateway: how an inbound message
becomes an LLM call, tool invocations, and a streamed reply.

### 5a. The `agent` RPC handler (`server-methods/agent.ts`)

Entry point for running the agent. The handler:

1. Validates params (`validateAgentParams`)
2. Resolves session — `loadSessionEntry` / `mergeSessionEntry` / `updateSessionStore`
3. Resolves outbound delivery plan — `resolveAgentDeliveryPlan` (which channel to
   reply on)
4. Calls `agentCommandFromIngress(params, context)` — this is the bridge to the
   embedded Pi agent runtime
5. Returns `{ runId, acceptedAt }` immediately; the agent runs asynchronously

**Session key semantics:** A session key is a routing identifier, not a security
boundary. `classifySessionKeyShape` determines if it is a main session, a subagent
session, or a custom key. All three share the same auth scope within one gateway.

**`sessions_spawn` path:** `reactivateCompletedSubagentSession` and the spawn
flow in `normalizeSpawnedRunMetadata` — this is where sub-agent context is set up.
The spawned session inherits the parent's tool policy unless overridden. This is
the scope-leakage risk identified in the STRIDE model.

### 5b. Agent event propagation (`server-chat.ts`)

`createAgentEventHandler` subscribes to the embedded agent runtime events and
translates them into WebSocket broadcasts:

```
embedded agent emits event (text chunk / tool call / tool result / lifecycle)
  └── createAgentEventHandler
        ├── normalizeHeartbeatChatFinalText  ← suppresses heartbeat noise
        ├── stripInlineDirectiveTagsForDisplay
        ├── persistGatewaySessionLifecycleEvent  ← writes to JSONL
        └── broadcast to subscribed WS clients
```

**Heartbeat handling:** Heartbeat runs are special agent runs that execute
autonomously on a schedule. `shouldHideHeartbeatChatOutput` suppresses their
output from the webchat surface — but the heartbeat run still has full tool access.

### 5c. Chat history and message injection (`server-methods/chat.ts`, 1875 lines)

The largest handler file. Covers:

- `chat.history` — reads JSONL transcripts and returns formatted history
- `chat.send` — injects a message directly into the session without running the agent
- `chat.inject` — lower-level injection, used by channel adapters
- Transcript event handling for real-time subscriptions

**Read focus:** The `chat.directive-tags.test.ts` (1832 lines) alongside `chat.ts`
is the best documentation of how directive tags (`<openclaw:inject>`,
`<openclaw:sys>`) are parsed and what they can do. These are an internal
instruction channel layered on top of user-visible message content.

---

## Session 6 — Tool Dispatch

**Files:**

- `src/gateway/tools-invoke-http.ts` (surveyable — already partially read)
- `src/gateway/server-methods/nodes.ts` (1194 lines)
- `src/gateway/exec-approval-manager.ts` (210 lines)
- `src/gateway/node-invoke-system-run-approval.ts` (303 lines)
- `src/security/dangerous-tools.ts` (40 lines — already read)
- `src/agents/tool-policy-pipeline.ts` (surveyable)

### 6a. HTTP tool dispatch (already read in Phase 0)

Key points to re-examine with fresh eyes after Sessions 3–5:

- The session key from `body.sessionKey` selects the agent and its policy context.
  An attacker who knows a valid session key for a broader policy context can use
  it to invoke tools with wider permissions — this is the policy-layer bypass
  risk in the STRIDE model.
- `x-openclaw-message-channel` header influences `resolveGroupToolPolicy` —
  an untrusted caller that can set this header could shift into a different policy
  group.

### 6b. Node tool dispatch (`server-methods/nodes.ts`, 1194 lines)

Nodes are remote execution extensions (mobile device, second machine) that the
gateway orchestrates. The `node.invoke` method:

```
node.invoke
  └── resolveEffectiveToolPolicy()
        ├── tool policy pipeline (same as HTTP)
        ├── + node command allowlist check (resolveNodeCommandAllowlist)
        └── dispatch to node via pending work queue
              └── node polls node.pending.pull, executes, returns via node.invoke.result
```

The node command allowlist (`node-command-policy.ts`) is separate from the exec
approval system — it governs what commands the gateway will request a remote node
to execute. The exec approval system governs what the local gateway host will run.

### 6c. Exec approval system (`exec-approval-manager.ts`)

`ExecApprovalManager` is the runtime allowlist for shell commands:

```
exec tool call
  └── getApproval(command, cwd, env, fileSnapshot)
        ├── check stored approvals (loadExecApprovals)
        ├── if match → approved
        ├── if not → emit approval request event
        └── wait for UI decision (exec.approval.waitDecision)
```

Approval entries bind: command string, cwd, env snapshot, optionally a file
content hash. This is best-effort integrity — it prevents a command from being
approved once and then having its script file swapped out silently.

**Limits of the approval system:** The approval check binds the exact command
string. Interpreter-level substitution (for example environment variable expansion
inside the shell, `eval`, dynamic `require`) is not modeled. The approval is a
point-in-time decision on the surface form of the command.

### 6d. Run approval for nodes (`node-invoke-system-run-approval.ts`)

Mirrors the exec approval system for node-dispatched runs. Key difference: the
approval check here must happen on the gateway side before the command is sent to
the node — the node itself has no UI to present an approval dialog.

---

## Session 7 — Sessions and Channels

**Files:**

- `src/gateway/server-methods/sessions.ts` (1210 lines)
- `src/gateway/session-transcript-files.fs.ts` (already read)
- `src/gateway/session-utils.ts` (surveyable)
- `src/gateway/server-channels.ts` (593 lines)
- `src/channels/allow-from.ts`, `src/channels/command-gating.ts` (surveyable)

### 7a. Sessions RPC (`server-methods/sessions.ts`, 1210 lines)

Covers all session management operations:

- `sessions.list` — enumerates session store entries
- `sessions.preview` — reads the last N messages from a JSONL transcript
- `sessions.send` — injects content into a session (distinct from `chat.send`)
- `sessions.reset` — archives and resets a session
- `sessions_spawn` — creates a child agent session (high-risk, WS-only)

**`sessions_spawn` in detail:**
The spawn params include `agentId`, `message`, `tools` (override), and `sandbox`
(default `"inherit"`). The child session's tool policy is resolved from:

1. The spawning session's context
2. Any explicit `tools` override in the spawn params
3. The `subagentToolPolicy` from config

There is no capability manifest or explicit delegation list — the child inherits
from parent context unless a blanket override is provided. This is the gap the
Phase 3 scoped delegation proposal addresses.

### 7b. Channel manager (`server-channels.ts`, 593 lines)

`createChannelManager` returns a stateful object that:

- Holds all active channel plugin instances
- Routes inbound messages to the agent loop
- Tracks channel health (used by `channel-health-monitor.ts`)

The `start()` method loads each configured channel plugin, initializes it, and
registers the `onMessage` callback. The callback path is:

```
channel adapter onMessage callback
  └── channelManager.handleInbound(channelId, message)
        └── resolveInboundRoute(channelId, message)
              ├── allow-from check
              ├── command-gating check
              └── agentCommand() or chat.send()
```

### 7c. Allowlist and command gating

`allow-from.ts` — per-channel sender allowlist. Patterns support exact match,
glob, and regex (with `safe-regex.ts` guarding against ReDoS). Owner-only senders
bypass all tool restrictions; non-owner senders are subject to the configured
tool profile.

`command-gating.ts` — governs which senders can invoke slash commands. Slash
commands map to internal gateway methods; some are owner-only.

---

## Dependency Map (Quick Reference)

```
server.impl.ts
  ├── server/http-listen.ts           HTTP bind
  ├── server-ws-runtime.ts
  │     └── server/ws-connection.ts
  │           └── server/ws-connection/
  │                 ├── message-handler.ts    WS message lifecycle
  │                 ├── auth-context.ts       handshake auth state
  │                 ├── handshake-auth-helpers.ts
  │                 ├── connect-policy.ts     control UI policy
  │                 └── unauthorized-flood-guard.ts
  ├── server-methods.ts               dispatch table + authorizeGatewayMethod
  │     └── server-methods/
  │           ├── agent.ts            → agentCommandFromIngress()
  │           ├── chat.ts             history / inject / send
  │           ├── sessions.ts         session CRUD + sessions_spawn
  │           └── nodes.ts            node tool dispatch
  ├── tools-invoke-http.ts            HTTP POST /tools/invoke
  ├── server-channels.ts              channel manager
  ├── auth.ts                         ResolvedGatewayAuth + per-request check
  ├── auth-rate-limit.ts              rate limiter
  ├── exec-approval-manager.ts        shell command approval
  ├── node-invoke-system-run-approval.ts  node command approval
  └── security/
        ├── dangerous-tools.ts        static deny lists
        └── audit.ts                  security audit engine
```

---

## Study Checkpoints

After completing all seven sessions you should be able to answer:

1. **Startup:** What are the five things that happen before the first WebSocket connection
   is accepted? What would happen if `activateSecretsRuntimeSnapshot` failed?

2. **Auth:** What is the difference between `authOk` and `sharedAuthOk` in
   `ConnectAuthState`? When does a device token fall back to the shared secret?

3. **Dispatch:** Why does `coreGatewayHandlers` use spread-merge instead of a
   class hierarchy? What is the consequence for a method that appears in two
   handler groups?

4. **Agent loop:** What is the relationship between `runId` and `sessionKey`?
   Can two concurrent agent runs share the same session key?

5. **Tool policy:** Trace a call to `sessions_spawn` over WebSocket with a
   `node` role client. Which checks does it pass or fail?

6. **Sessions:** What happens to a JSONL transcript when `sessions.reset` is
   called? At what point is data actually deleted vs. archived?

7. **Channels:** A Telegram message arrives from a sender not in the allowlist.
   Name every point in the code where it could be rejected before reaching the
   agent.

---

## Suggested Next Deep-Dives

After this walkthrough, the most productive deeper reads (in order of security relevance) are:

| Topic                   | Files                                                                 | Why                                                                    |
| ----------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Tool policy pipeline    | `src/agents/tool-policy-pipeline.ts`, `src/agents/pi-tools.policy.ts` | Understand how the layered allow/deny policy actually evaluates        |
| Embedded agent runtime  | `src/agents/pi-embedded-runner/`                                      | Where SOUL.md is loaded and the LLM loop runs                          |
| Exec safe-bin policy    | `src/infra/exec-safe-bin-runtime-policy.ts`                           | Governs which binaries are considered "safe" for exec without approval |
| Skill install gating    | `src/agents/skills/refresh.ts`, `src/agents/skills/install.ts`        | Supply-chain control point                                             |
| SSRF and network policy | `src/infra/net/ssrf.ts`                                               | Controls agent network egress                                          |
| Security audit engine   | `src/security/audit.ts` (full read)                                   | Complete picture of existing audit checks                              |

---

## See Also

- [Attack Surface Map](attack-surface-map.md)
- [STRIDE Threat Model](stride-threat-model.md)
- [Security (hardening guide)](index.md)
- [Agent Loop](/concepts/agent-loop)
- [Sandboxing](/gateway/sandboxing)
