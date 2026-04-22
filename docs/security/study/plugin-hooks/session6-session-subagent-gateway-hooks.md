---
title: "Session 6 — Session, Subagent, and Gateway Lifecycle Hooks"
summary: "Source walkthrough of the eight remaining hooks: session_start/end (session lifecycle), subagent_spawning/delivery_target/spawned/ended (subagent orchestration), and gateway_start/stop (process lifecycle) — with call site traces in session.ts, subagent-spawn.ts, subagent-announce-delivery.ts, and server.impl.ts"
read_when:
  - Building a plugin that tracks conversation lifecycles or subagent orchestration
  - Implementing custom subagent delivery routing via subagent_delivery_target
  - Understanding when gateway_start and gateway_stop fire relative to plugin initialization
---

# Session 6 — Session, Subagent, and Gateway Lifecycle Hooks

This session covers the eight lifecycle hooks that span session management, subagent
orchestration, and gateway process boundaries. Most are Void (fire-and-forget
observability), but two are Modifying with meaningful return values: `subagent_spawning`
(thread binding confirmation) and `subagent_delivery_target` (delivery route override).

**Prerequisite:** Sessions 1–4. Familiarity with the session key architecture from
[Gateway Session 7](../../gateway/security/study/sessions/session7-sessions-and-channels.md) is helpful.

---

## 1. What this session covers

| File                                         | Role                                               |
| -------------------------------------------- | -------------------------------------------------- |
| `src/auto-reply/reply/session.ts`            | `session_start`, `session_end` call sites          |
| `src/agents/subagent-spawn.ts`               | `subagent_spawning`, `subagent_spawned` call sites |
| `src/agents/subagent-announce-delivery.ts`   | `subagent_delivery_target` call site               |
| `src/agents/subagent-registry-completion.ts` | `subagent_ended` (normal completion)               |
| `src/gateway/session-reset-service.ts`       | `subagent_ended` (force-end)                       |
| `src/gateway/server.impl.ts`                 | `gateway_start` call site                          |
| `src/plugins/hook-runner-global.ts`          | `gateway_stop` call site                           |
| `src/plugins/types.ts`                       | All event, context, and result type definitions    |

---

## 2. Session lifecycle hooks

### 2a. `session_start`

**Hook name:** `session_start`
**Execution model:** Void
**Call site:** [src/auto-reply/reply/session.ts:679](../../../../../src/auto-reply/reply/session.ts)

#### When it fires

Fires when a session becomes active — either when a new session is created or when an
existing session is resumed (e.g. after a gateway restart or reconnect). This hook fires
**after** the session is established, not before.

```typescript
// src/auto-reply/reply/session.ts:679
void hookRunner.runSessionStart(payload.event, payload.context).catch(() => {});
```

The `void ... .catch(() => {})` pattern discards both the promise and any errors. This
is deliberately silent — a failing `session_start` handler must not prevent session
activation.

#### Event shape

```typescript
// src/plugins/types.ts:2189
type PluginHookSessionStartEvent = {
  sessionId: string;
  sessionKey?: string;
  resumedFrom?: string; // prior sessionId if this session was resumed from another
};
```

`resumedFrom` is populated when a session continues from a previous one (e.g., the
user reconnected to an existing agent conversation). It allows a plugin to correlate
resumed sessions with their predecessors for analytics.

#### Context shape

```typescript
// src/plugins/types.ts:2182
type PluginHookSessionContext = {
  agentId?: string;
  sessionId: string;
  sessionKey?: string;
};
```

---

### 2b. `session_end`

**Hook name:** `session_end`
**Execution model:** Void
**Call site:** [src/auto-reply/reply/session.ts:667](../../../../../src/auto-reply/reply/session.ts)

#### When it fires

Fires when a session is deactivated — on `/new`, `/reset`, timeout, or explicit
deletion. Like `session_start`, this is fire-and-forget.

```typescript
// src/auto-reply/reply/session.ts:667
void hookRunner.runSessionEnd(payload.event, payload.context).catch(() => {});
```

#### Event shape

```typescript
// src/plugins/types.ts:2196
type PluginHookSessionEndEvent = {
  sessionId: string;
  sessionKey?: string;
  messageCount: number; // total messages at session end
  durationMs?: number; // session wall-clock duration
};
```

`durationMs` is the total time from session activation to deactivation. Combined with
`messageCount`, this enables session-level analytics (messages per minute, session
duration distributions, etc.).

#### Relationship with `before_reset`

`session_end` and `before_reset` are often confused:

|                    | `before_reset`          | `session_end`             |
| ------------------ | ----------------------- | ------------------------- |
| When               | Before messages cleared | After session deactivated |
| Awaited            | Yes                     | No (fire-and-forget)      |
| Has messages?      | Yes (`messages` field)  | No (only count)           |
| Fires on timeout?  | No                      | Yes                       |
| Fires on `/reset`? | Yes                     | Yes (both fire)           |

For archiving, use `before_reset` (awaited, has messages). For analytics, use
`session_end` (fires on all paths including timeout).

---

## 3. Subagent lifecycle hooks

The subagent system allows an agent to spawn child agents — either one-shot runs (`mode: "run"`)
or persistent sessions (`mode: "session"`). Four hooks bracket this lifecycle.

```
parent agent calls sessions.spawn or subagent SDK method
  │
  └─► subagent_spawning (Modifying)  ── §3a
        thread binding provisioned by channel plugins
        │
        ├─► [child agent session created, runId assigned]
        │
        └─► subagent_spawned (Void)   ── §3b
              notification: spawn confirmed with runId
              │
              ├─► [child agent runs to completion]
              │
              ├─► subagent_delivery_target (Modifying)  ── §3c
              │     where to deliver the completion message
              │
              └─► subagent_ended (Void)   ── §3d
                    completion notification with outcome
```

---

### 3a. `subagent_spawning`

**Hook name:** `subagent_spawning`
**Execution model:** Modifying
**Call site:** [src/agents/subagent-spawn.ts:257](../../../../../src/agents/subagent-spawn.ts)

#### When it fires

Fires in `subagent-spawn.ts` before the child session is created. Runs **sequentially**
(not parallel) because channel plugins need to deterministically provision thread bindings
before the spawn proceeds.

```typescript
// src/agents/subagent-spawn.ts:257
const result = await hookRunner.runSubagentSpawning(
  {
    childSessionKey,
    agentId: params.agentId,
    label: params.label,
    mode: params.mode, // "run" | "session"
    requester: params.requester,
    threadRequested: params.threadRequested,
  },
  {
    runId: params.runId,
    childSessionKey,
    requesterSessionKey: params.requesterSessionKey,
  },
);
```

#### Event shape

```typescript
// src/plugins/types.ts:2227 — PluginHookSubagentSpawningEvent = PluginHookSubagentSpawnBase
type PluginHookSubagentSpawnBase = {
  childSessionKey: string; // the session key assigned to the new child agent
  agentId: string;
  label?: string; // display label for the subagent
  mode: "run" | "session"; // one-shot vs persistent
  requester?: {
    channel?: string;
    accountId?: string;
    to?: string;
    threadId?: string | number;
  };
  threadRequested: boolean; // whether the caller requested a reply thread
};
```

#### Result shape

```typescript
// src/plugins/types.ts:2229
type PluginHookSubagentSpawningResult =
  | { status: "ok"; threadBindingReady?: boolean }
  | { status: "error"; error: string };
```

A channel plugin that provisions a thread binding returns `{ status: "ok", threadBindingReady: true }`.
If the binding could not be provisioned, it returns `{ status: "error", error: "..." }` —
the spawn is then aborted.

#### Merge logic

```typescript
// src/plugins/hooks.ts:213
const mergeSubagentSpawningResult = (acc, next) => {
  if (acc?.status === "error") return acc; // first error wins, stops chain
  if (next.status === "error") return next; // error from any handler propagates
  return {
    status: "ok",
    // threadBindingReady is OR'd across all handlers
    threadBindingReady: Boolean(acc?.threadBindingReady || next.threadBindingReady),
  };
};
```

`threadBindingReady` is true if **any** handler confirms the binding is ready. This
allows multiple channel plugins to each provision their own binding independently.

---

### 3b. `subagent_spawned`

**Hook name:** `subagent_spawned`
**Execution model:** Void
**Call site:** [src/agents/subagent-spawn.ts:793](../../../../../src/agents/subagent-spawn.ts)

#### When it fires

Fires after the child session is successfully created and assigned a `runId`. This is a
confirmation hook — the spawn has committed, the child agent will run.

```typescript
// src/agents/subagent-spawn.ts:793
await hookRunner.runSubagentSpawned(
  {
    ...spawnBase, // same fields as subagent_spawning event
    runId: assignedRunId, // now available
  },
  {
    runId: assignedRunId,
    childSessionKey,
    requesterSessionKey: params.requesterSessionKey,
  },
);
```

#### Event shape

`PluginHookSubagentSpawnBase` + `{ runId: string }` — same as `subagent_spawning` but
with `runId` added. The `runId` is the stable identifier for this specific child agent
invocation.

---

### 3c. `subagent_delivery_target`

**Hook name:** `subagent_delivery_target`
**Execution model:** Modifying
**Call site:** [src/agents/subagent-announce-delivery.ts:246](../../../../../src/agents/subagent-announce-delivery.ts)

#### What it does

After a child agent completes, the gateway needs to decide **where to deliver the
completion message** — which channel, which account, which conversation thread. This hook
lets plugins override the default delivery target.

```typescript
// src/agents/subagent-announce-delivery.ts:246
const result = await hookRunner.runSubagentDeliveryTarget(
  {
    childSessionKey,
    requesterSessionKey,
    requesterOrigin: {
      channel: params.requesterChannel,
      accountId: params.requesterAccountId,
      to: params.requesterTo,
      threadId: params.requesterThreadId,
    },
    childRunId: params.childRunId,
    spawnMode: params.spawnMode,
    expectsCompletionMessage: params.expectsCompletionMessage,
  },
  {
    runId: params.childRunId,
    childSessionKey,
    requesterSessionKey,
  },
);
```

#### Event shape

```typescript
// src/plugins/types.ts:2240
type PluginHookSubagentDeliveryTargetEvent = {
  childSessionKey: string;
  requesterSessionKey: string;
  requesterOrigin?: {
    channel?: string;
    accountId?: string;
    to?: string;
    threadId?: string | number;
  };
  childRunId?: string;
  spawnMode?: "run" | "session";
  expectsCompletionMessage: boolean;
};
```

#### Result shape

```typescript
// src/plugins/types.ts:2254
type PluginHookSubagentDeliveryTargetResult = {
  origin?: {
    channel?: string;
    accountId?: string;
    to?: string;
    threadId?: string | number;
  };
};
```

Return an `origin` to override the default delivery target. The gateway uses the first
non-empty `origin` returned across all plugins.

#### Merge logic

```typescript
// src/plugins/hooks.ts:229
const mergeSubagentDeliveryTargetResult = (acc, next) => {
  if (acc?.origin) return acc; // first plugin to provide an origin wins
  return next;
};
```

**First-set-wins** on `origin`. If Plugin A (priority 100) returns an `origin`, Plugin B
(priority 50) cannot override it.

---

### 3d. `subagent_ended`

**Hook name:** `subagent_ended`
**Execution model:** Void
**Two call sites:**

- [src/agents/subagent-registry-completion.ts:72](../../../../../src/agents/subagent-registry-completion.ts) — normal completion
- [src/gateway/session-reset-service.ts:99](../../../../../src/gateway/session-reset-service.ts) — force-end (session reset/delete)

#### When it fires

Fires when a child agent session ends — either naturally (run completed, session ended)
or forcefully (reset, delete, kill). The two call sites cover both paths.

```typescript
// src/agents/subagent-registry-completion.ts:72
await hookRunner.runSubagentEnded(
  {
    targetSessionKey: entry.sessionKey,
    targetKind: entry.kind, // "subagent" | "acp"
    reason: entry.reason,
    sendFarewell: entry.sendFarewell,
    accountId: entry.accountId,
    runId: entry.runId,
    endedAt: entry.endedAt,
    outcome: entry.outcome, // "ok" | "error" | "timeout" | "killed" | "reset" | "deleted"
    error: entry.error,
  },
  { childSessionKey: entry.sessionKey },
);
```

#### Event shape

```typescript
// src/plugins/types.ts:2269
type PluginHookSubagentEndedEvent = {
  targetSessionKey: string;
  targetKind: "subagent" | "acp";
  reason: string;
  sendFarewell?: boolean;
  accountId?: string;
  runId?: string;
  endedAt?: number; // Unix timestamp ms
  outcome?: "ok" | "error" | "timeout" | "killed" | "reset" | "deleted";
  error?: string; // error message if outcome !== "ok"
};
```

`outcome` is the most actionable field — it tells you why the subagent ended. A plugin
that manages conversation threads should handle `"reset"` and `"deleted"` differently
from `"ok"` and `"error"`.

---

## 4. Gateway lifecycle hooks

### 4a. `gateway_start`

**Hook name:** `gateway_start`
**Execution model:** Void
**Call site:** [src/gateway/server.impl.ts:1352](../../../../../src/gateway/server.impl.ts)

#### When it fires

Fires at the end of gateway startup, after HTTP/WS servers are listening, channels are
started, and all sidecars are running. This is Phase 11 in the startup sequence (see
[Gateway Session 1](../../gateway/security/study/sessions/session1-server-impl.md)).

```typescript
// src/gateway/server.impl.ts:1349
const hookRunner = getGlobalHookRunner();
if (hookRunner?.hasHooks("gateway_start")) {
  void hookRunner.runGatewayStart({ port }, { port }).catch((err) => {
    log.warn(`gateway_start hook failed: ${String(err)}`);
  });
}
```

`gateway_start` is fire-and-forget — startup does not wait for handlers. A plugin
`gateway_start` handler that performs slow I/O (e.g. connecting to an external SIEM)
does not delay the gateway from accepting connections.

#### Event shape

```typescript
// src/plugins/types.ts:2287
type PluginHookGatewayStartEvent = {
  port: number; // the port the gateway is listening on
};
```

#### Context shape

```typescript
// src/plugins/types.ts:2282
type PluginHookGatewayContext = {
  port?: number;
};
```

The port is available in both `event` and `ctx` for convenience.

---

### 4b. `gateway_stop`

**Hook name:** `gateway_stop`
**Execution model:** Void
**Call site:** [src/plugins/hook-runner-global.ts:74](../../../../../src/plugins/hook-runner-global.ts)

#### When it fires

Fires during graceful gateway shutdown, via `runGlobalGatewayStopSafely`. Unlike
`gateway_start`, this call is **awaited** inside the `close()` handler — the gateway
waits for all `gateway_stop` handlers to finish before tearing down HTTP/WS servers and
clearing credentials.

```typescript
// src/plugins/hook-runner-global.ts:74
export async function runGlobalGatewayStopSafely(params): Promise<void> {
  const hookRunner = getGlobalHookRunner();
  if (!hookRunner?.hasHooks("gateway_stop")) {
    return;
  }
  try {
    await hookRunner.runGatewayStop(params.event, params.ctx);
  } catch (err) {
    if (params.onError) {
      params.onError(err);
      return;
    }
    log.warn(`gateway_stop hook failed: ${String(err)}`);
  }
}
```

`gateway_stop` is the correct hook for:

- Flushing in-flight SIEM events before shutdown
- Closing external database connections
- Writing final audit records
- Graceful cleanup of plugin-managed resources

#### Event shape

```typescript
// src/plugins/types.ts:2292
type PluginHookGatewayStopEvent = {
  reason?: string; // e.g. "SIGTERM", "SIGINT", "restart"
};
```

---

## 5. Lifecycle timing summary

```
Gateway process starts
  └─► Plugins loaded, hook runner initialized
  └─► HTTP/WS server starts
  └─► Channels start
  └─► gateway_start fires (fire-and-forget)    ← §4a

  ┌── For each conversation ─────────────────────────────────────┐
  │  User connects, session created                               │
  │    └─► session_start fires                                   │
  │                                                               │
  │    ┌── For each agent run ─────────────────────────────────┐  │
  │    │  (See Sessions 2, 3, 5 for agent lifecycle hooks)    │  │
  │    └───────────────────────────────────────────────────────┘  │
  │                                                               │
  │    ┌── For each subagent spawned ──────────────────────────┐  │
  │    │  subagent_spawning fires (Modifying)                  │  │
  │    │  subagent_spawned fires (Void)                        │  │
  │    │  [child agent runs]                                   │  │
  │    │  subagent_delivery_target fires (Modifying)           │  │
  │    │  subagent_ended fires (Void)                          │  │
  │    └───────────────────────────────────────────────────────┘  │
  │                                                               │
  │  Session ends (/new, /reset, timeout)                         │
  │    └─► session_end fires                                     │
  └──────────────────────────────────────────────────────────────┘

Gateway process stops
  └─► gateway_stop fires (awaited)             ← §4b
  └─► HTTP/WS server closes
  └─► Credentials cleared
```

---

## 6. Exercises

### Exercise 1 — Track session duration via `session_start` / `session_end`

1. Register `session_start` with a handler that records `{ sessionId, startTime: Date.now() }` in a plugin-local map.
2. Register `session_end` with a handler that reads `durationMs` from the event and cross-checks it against `Date.now() - storedStartTime`.
3. Start a session, send a few messages, then issue `/new`.
4. Verify both handlers fire and the durations roughly match.
5. Also trigger a timeout by setting a short session timeout in config — verify `session_end` fires for timeout too.

### Exercise 2 — Observe subagent spawn order

1. Register all four subagent hooks with handlers that log `event.childSessionKey` and
   the hook name.
2. From an agent run, use the subagent SDK method to spawn a child agent.
3. Observe the hook fire sequence: `subagent_spawning` → `subagent_spawned` → (child
   agent runs) → `subagent_delivery_target` → `subagent_ended`.
4. Note the `runId` is absent from `subagent_spawning` and `subagent_ended` in some
   cases — the run ID is only guaranteed in `subagent_spawned`.

### Exercise 3 — Override delivery target via `subagent_delivery_target`

1. Register `subagent_delivery_target` with a handler that returns a specific `origin`:
   ```typescript
   handler: async (event) => {
     if (event.spawnMode === "run") {
       return {
         origin: {
           channel: "telegram",
           accountId: "my-bot-account",
           to: "my-telegram-chat-id",
         },
       };
     }
   };
   ```
2. Spawn a subagent in `"run"` mode from an agent conversation.
3. Verify the completion message is delivered to your Telegram chat instead of the
   default channel.

### Exercise 4 — Flush SIEM events on `gateway_stop`

1. Register `gateway_stop` with a handler that sleeps for 500ms (simulating I/O flush):
   ```typescript
   handler: async (event) => {
     await new Promise((r) => setTimeout(r, 500));
     console.log("SIEM flush complete, gateway shutting down");
   };
   ```
2. Start the gateway and stop it cleanly (SIGTERM or Ctrl+C).
3. Observe "SIEM flush complete" in the logs before the gateway closes the HTTP server.
4. Confirm the `await` in `runGlobalGatewayStopSafely` means the gateway waited for your
   handler before proceeding with teardown.

---

## See Also

- [Session 2 — Agent Lifecycle Hooks](session2-agent-lifecycle-hooks.md) — per-run hooks that fire within each agent run
- [Session 3 — Compaction and Reset Hooks](session3-compaction-reset-hooks.md) — `before_reset` vs `session_end`
- [Session 7 — Security Analysis](session7-security-analysis.md)
- [Gateway Session 1 — server.impl.ts](../../gateway/security/study/sessions/session1-server-impl.md) — Phase 11 where `gateway_start` fires
