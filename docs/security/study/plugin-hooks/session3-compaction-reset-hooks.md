---
title: "Session 3 — Compaction and Reset Hooks"
summary: "Source walkthrough of before_compaction, after_compaction, and before_reset — covering the compaction pipeline in pi-embedded-runner, the sessionFile path as an async read opportunity, and the reset call site in commands-core.ts"
read_when:
  - Building a plugin that archives sessions before compaction or reset
  - Understanding the timeline of the compaction pipeline in relation to the LLM call
  - Investigating what data is available at each compaction hook point
---

# Session 3 — Compaction and Reset Hooks

This session covers three hooks that fire at the boundaries of session memory
management: before and after context-window compaction, and before a session is
cleared by `/new` or `/reset`. These are observability-only hooks (all Void), but
the data they expose is uniquely valuable — they are the only points where
pre-compaction transcript content is available to plugins.

**Prerequisite:** Sessions 1 and 2.

---

## 1. What this session covers

| File                                                | Role                                            |
| --------------------------------------------------- | ----------------------------------------------- |
| `src/agents/pi-embedded-runner/compaction-hooks.ts` | Primary compaction hook integration layer       |
| `src/agents/pi-embedded-runner/compact.ts`          | Overflow/retry compaction path                  |
| `src/auto-reply/reply/commands-core.ts`             | `before_reset` call site                        |
| `src/plugins/types.ts`                              | Event type definitions                          |
| `src/plugins/hooks.ts`                              | Runner function implementations (lines 542–568) |

---

## 2. The compaction pipeline overview

Context-window compaction fires when the agent runner detects that the session message
history is approaching the model's context limit. The pipeline has two paths:

**Standard compaction (auto-compaction):**

```
context window nearing limit
  └─► handleAutoCompactionStart()  (compaction-hooks.ts:~190)
        ├─► before_compaction hook fires   ← Session 3
        ├─► compaction LLM call runs (async, in parallel with hook handlers)
        └─► handleAutoCompactionEnd()
              └─► after_compaction hook fires   ← Session 3
```

**Overflow compaction (compact.ts):**

```
context still too large after standard compaction
  └─► compact() is called directly (compact.ts:~1000)
        ├─► before_compaction hook fires (line 1008)
        ├─► compaction LLM call runs
        └─► after_compaction fires only if willRetry is false (line 1053)
```

---

## 3. `before_compaction`

**Hook name:** `before_compaction`
**Execution model:** Void
**Call sites:**

- [src/agents/pi-embedded-runner/compaction-hooks.ts:201](../../../../../src/agents/pi-embedded-runner/compaction-hooks.ts)
- [src/agents/pi-embedded-runner/compact.ts:1008](../../../../../src/agents/pi-embedded-runner/compact.ts)

### 3a. When it fires

Fires when compaction is about to start — **before** the compaction LLM call, while all
original messages are still on disk. This is the archive window.

### 3b. Primary call site walkthrough (`compaction-hooks.ts:201`)

```typescript
// src/agents/pi-embedded-runner/compaction-hooks.ts:~196
async function handleAutoCompactionStart(params) {
  if (params.hookRunner?.runBeforeCompaction) {
    await params.hookRunner.runBeforeCompaction?.(
      {
        messageCount: params.messageCount,
        compactingCount: params.compactingCount,
        tokenCount: params.tokenCount,
        messages: params.messages,
        sessionFile: params.sessionFile, // path to JSONL on disk
      },
      params.agentCtx,
    );
  }
  // compaction LLM call starts here (can run in parallel with hook handlers
  // since hooks are fire-and-forget via Void model)
}
```

Note that the compaction-hooks layer delegates to the runner via `?.runBeforeCompaction` —
the hook runner itself calls `runVoidHook` which launches all handlers in parallel. The
compaction LLM call runs **after** `await params.hookRunner.runBeforeCompaction(...)` —
the `await` here is the outer await on `runVoidHook`'s `Promise.all`, not on individual
handlers. This means:

- All `before_compaction` handlers complete before the compaction LLM call begins.
- However, the compaction pipeline does not **wait for** anything a plugin does with
  `sessionFile` asynchronously inside the handler — the handler just needs to return
  before the LLM call starts.

### 3c. Secondary call site (`compact.ts:1008`)

```typescript
// src/agents/pi-embedded-runner/compact.ts:1008
if (hookRunner?.hasHooks?.("before_compaction") && hookRunner.runBeforeCompaction) {
  await hookRunner.runBeforeCompaction(
    {
      messageCount: session.messages.length,
      tokenCount: tokenCount,
      messages: session.messages,
      sessionFile: sessionFile,
    },
    agentCtx,
  );
}
```

The overflow path (`compact.ts`) calls the hook directly rather than through the
compaction-hooks layer. The event shape is identical, but `compactingCount` may be
absent here since the overflow path doesn't always pre-compute it.

### 3d. Event shape

```typescript
// src/plugins/types.ts:1949
type PluginHookBeforeCompactionEvent = {
  messageCount: number; // total messages in session before any truncation
  compactingCount?: number; // messages fed to compaction LLM (after history-limit truncation)
  tokenCount?: number; // estimated token count
  messages?: unknown[]; // in-memory session messages (may be absent in overflow path)
  sessionFile?: string; // absolute path to JSONL transcript on disk
};
```

**The `sessionFile` field is the key value here.** All session messages — including
those that will be compacted away — are already written to disk at this point. A plugin
can:

1. Read `sessionFile` asynchronously without blocking the compaction LLM call
2. Archive the full transcript to external storage (S3, database, audit log)
3. Extract tool call patterns, response quality metrics, or conversation analytics

The in-memory `messages` array is also provided for convenience, but it may not match
the on-disk state exactly if there were recent writes that have not been flushed.

### 3e. Context shape

Same `PluginHookAgentContext` as all agent lifecycle hooks:
`{ runId?, agentId?, sessionKey?, sessionId?, workspaceDir?, messageProvider?, trigger?, channelId? }`

---

## 4. `after_compaction`

**Hook name:** `after_compaction`
**Execution model:** Void
**Call sites:**

- [src/agents/pi-embedded-runner/compaction-hooks.ts:285](../../../../../src/agents/pi-embedded-runner/compaction-hooks.ts)
- [src/agents/pi-embedded-runner/compact.ts:1053](../../../../../src/agents/pi-embedded-runner/compact.ts) (only when `willRetry === false`)

### 4a. When it fires

Fires after the compaction LLM call completes and the session history has been replaced
with the compacted summary. The original messages are **still on disk** at `sessionFile`
(they are not deleted during compaction — only the in-memory session is replaced).

### 4b. The `willRetry` guard in `compact.ts`

```typescript
// src/agents/pi-embedded-runner/compact.ts:1050
if (!willRetry) {
  if (hookRunner.runAfterCompaction) {
    await hookRunner.runAfterCompaction(
      {
        messageCount: compactedSession.messages.length,
        tokenCount: newTokenCount,
        compactedCount: session.messages.length - compactedSession.messages.length,
        sessionFile: sessionFile,
      },
      agentCtx,
    );
  }
}
```

When `willRetry === true` (the context is still too large after one compaction pass and
another pass is needed), `after_compaction` is **not** called. This prevents plugins from
receiving misleading "compaction complete" signals mid-retry. The hook only fires on the
final successful compaction.

### 4c. Event shape

```typescript
// src/plugins/types.ts:1969
type PluginHookAfterCompactionEvent = {
  messageCount: number; // messages remaining after compaction
  tokenCount?: number; // token count after compaction
  compactedCount: number; // how many messages were compacted away
  sessionFile?: string; // JSONL path — pre-compaction messages still on disk
};
```

`compactedCount = beforeMessageCount - afterMessageCount`. A large `compactedCount`
relative to `messageCount` indicates an aggressive compaction — the session history was
heavily summarized.

**`sessionFile` after compaction:** The JSONL file still contains all the pre-compaction
messages. The compaction process appends a new summary entry rather than overwriting the
file. This means a plugin can still read the original conversation content in
`after_compaction` — but it must do so via the file (the in-memory messages array is now
the compacted version).

---

## 5. `before_reset`

**Hook name:** `before_reset`
**Execution model:** Void
**Call site:** [src/auto-reply/reply/commands-core.ts:108](../../../../../src/auto-reply/reply/commands-core.ts)

### 5a. When it fires

Fires when the user issues `/new` or `/reset`, **before** the session is cleared. This is
the **last opportunity** for a plugin to read session content before it is discarded from
memory. Unlike compaction hooks, `before_reset` does not run when the gateway clears a
session internally (e.g. on timeout) — only on explicit user commands.

### 5b. Call site walkthrough

```typescript
// src/auto-reply/reply/commands-core.ts:~105
if (hookRunner?.hasHooks("before_reset")) {
  await hookRunner.runBeforeReset(
    {
      sessionFile: resolvedSessionFile, // may be undefined for ephemeral sessions
      messages: currentMessages, // in-memory messages before clear
      reason: "reset", // or "new" depending on command
    },
    agentCtx,
  );
}
// session messages are cleared after this
```

The `await` here means the session is not cleared until all `before_reset` handlers
complete. Unlike most Void hooks, this one matters for ordering — a plugin that archives
session content in `before_reset` has a reliable guarantee that the archive finishes
before the messages disappear.

### 5c. Event shape

```typescript
// src/plugins/types.ts:1963
type PluginHookBeforeResetEvent = {
  sessionFile?: string; // path to JSONL transcript — undefined for ephemeral sessions
  messages?: unknown[]; // in-memory messages about to be cleared
  reason?: string; // "reset" | "new"
};
```

### 5d. What `before_reset` does NOT cover

`before_reset` fires only on explicit `/new` and `/reset` commands. It does **not** fire
when:

- The gateway times out a session
- The session is deleted via `sessions.delete` API
- A subagent session ends naturally

For those cases, use `session_end` (covered in Session 6) or `subagent_ended`.

---

## 6. Timeline comparison: `before_compaction` vs `before_reset`

|                         | `before_compaction`           | `before_reset`             |
| ----------------------- | ----------------------------- | -------------------------- |
| Trigger                 | Context window limit reached  | `/new` or `/reset` command |
| Session after hook      | Continues (compacted summary) | Cleared                    |
| Messages on disk        | Yes, at `sessionFile`         | Yes (if not ephemeral)     |
| Messages in memory      | Yes (pre-compaction array)    | Yes (about to be cleared)  |
| Hook blocks pipeline    | Yes (via Void `await`)        | Yes (via Void `await`)     |
| Fires on timeout/delete | No                            | No                         |

Both hooks are awaited before the destructive operation proceeds, giving plugins a
reliable synchronization point for archiving.

---

## 7. Exercises

### Exercise 1 — Archive a session before compaction

1. Register `before_compaction` with a handler that writes `event.sessionFile` path and
   `event.messageCount` to a local log file.
2. Create a long session that triggers context-window compaction (send many messages to
   exhaust the context limit, or use a small `contextWindow` config value).
3. Observe the log entry written by the hook.
4. Read the JSONL file at `event.sessionFile` from within the hook — verify it contains
   the pre-compaction messages.

### Exercise 2 — Differentiate compaction paths

1. Register both `before_compaction` and `after_compaction` with handlers that log
   `event.compactingCount` (before) and `event.compactedCount` (after).
2. Trigger standard auto-compaction.
3. Compare `compactingCount` (fed to LLM) with `compactedCount` (removed from history) —
   they may differ because the LLM may produce a summary that covers more messages than
   were sent to it.
4. Force an overflow compaction by configuring an extremely small context window. Verify
   that `after_compaction` is NOT called if `willRetry` is true for the first pass.

### Exercise 3 — Observe `before_reset` ordering guarantee

1. Register `before_reset` with a handler that sleeps for 200ms
   (`await new Promise(r => setTimeout(r, 200))`), then logs "archive complete".
2. Issue `/reset` from a connected client.
3. Verify in gateway logs that "archive complete" appears before the "session cleared"
   log line — confirming the hook blocks the reset operation.

---

## See Also

- [Session 2 — Agent Lifecycle Hooks](session2-agent-lifecycle-hooks.md)
- [Session 5 — Tool Execution Hooks](session5-tool-execution-hooks.md) — `before_message_write` for per-message interception
- [Session 6 — Session, Subagent, and Gateway Hooks](session6-session-subagent-gateway-hooks.md) — `session_end` for session lifecycle
