---
title: "Session 7 — Sessions and Channels"
summary: "Step-by-step walkthrough of gateway session CRUD (list, create, send, reset, delete, compact), the sessions.reset/delete data lifecycle, channel manager startup and restart policy, allow-from sender gating, and command-gating — with VS Code debugger breakpoints"
read_when:
  - Following the Gateway Source Walkthrough, Session 7
  - Learning how sessions.reset and sessions.delete differ in what data they keep or destroy
  - Investigating channel manager restart policy, allow-from, or command-gating behavior
---

# Session 7 — Sessions and Channels

This session covers two parallel surfaces that rarely interact directly but share
the same underlying `SessionEntry` data model: the session CRUD methods
(`sessions.*` handlers) and the channel adapter manager that routes inbound
messages from Telegram, Discord, Slack, iMessage, etc. into the agent loop.

**Prerequisite:** Complete Sessions 1–6. You should know `handleGatewayRequest`
(Session 4), how `agentCommandFromIngress` is called (Session 5), and the
`performGatewaySessionReset` function referenced in the Session 5 `agent` handler.

---

## 1. What this session covers

| File                                     | Lines   | Role                                                                                                           |
| ---------------------------------------- | ------- | -------------------------------------------------------------------------------------------------------------- |
| `src/gateway/server-methods/sessions.ts` | 1210    | All `sessions.*` RPC handlers                                                                                  |
| `src/gateway/session-reset-service.ts`   | ~430    | `performGatewaySessionReset`, `cleanupSessionBeforeMutation`                                                   |
| `src/gateway/session-utils.ts`           | (large) | Store helpers: `loadSessionEntry`, `readSessionMessages`, `archiveSessionTranscripts`, `listSessionsFromStore` |
| `src/gateway/server-channels.ts`         | 593     | `createChannelManager` — lifecycle, restart policy, health monitor                                             |
| `src/channels/allow-from.ts`             | 53      | `isSenderIdAllowed`, `mergeDmAllowFromSources`, `resolveGroupAllowFromSources`                                 |
| `src/channels/command-gating.ts`         | 67      | `resolveControlCommandGate`, `resolveCommandAuthorizedFromAuthorizers`                                         |

---

## 2. How sessions connect to prior sessions

From Session 4, `coreGatewayHandlers` spreads in `sessionsHandlers` from this
file alongside `agentHandlers` and `chatHandlers`. The handlers share
`GatewayRequestContext` which carries `chatAbortControllers` — session mutations
that interrupt active runs use this map.

From Session 5, `performGatewaySessionReset` is also called directly by the
`agent` handler when a message starts with `/new` or `/reset`.

---

## 3. The `SessionEntry` data model

A `SessionEntry` is the persistent metadata row for one session in `sessions.json`
(the gateway's session store). It is keyed by a **session key** (routing identifier)
and holds:

- `sessionId` — UUID, identifies the JSONL transcript file on disk
- `sessionFile` — absolute path to the JSONL transcript
- `model`, `modelProvider`, `thinkingLevel`, `fastMode`, `verboseLevel` — per-session LLM settings
- `channel`, `groupId`, `groupChannel`, `space` — routing context for delivery
- `spawnedBy`, `parentSessionKey`, `spawnDepth`, `subagentRole` — subagent lineage
- `sendPolicy`, `queueMode`, `queueCap` — inbound message policy
- Token counters: `inputTokens`, `outputTokens`, `totalTokens`, `contextTokens`, `estimatedCostUsd`
- `status`, `startedAt`, `endedAt`, `runtimeMs` — lifecycle fields (written by agent events)

**Key rule:** `sessionKey` is a routing identifier, not a security boundary.
Multiple session keys can point to the same session store row via key migration/pruning.

---

## 4. Sessions handler overview (`server-methods/sessions.ts`)

**File:** `src/gateway/server-methods/sessions.ts`

### 4a. Full handler inventory

| Method                          | Line | Description                                                |
| ------------------------------- | ---- | ---------------------------------------------------------- |
| `sessions.list`                 | 501  | Enumerate all sessions from the combined store             |
| `sessions.subscribe`            | 516  | Register connId for `sessions.changed` broadcast           |
| `sessions.unsubscribe`          | 523  | Deregister connId                                          |
| `sessions.messages.subscribe`   | 530  | Subscribe to per-session transcript events                 |
| `sessions.messages.unsubscribe` | 554  | Unsubscribe                                                |
| `sessions.preview`              | 576  | Read last N messages from up to 64 sessions                |
| `sessions.resolve`              | 638  | Resolve a session key to its canonical form                |
| `sessions.create`               | 652  | Create a new session (+ optional initial message)          |
| `sessions.send`                 | 830  | Send a message to a session (no interrupt of active run)   |
| `sessions.steer`                | 842  | Send a message; interrupt active run first                 |
| `sessions.abort`                | 854  | Abort active run for a session                             |
| `sessions.patch`                | 913  | Update session metadata (model, label, sendPolicy, etc.)   |
| `sessions.reset`                | 978  | Archive transcript + create new session ID (keep metadata) |
| `sessions.delete`               | 1004 | Remove session entry (+ optionally archive transcript)     |
| `sessions.get`                  | 1081 | Read messages from a single session                        |
| `sessions.compact`              | 1103 | Trim JSONL transcript to last N lines                      |

### 4b. Webchat write guard (lines 201–222)

```typescript
function rejectWebchatSessionMutation({ action, client, isWebchatConnect, respond }) {
  if (!client?.connect || !isWebchatConnect(client.connect)) return false;
  if (client.connect.client.id === GATEWAY_CLIENT_IDS.CONTROL_UI) return false;
  respond(
    false,
    undefined,
    errorShape(
      INVALID_REQUEST,
      `webchat clients cannot ${action} sessions; use chat.send for session-scoped updates`,
    ),
  );
  return true;
}
```

Called by `sessions.patch` and `sessions.delete`. Standard webchat clients are
blocked from these mutations — only the control UI client (`CONTROL_UI` id) is
exempted. This prevents webchat clients from destructively mutating sessions out
from under ongoing runs.

### 4c. `sessions.list` (line 501)

```
sessions.list({ opts })
  └── loadCombinedSessionStoreForGateway(cfg)   ← merges all store files
  └── listSessionsFromStore({ cfg, storePath, store, opts })
      └── respond(true, result)
```

`loadCombinedSessionStoreForGateway` merges the primary store with any legacy
or migrated store files into a single in-memory view. The caller receives the
merged result without needing to know which physical file each entry came from.

### 4d. `sessions.preview` (line 576)

```
sessions.preview({ keys: string[], limit: number, maxChars: number })
  ├── keys capped at 64
  ├── limit: 1–∞ (default 12)
  ├── maxChars: 20–∞ (default 240)
  └── for each key:
        loadSessionEntry(key)
        readSessionPreviewItemsFromTranscript(sessionId, storePath, { limit, maxChars })
        → SessionsPreviewEntry { key, sessionId, items }
```

Preview reads only the tail of the JSONL transcript — it does not load full
message content into memory. The `maxChars` cap truncates individual text fields.
Keys that have no session entry return an empty `items: []` without error.

### 4e. `sessions.create` (line 652)

```
sessions.create({ key?, agentId?, label?, model?, parentSessionKey?, task?, message? })
  ├── generate key: provided || buildDashboardSessionKey(agentId)
  ├── updateSessionStore → applySessionsPatchToStore  ← atomic write
  ├── ensureSessionTranscriptFile(sessionId, storePath)  ← create JSONL with header
  ├── if transcript creation fails → delete store entry and return error
  └── if task/message provided → chatHandlers["chat.send"](...)
        └── respond includes { runStarted, runId, messageSeq } if agent was started
```

**Two-phase creation:** The store entry and the JSONL file are created in two
separate operations. If the JSONL creation fails, the store entry is rolled back.
There is a brief window between the two where the store entry exists but has no
transcript — queries during this window return an empty message list gracefully.

### 4f. `sessions.send` vs `sessions.steer` (lines 830–852)

Both call `handleSessionSend` but differ in the `interruptIfActive` flag:

- `sessions.send` — `interruptIfActive: false`: message is queued if a run is
  active; it will not interrupt the current run
- `sessions.steer` — `interruptIfActive: true`: calls `interruptSessionRunIfActive`
  first, which calls `chat.abort` and waits for the embedded Pi run to end
  (up to 15 s) before injecting the new message

`interruptSessionRunIfActive` (lines 303–378) checks two abort targets:

1. `context.chatAbortControllers` — tracked gateway-layer abort controllers
2. `isEmbeddedPiRunActive(sessionId)` — whether the embedded Pi runner has an
   active run for this session

Both must terminate before the steer message is delivered.

---

## 5. Session reset and delete lifecycle (`session-reset-service.ts`)

**File:** `src/gateway/session-reset-service.ts`

### 5a. `sessions.reset` (handler at `sessions.ts:978`, implementation in `session-reset-service.ts`)

`performGatewaySessionReset` is the canonical reset path — it is called by both
`sessions.reset` and the `agent` handler's `/new`/`/reset` command.

```
performGatewaySessionReset({ key, reason: "new" | "reset" })
  ├── loadSessionEntry(key)
  ├── triggerInternalHook("command", reason, sessionKey, { sessionEntry })
  ├── cleanupSessionBeforeMutation(...)          ← abort run + close ACP + close browser tabs
  │     ├── abortEmbeddedPiRun(sessionId)
  │     ├── waitForEmbeddedPiRunEnd(sessionId, 15s)  ← cooperative abort
  │     ├── stopSubagentsForRequester(sessionKey)
  │     ├── clearSessionQueues(storeKeys)
  │     ├── clearBootstrapSnapshot(canonicalKey)
  │     ├── closeTrackedBrowserTabsForSessions(...)
  │     └── closeAcpRuntimeForSession(...)       ← cancel/close ACP session (15s timeout each)
  ├── updateSessionStore(storePath):
  │     ├── new sessionId = randomUUID()
  │     ├── new sessionFile path (same directory, new UUID name)
  │     ├── carry forward: model settings, thinkingLevel, fastMode, verboseLevel,
  │     │   spawnedBy, channel, delivery context, exec settings, label, etc.
  │     ├── zero out: inputTokens, outputTokens, totalTokens, systemSent, abortedLastRun
  │     └── model/contextTokens/systemPromptReport stripped (stripRuntimeModelState)
  ├── archiveSessionTranscriptsForSession(oldSessionId, ..., reason: "reset")
  │     └── renames old JSONL to <name>.bak.<timestamp>.jsonl  ← not deleted
  └── write new empty JSONL header at nextEntry.sessionFile
```

**What is preserved on reset:** model preferences, labels, delivery routing,
spawn lineage, exec settings, queue config, ACP binding.

**What is zeroed on reset:** token counters, `systemSent`, `abortedLastRun`,
model/context state from the previous run.

**What happens to the old transcript:** Archived (renamed to `.bak.*.jsonl`)
but **not deleted**. The gateway only writes the `.bak` suffix rename — no bytes
are destroyed.

### 5b. `sessions.delete` (handler at `sessions.ts:1004`)

```
sessions.delete({ key, deleteTranscript?: boolean })
  ├── rejectWebchatSessionMutation check
  ├── refuse if key === resolveMainSessionKey(cfg)   ← main session cannot be deleted
  ├── cleanupSessionBeforeMutation(...)              ← same as reset: abort + ACP + browser tabs
  ├── updateSessionStore: delete store[primaryKey]
  ├── if deleteTranscript (default true):
  │     archiveSessionTranscriptsForSession(sessionId, reason: "deleted")
  │     ← same .bak rename, NOT an actual file removal
  └── emitSessionUnboundLifecycleEvent(...)
        ├── channelRuntime.discord.threadBindings.unbindBySessionKey(...)
        └── hookRunner.runSubagentEnded(...)   if hooks registered
```

**"Delete" does not delete transcripts from disk.** The default behavior is the
same `.bak` rename used by reset. Passing `deleteTranscript: false` skips even
that rename. In neither case are the JSONL bytes removed. This is intentional —
the bak files serve as audit history and can be read manually.

The **main session** (`resolveMainSessionKey(cfg)`) cannot be deleted — it
is the default routing target and always exists.

### 5c. `sessions.compact` (handler at `sessions.ts:1103`)

```
sessions.compact({ key, maxLines?: number })
  ├── maxLines default: 400
  ├── count lines in JSONL transcript
  ├── if lines.length <= maxLines → respond { compacted: false, kept: N }
  └── else:
        archiveFileOnDisk(filePath, "bak")     ← rename current to .bak
        keptLines = lines.slice(-maxLines)     ← last N lines
        write new JSONL with keptLines         ← overwrite with truncated content
        emitSessionsChanged(context, { reason: "compacted" })
```

Unlike reset and delete, `sessions.compact` **does overwrite** the transcript
with a shorter file. The old file is archived first (`.bak` rename), so the
full history is still on disk. Compact is the only path that changes what an
agent can "see" in its context window without running a session reset.

---

## 6. Channel manager (`server-channels.ts`)

**File:** `src/gateway/server-channels.ts`

`createChannelManager` is instantiated once in `server.impl.ts` and passed into
the `GatewayRequestContext`. It owns the full lifecycle of all channel adapters.

### 6a. Startup sequence (line 497)

```
startChannels()
  └── for each plugin in listChannelPlugins():
        startChannel(plugin.id)
          └── startChannelInternal(channelId, accountId?)
                ├── plugin.config.listAccountIds(cfg)     ← which accounts?
                ├── plugin.config.resolveAccount(cfg, id) ← account credentials
                ├── plugin.config.isEnabled(account, cfg) ← skip if disabled
                ├── plugin.config.isConfigured(account, cfg) ← skip if unconfigured
                └── plugin.gateway.startAccount({         ← hand off to plugin
                      cfg, accountId, account, runtime, abortSignal, log, getStatus, setStatus
                    })
                      └── returns task Promise
```

The `AbortController` is created before the `await` to guard against overlapping
start calls for the same account. The `starting` map holds a gate promise that
concurrent callers can await to avoid duplicate boots.

### 6b. Auto-restart policy (lines 356–411)

```
task Promise rejects or resolves (channel exits):
  ├── if manuallyStopped → stop (no restart)
  ├── attempt = (restartAttempts.get(key) ?? 0) + 1
  ├── if attempt > MAX_RESTART_ATTEMPTS (10) → give up, log error
  └── else:
        delayMs = computeBackoff(CHANNEL_RESTART_POLICY, attempt)
          ← initialMs: 5s, maxMs: 5 min, factor: 2, jitter: 10%
        await sleepWithAbort(delayMs, abort.signal)
        startChannelInternal(channelId, id, { preserveRestartAttempts: true })
```

| Attempt | Min delay   |
| ------- | ----------- |
| 1       | 5 s         |
| 2       | 10 s        |
| 3       | 20 s        |
| 4       | 40 s        |
| 5       | 80 s        |
| 6       | 160 s       |
| 7       | 300 s (max) |
| 8–10    | 300 s (max) |

`manuallyStopped` is a `Set<string>` keyed by `"channelId:accountId"`. Calling
`stopChannel(...)` adds to this set so the restart loop does not kick in.
`startChannel(...)` removes from this set first.

### 6c. Health monitor config (lines 149–206)

`isHealthMonitorEnabled(channelId, accountId)` resolves whether the health
monitor watches this channel account. Resolution order:

1. `cfg.channels.<channelId>.accounts.<accountId>.healthMonitor.enabled` — per-account override
2. `cfg.channels.<channelId>.healthMonitor.enabled` — per-channel override
3. `plugin.config.resolveAccount(cfg, accountId)` — probes config; if resolution
   throws → `false` (fail closed)
4. Default: `true`

### 6d. Runtime snapshot (lines 534–593)

`getRuntimeSnapshot()` returns a `ChannelRuntimeSnapshot` — a point-in-time view
of every channel account's running state. Used by `channels.status` and health
monitor. Fields per account: `accountId`, `running`, `restartPending`, `connected`,
`lastError`, `lastStartAt`, `lastStopAt`, `reconnectAttempts`.

---

## 7. Allow-from gating (`channels/allow-from.ts`)

**File:** `src/channels/allow-from.ts`

`isSenderIdAllowed` is the core check for per-channel sender allowlists:

```typescript
function isSenderIdAllowed(
  allow: { entries: string[]; hasWildcard: boolean; hasEntries: boolean },
  senderId: string | undefined,
  allowWhenEmpty: boolean,
): boolean {
  if (!allow.hasEntries) return allowWhenEmpty; // empty list → configurable default
  if (allow.hasWildcard) return true; // "*" → allow all
  if (!senderId) return false; // no sender id → deny
  return allow.entries.includes(senderId); // exact match only
}
```

**Three cases:**

1. `allowFrom` not configured (`hasEntries = false`) → controlled by `allowWhenEmpty`
   (typically `true` for DMs, `false` for groups depending on `dmPolicy`)
2. `allowFrom: ["*"]` → wildcard, allow all senders
3. `allowFrom: ["123", "456"]` → exact match against senderId string

**No glob or regex at this level.** Glob/regex support for allowlists lives in the
channel-specific allow-from resolver upstream of this call (not in this file).

`mergeDmAllowFromSources` (line 1) combines static config `allowFrom` with store
entries, filtered by `dmPolicy`:

- `dmPolicy === "allowlist"` → store entries ignored, only config entries used
- Otherwise → store + config entries merged

`resolveGroupAllowFromSources` (line 12): for group contexts, prefers
`groupAllowFrom` over `allowFrom` when present. Falls back to `allowFrom` unless
`fallbackToAllowFrom: false`.

---

## 8. Command gating (`channels/command-gating.ts`)

**File:** `src/channels/command-gating.ts`

Command gating governs which senders can invoke slash commands over channel
adapters. It is separate from role-based authorization (Session 4) — this layer
applies before the message even reaches the gateway WS.

```
resolveControlCommandGate({ useAccessGroups, authorizers, allowTextCommands, hasControlCommand })
  ├── resolveCommandAuthorizedFromAuthorizers({ useAccessGroups, authorizers })
  │     ├── if !useAccessGroups:
  │     │     mode="allow" → true
  │     │     mode="deny"  → false
  │     │     mode="configured" → true if any authorizer is configured && allowed
  │     └── if useAccessGroups:
  │           true if any authorizer is configured && allowed
  └── shouldBlock = allowTextCommands && hasControlCommand && !commandAuthorized
      return { commandAuthorized, shouldBlock }
```

`useAccessGroups` is derived from whether the channel has access group rules
configured. When `false`, the `modeWhenAccessGroupsOff` parameter controls the
default (default: `"allow"` — commands pass through).

`resolveDualTextControlCommandGate` is a helper for channels that have two
separate authorization sources (e.g., a primary owner check and a secondary
group-member check) — either can authorize the command.

---

## 9. Full flow: `sessions.reset`

```
WS client: { method: "sessions.reset", params: { key: "main:...", reason: "reset" } }
  │
  ▼
authorizeGatewayMethod("sessions.reset", client)   ← requires operator.write scope
  └── sessionsHandlers["sessions.reset"]
        ├── performGatewaySessionReset({ key, reason, commandSource })
        │     ├── triggerInternalHook("command", "reset", key, ...)
        │     ├── cleanupSessionBeforeMutation(...)
        │     │     ├── abortEmbeddedPiRun(sessionId)   ← signal to Pi runner
        │     │     ├── waitForEmbeddedPiRunEnd(sessionId, 15s)
        │     │     ├── stopSubagentsForRequester(key)
        │     │     ├── clearSessionQueues(storeKeys)
        │     │     ├── clearBootstrapSnapshot(key)
        │     │     └── closeAcpRuntimeForSession(...)
        │     ├── updateSessionStore: new sessionId + sessionFile, zero counters
        │     ├── archiveSessionTranscriptsForSession(oldId, reason:"reset")
        │     │     └── rename old JSONL → <name>.bak.<timestamp>.jsonl
        │     └── write new empty JSONL header
        ├── respond(true, { ok, key, entry })
        └── emitSessionsChanged(context, { sessionKey, reason: "reset" })
              └── broadcastToConnIds("sessions.changed", { ... }, sessionEventSubscribers)
```

---

## 10. Full flow: inbound channel message routing

```
Telegram message arrives at Telegram channel adapter
  │
  ▼
telegram plugin onMessage callback
  └── channelManager context → resolve routing
        ├── allow-from check (isSenderIdAllowed)    ← is sender in allowFrom list?
        ├── command-gating check (resolveControlCommandGate)  ← can sender use slash cmds?
        └── if allowed:
              agentCommandFromIngress(ingressOpts, runtime, deps)
                └── ... Session 5 agent dispatch path
              OR chat.send(...)   ← for message injection without running agent
```

The channel adapter provides the `senderIsOwner` and `allowModelOverride` flags
based on the sender's configured role. These are the same flags that govern model
override permission in the `agent` handler (Session 5).

---

## 11. Key constants

| Constant                            | Value              | Location                       |
| ----------------------------------- | ------------------ | ------------------------------ |
| Channel restart max attempts        | 10                 | `server-channels.ts:24`        |
| Channel restart initial delay       | 5,000 ms           | `server-channels.ts:19`        |
| Channel restart max delay           | 300,000 ms (5 min) | `server-channels.ts:20`        |
| Channel restart backoff factor      | 2                  | `server-channels.ts:21`        |
| Channel restart jitter              | 10%                | `server-channels.ts:22`        |
| Run abort wait on reset/delete      | 15,000 ms          | `session-reset-service.ts:145` |
| ACP cleanup timeout                 | 15,000 ms          | `session-reset-service.ts:37`  |
| `sessions.preview` max keys         | 64                 | `sessions.ts:585`              |
| `sessions.preview` default limit    | 12                 | `sessions.ts:587`              |
| `sessions.preview` default maxChars | 240                | `sessions.ts:591`              |
| `sessions.compact` default maxLines | 400                | `sessions.ts:1116`             |
| `sessions.get` default limit        | 200                | `sessions.ts:1090`             |

---

## 12. Breakpoint summary

| Breakpoint file + line                        | What you will see                                                                |
| --------------------------------------------- | -------------------------------------------------------------------------------- |
| `src/gateway/server-methods/sessions.ts:501`  | `sessions.list` entry — inspect `loadCombinedSessionStoreForGateway` result      |
| `src/gateway/server-methods/sessions.ts:701`  | `sessions.create` store write — inspect `applySessionsPatchToStore` return       |
| `src/gateway/server-methods/sessions.ts:730`  | `ensureSessionTranscriptFile` — observe transcript creation and path             |
| `src/gateway/server-methods/sessions.ts:303`  | `interruptSessionRunIfActive` — inspect `hasTrackedRun` and `hasEmbeddedRun`     |
| `src/gateway/server-methods/sessions.ts:978`  | `sessions.reset` entry — inspect key before cleanup                              |
| `src/gateway/server-methods/sessions.ts:1028` | `sessions.delete` — observe `deleteTranscript` flag and archive call             |
| `src/gateway/server-methods/sessions.ts:1176` | `sessions.compact` trim — inspect old line count vs `maxLines`                   |
| `src/gateway/session-reset-service.ts:284`    | `cleanupSessionBeforeMutation` — step through abort + ACP + browser-tab cleanup  |
| `src/gateway/session-reset-service.ts:299`    | `updateSessionStore` inside reset — observe new sessionId allocation             |
| `src/gateway/session-reset-service.ts:389`    | `archiveSessionTranscriptsForSession` — observe bak rename                       |
| `src/gateway/server-channels.ts:239`          | `startChannelInternal` — inspect plugin, accountId, and whether task is started  |
| `src/gateway/server-channels.ts:370`          | Channel restart loop — inspect `attempt` and `delayMs` from backoff              |
| `src/channels/allow-from.ts:38`               | `isSenderIdAllowed` — inspect `allow.entries`, `hasWildcard`, `senderId`         |
| `src/channels/command-gating.ts:38`           | `resolveCommandAuthorizedFromAuthorizers` — inspect `useAccessGroups` and result |

---

## 13. Exercises

### Exercise 1 — Trace `sessions.reset` cleanup

Set breakpoints at `session-reset-service.ts:284` (`cleanupSessionBeforeMutation`)
and line 389 (`archiveSessionTranscriptsForSession`). Trigger a reset from the
CLI: `openclaw sessions reset`. Step through the cleanup: observe which of the
five cleanup steps fire when the session has no active run, and which fire when
the agent is actively running. Verify the old `.jsonl` becomes a `.bak.*.jsonl`
file on disk.

### Exercise 2 — `sessions.send` vs `sessions.steer` interrupt behavior

Start an agent run on a session. While it is running, send both a `sessions.send`
and a `sessions.steer` (from a second WS connection or a second CLI tab). Set a
breakpoint at `sessions.ts:303` (`interruptSessionRunIfActive`). Observe that
`sessions.send` skips this function entirely, while `sessions.steer` enters it
and waits for the abort to complete before proceeding.

### Exercise 3 — Channel manager restart

Stop a configured channel (e.g., disconnect the Telegram adapter) and set a
breakpoint at `server-channels.ts:370` (restart loop). Observe `attempt` count
and `delayMs` growing exponentially. Verify `manuallyStopped` prevents restart
when `stopChannel` is called explicitly.

### Exercise 4 — Allow-from wildcard vs exact

Set a breakpoint at `allow-from.ts:38`. Configure a channel with `allowFrom: ["*"]`
and send a message from an unknown sender. Observe `hasWildcard: true` and the
function returns `true`. Then change to `allowFrom: ["known-id"]` and send from an
unknown sender. Observe `entries.includes(senderId)` fails and the message is
rejected before reaching the agent.

### Exercise 5 — `sessions.compact` effects

Pick a session with a large JSONL transcript (> 400 lines). Set a breakpoint at
`sessions.ts:1176` (compact trim). Call `sessions.compact` with `maxLines: 10`.
Observe the old file archived as `.bak.*`. Then call `sessions.list` and
`chat.history` — verify the history is now limited to the compacted lines. Notice
that a subsequent agent run will only have access to the compacted context.

---

## 14. Security observations

**`sessions.delete` does not erase data:** The `.bak` rename means deleted
sessions remain readable at the file system level. This is a deliberate audit
trail. If a user expects data erasure, this behavior is not what they expect —
it should be surfaced in any privacy or compliance review.

**Main session deletion guard (`sessions.ts:1019`):**
The main session key (`resolveMainSessionKey(cfg)`) cannot be deleted. This
prevents a client from destroying the default routing target, which would break
all subsequent `agent` calls that default to the main session.

**`sessions.steer` waits up to 15 s for abort (`session-reset-service.ts:145`):**
The `waitForEmbeddedPiRunEnd` call has a hard timeout. If the agent run does not
stop within 15 s, the steer is rejected with `UNAVAILABLE`. This prevents an
indefinitely-blocked steer from holding the client connection open, but it also
means a slow-running agent can deny steer commands.

**Channel restart count is per `channelId:accountId` key and resets on success:**
If a channel consistently crashes, it gives up after 10 attempts and stops
restarting. An operator must explicitly call `startChannel` to resume. This is a
self-protection mechanism — a misconfigured channel cannot spin in an infinite
restart loop consuming resources.

**`isSenderIdAllowed` uses exact string match only:**
There is no glob or regex at this layer. A misconfigured `allowFrom` list with a
partial ID or a glob pattern like `"123*"` will simply never match — the gateway
will reject all senders that don't exactly match a listed string. This is a common
misconfiguration source.
