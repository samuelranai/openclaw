---
title: "Session 5 — Agent Loop and Chat"
summary: "Step-by-step walkthrough of the gateway's agent dispatch, idempotency/dedupe, async fire-and-forget, agent event propagation, and the chat.send/chat.history/chat.inject/chat.abort handlers — with VS Code debugger breakpoints"
read_when:
  - Following the Gateway Source Walkthrough, Session 5
  - Learning how an inbound RPC message triggers the LLM agent and streams back output
  - Investigating idempotency, session key semantics, or chat event broadcasting
---

# Session 5 — Agent Loop and Chat

This session covers the most complex part of the gateway: how a `chat.send` or
`agent` RPC message travels from a WS client through the gateway's session layer
into the embedded agent runtime, and how LLM output is streamed back as `chat`
and `agent` events.

**Prerequisite:** Complete Sessions 1–4. You should know `handleGatewayRequest`
(Session 4) and how a `GatewayRequestContext` is built before any handler runs.

---

## 1. What this session covers

| File                                              | Lines   | Role                                                                         |
| ------------------------------------------------- | ------- | ---------------------------------------------------------------------------- |
| `src/gateway/server-methods/agent.ts`             | 907     | `agent`, `agent.identity.get`, `agent.wait` handlers                         |
| `src/gateway/server-chat.ts`                      | 883     | `createAgentEventHandler` — translates agent runtime events to WS broadcasts |
| `src/gateway/server-methods/chat.ts`              | 1875    | `chat.history`, `chat.abort`, `chat.send`, `chat.inject` handlers            |
| `src/gateway/server-methods/agent-wait-dedupe.ts` | ~230    | Idempotency store + `agent.wait` subscribe/notify mechanism                  |
| `src/gateway/server-methods/agent-job.ts`         | ~160    | `waitForAgentJob` — lifecycle-event-based completion polling                 |
| `src/agents/agent-command.ts`                     | (large) | `agentCommandFromIngress` — bridge to embedded agent runtime                 |

---

## 2. Two entry points: `agent` vs `chat.send`

These two RPC methods both run the LLM, but they are architecturally different:

|                  | `agent`                                                                                   | `chat.send`                                                     |
| ---------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| Who calls it     | `openclaw agent --message "..."`, subagent spawns, channel adapters                       | Web UI, CLI chat interactive mode                               |
| Response style   | Fire-and-forget: immediate `{ status: "accepted" }`, result arrives as second `res` frame | Immediate `{ status: "started" }`, then streaming `chat` events |
| Abort support    | `agent.wait` to poll completion                                                           | `chat.abort`                                                    |
| Session key req. | Optional (defaults to resolved main session)                                              | Required                                                        |
| Deliver param    | Supports `deliver: true` to route reply via messaging channel                             | Supports `deliver: true` for external delivery                  |

Both ultimately call `agentCommandFromIngress` from `src/agents/agent-command.ts` —
that function is the boundary between the gateway layer and the embedded Pi agent runtime.

---

## 3. The `agent` handler (`server-methods/agent.ts`)

**File:** `src/gateway/server-methods/agent.ts`

### 3a. Validation and permission checks (lines 233–294)

```
agent handler entry
  ├── validateAgentParams(p)            ← AJV validation
  ├── resolveSenderIsOwnerFromClient()  ← has ADMIN_SCOPE?
  ├── resolveAllowModelOverrideFromClient()  ← ADMIN_SCOPE or internal.allowModelOverride
  └── if provider/model override requested but not allowed → 403
```

`resolveSenderIsOwnerFromClient` checks `client.connect.scopes.includes(ADMIN_SCOPE)`.
Only operators with `operator.admin` scope can override provider/model.

### 3b. Idempotency check (lines 309–315)

```typescript
const idem = request.idempotencyKey; // caller-supplied, typically a UUID
const cached = context.dedupe.get(`agent:${idem}`);
if (cached) {
  respond(cached.ok, cached.payload, cached.error, { cached: true });
  return;
}
```

Every `agent` call carries a caller-supplied `idempotencyKey`. On retry the
handler short-circuits and returns the stored result. The dedupe store lives on
`GatewayRequestContext.dedupe` — it is per-connection, not global. Two different
WS connections do not share a dedupe store.

### 3c. Session resolution and update (lines 461–560)

```
if requestedSessionKey:
  loadSessionEntry(requestedSessionKey)   ← reads sessions.json store
  mergeSessionEntry(existing, patch)      ← idempotent merge
  updateSessionStore(storePath, ...)      ← atomic write-back
  if main session → addChatRun(idem, {...})   ← registers for chat.send streaming
  registerAgentRunContext(idem, {...})    ← runId → sessionKey lookup
```

**Session key semantics:** A session key is a routing identifier, not a security
boundary. `classifySessionKeyShape` returns `"main"`, `"subagent"`, `"custom"`,
or `"malformed_agent"`. All shapes share the same auth scope within one gateway.

If the session key is a **main session** (or `"global"`), `addChatRun` registers
the run so the `chat` event broadcaster (`server-chat.ts`) can link it to the
session. This is the bridge between the `agent` fire-and-forget path and the
streaming `chat` events the web UI consumes.

### 3d. Delivery plan (lines 602–662)

```
resolveAgentDeliveryPlan({
  sessionEntry, requestedChannel, explicitTo, wantsDelivery, ...
})
```

Determines _which channel_ the agent reply should be sent on. If `deliver: true`
and the resolved channel is still `INTERNAL_MESSAGE_CHANNEL`, the handler fails
with an error — `deliver: true` requires a known external channel.

### 3e. Fire-and-forget dispatch (lines 682–763)

```typescript
// 1. Store in-flight ack so retries do not spawn a second run
setGatewayDedupeEntry({ dedupe, key: `agent:${idem}`, entry: { ok: true, payload: accepted } });

// 2. Immediate ack to caller
respond(true, { runId, status: "accepted", acceptedAt }, undefined, { runId });

// 3. Async: calls the embedded agent runtime
dispatchAgentRunFromGateway({ ingressOpts: {...}, runId, idempotencyKey, respond, context });
```

`dispatchAgentRunFromGateway` (lines 180–230) wraps `agentCommandFromIngress` in
a `.then()/.catch()` chain. On completion it:

1. Writes the terminal result into the dedupe store
2. Sends a **second `res` frame** to the calling WS connection

The caller sees two `res` frames for the same request id: first `"accepted"`, then
the final `{ status: "ok", result }`. TypeScript clients with `expectFinal: true`
wait for the second frame; Swift/mobile clients use the first.

### 3f. `agent.wait` handler (lines 819–906)

`agent.wait` lets a caller block (long-poll style) until an agent run completes:

```
agent.wait(runId, timeoutMs)
  ├── readTerminalSnapshotFromGatewayDedupe()  ← check if already done
  ├── if already done → respond immediately
  └── else: race two promises
        ├── waitForAgentJob (lifecycle events via AGENT_WAITERS_BY_RUN_ID)
        └── waitForTerminalGatewayDedupe (dedupe store polling)
      → first to resolve wins, other is aborted
      → if both timeout → { status: "timeout" }
```

The two-race design handles the case where the caller is on a different WS
connection than the one that dispatched the run (for example, a reconnect).

---

## 4. Agent event propagation (`server-chat.ts`)

**File:** `src/gateway/server-chat.ts`

`createAgentEventHandler` (line 455) is the subscriber registered to the
embedded agent runtime's event bus. Every LLM chunk, tool call, and lifecycle
event flows through it.

### 4a. Event routing

```
embedded agent emits AgentEventPayload { runId, stream, seq, data, ... }
  └── createAgentEventHandler(evt)
        ├── resolve sessionKey and clientRunId via chatRunState.registry
        ├── if stream === "assistant" && isControlUiVisible
        │     └── emitChatDelta(...)   ← throttled (150 ms), strips directive tags
        ├── if stream === "tool"
        │     ├── broadcast to toolEventRecipients (run-scoped WS cap)
        │     └── broadcast "session.tool" to sessionEventSubscribers (session-scoped)
        ├── broadcast "agent" event to all connected clients
        └── if lifecyclePhase === "end" | "error"
              └── emitChatFinal(...)  ← flushes throttled delta, cleans up buffers
```

### 4b. Delta throttle (line 566)

```typescript
const now = Date.now();
const last = chatRunState.deltaSentAt.get(clientRunId) ?? 0;
if (now - last < 150) {
  return;
}
```

Streaming deltas are throttled at 150 ms to prevent flooding slow clients.
`emitChatFinal` always flushes any un-sent delta **before** emitting the final
event, so clients always receive the complete text.

### 4c. Heartbeat suppression (lines 44–58)

Heartbeat runs execute autonomously on a schedule with full tool access.
`shouldHideHeartbeatChatOutput` suppresses their text from the webchat surface
unless `cfg.agents.defaults.heartbeat.showOk = true`.

**Security note:** Even when suppressed, heartbeat output is never blocked at
the agent runtime level. The suppression is display-layer only.

### 4d. Directive tag stripping (line 543)

```typescript
const cleanedText = stripInlineDirectiveTagsForDisplay(text).text;
```

Directive tags (`<openclaw:inject>`, `<openclaw:sys>`) are stripped from all
text before broadcasting to clients. They are an internal instruction channel
and must not reach the UI as raw markup.

### 4e. Silent reply suppression (lines 556–559)

If the accumulated buffer matches `SILENT_REPLY_TOKEN`, the event is silently
dropped. Silent replies are used by heartbeat and system-level responses that
should produce no visible output.

### 4f. Sequence gap detection (lines 746–758)

```typescript
if (last > 0 && evt.seq !== last + 1) {
  broadcast("agent", { stream: "error", reason: "seq gap", ... });
}
```

The agent event handler tracks sequence numbers per run. A gap emits an `agent`
error event to clients. This does not abort the run — it is a diagnostic signal.

---

## 5. Chat handlers (`server-methods/chat.ts`)

**File:** `src/gateway/server-methods/chat.ts`

### 5a. `chat.history` (line 1154)

```
chat.history({ sessionKey, limit })
  ├── loadSessionEntry(sessionKey)
  ├── readSessionMessages(sessionId, storePath, sessionFile)   ← reads JSONL
  ├── augmentChatHistoryWithCliSessionImports(...)              ← merges imported CLI sessions
  ├── stripEnvelopeFromMessages(...)    ← removes channel metadata
  ├── sanitizeChatHistoryMessages(...)  ← cleans content
  ├── replaceOversizedChatHistoryMessages(...)   ← 128 KB single-message cap
  ├── capArrayByJsonBytes(...)          ← total bytes cap (getMaxChatHistoryMessagesBytes())
  └── respond(true, { sessionKey, sessionId, messages, thinkingLevel, fastMode, verboseLevel })
```

**Limits:**

- Default: `limit = 200`, hard max = 1000 messages
- Single message hard cap: 128 KB (CHAT_HISTORY_MAX_SINGLE_MESSAGE_BYTES)
- Total response bytes: `getMaxChatHistoryMessagesBytes()` from server-constants.ts
- Text field hard cap: 12,000 chars (CHAT_HISTORY_TEXT_MAX_CHARS)
- Oversized messages replaced with `"[chat.history omitted: message too large]"` placeholder

### 5b. `chat.abort` (line 1223)

```
chat.abort({ sessionKey, runId? })
  ├── resolveChatAbortRequester(client)   ← is admin? has connId/deviceId?
  ├── if no runId → abort all active runs for sessionKey
  └── if runId → abort specific run
        └── abortChatRunById(runId, ops, requester)
```

`AbortController` is stored in `context.chatAbortControllers` keyed by `clientRunId`.
Abort is cooperative — the agent runtime's abort signal is checked at each LLM turn
boundary. There is no forced kill.

### 5c. `chat.send` (line 1304) — the interactive chat path

```
chat.send({ sessionKey, message, idempotencyKey, ... })
  ├── validateChatSendParams
  ├── sanitizeChatSendMessageInput     ← NFC normalize, strip NUL, strip disallowed control chars
  ├── if systemInputProvenance/systemProvenanceReceipt → require ADMIN_SCOPE
  ├── if stopCommand ("/stop") → chat.abort path, return
  ├── check dedupe cache (context.dedupe.get("chat:{clientRunId}"))
  ├── check chatAbortControllers (in-flight guard)
  ├── register AbortController in context.chatAbortControllers
  ├── respond(true, { runId, status: "started" }, ...)   ← immediate ack
  ├── resolveAgentSessionFilePath / emitSessionTranscriptUpdate   ← writes user turn to JSONL
  ├── dispatchInboundMessage(ctx, ...)  ← drives agent via auto-reply dispatch
  │     └── ... → agentCommandFromIngress eventually (same bridge as `agent` handler)
  └── on completion:
        setGatewayDedupeEntry(...)   ← persist result
        emitSideResults(...)          ← "btw" payloads for parallel side-channel results
```

**Key difference from `agent`:** `chat.send` uses `dispatchInboundMessage` (the
channel dispatch path with MsgContext), while `agent` uses `agentCommandFromIngress`
directly. Both reach the same embedded agent runtime but through different wrappers:
`chat.send` is the **interactive UI** path (webchat, CLI interactive), `agent` is
the **programmatic API** path (channel adapters, subagent spawns).

**Input sanitization:** `sanitizeChatSendMessageInput` (line 287):

- NFC normalize (Unicode canonical form)
- Reject NUL bytes (`\u0000`)
- Strip disallowed control characters (keep tab/LF/CR and printable range)

**System provenance injection:** `p.systemInputProvenance` and
`p.systemProvenanceReceipt` require `ADMIN_SCOPE`. These fields embed system-level
attribution metadata into the user turn for audit purposes.

### 5d. `chat.inject` (line 1812)

Lower-level injection: writes a message directly into a session's JSONL transcript
without running the agent. Used by channel adapters to inject received messages.
No LLM call is triggered.

```
chat.inject({ sessionKey, message, role, ... })
  ├── validateChatInjectParams
  ├── loadSessionEntry(sessionKey)
  └── appendInjectedAssistantMessageToTranscript(...)
        └── SessionManager.appendMessage(...)   ← MUST use appendMessage, not raw JSONL write
```

**Critical:** `appendInjectedAssistantMessageToTranscript` uses
`SessionManager.appendMessage` (from Pi). Direct raw JSONL writes break the
`parentId` chain in Pi session transcripts — this is documented in
`src/gateway/server-methods/CLAUDE.md`.

---

## 6. Full flow: `chat.send` → streaming `chat` events

```
WS client sends: { method: "chat.send", id: "r1", params: { sessionKey: "main:...", message: "hello", idempotencyKey: "uuid1" } }
  │
  ▼
handleGatewayRequest (server-methods.ts)
  └── authorizeGatewayMethod("chat.send", client)   ← scope: operator.write
  └── chatHandlers["chat.send"](...)
        ├── sanitize / validate
        ├── register AbortController in chatAbortControllers
        ├── respond(true, { runId: "uuid1", status: "started" })
        │     └── WS client receives: { id: "r1", ok: true, payload: { runId, status: "started" } }
        ├── emitSessionTranscriptUpdate(...)    ← user turn written to JSONL
        └── dispatchInboundMessage(ctx)         ← calls embedded agent
              └── async: agent runs, emits events to event bus
                    │
                    ▼
              createAgentEventHandler (server-chat.ts) handles each event:
                ├── stream="assistant": emitChatDelta(...)
                │     └── broadcast("chat", { state: "delta", message: {...} })
                │           └── WS clients receive streaming chunks
                ├── stream="tool": broadcast to toolEventRecipients
                └── lifecyclePhase="end": emitChatFinal(...)
                      ├── flushBufferedChatDeltaIfNeeded(...)
                      └── broadcast("chat", { state: "final", message: {...} })
                            └── WS clients receive: { state: "final" }
              └── chat.send completion:
                    setGatewayDedupeEntry("chat:uuid1", finalPayload)
```

---

## 7. Key constants and limits

| Constant                            | Value               | Location                       |
| ----------------------------------- | ------------------- | ------------------------------ |
| Delta throttle interval             | 150 ms              | `server-chat.ts:566`           |
| Tool event recipient TTL            | 10 min + 30 s grace | `server-chat.ts:265–266`       |
| `chat.history` default limit        | 200 messages        | `chat.ts:1182`                 |
| `chat.history` hard max             | 1000 messages       | `chat.ts:1181`                 |
| Single message size cap             | 128 KB              | `chat.ts:99`                   |
| Text field truncation               | 12,000 chars        | `chat.ts:98`                   |
| Attachment max bytes                | 5 MB                | `agent.ts:325`, `chat.ts:1394` |
| `agent.wait` default timeout        | 30,000 ms           | `agent.ts:834`                 |
| TOOL_EVENT_RECIPIENT_TTL_MS         | 600,000 ms (10 min) | `server-chat.ts:265`           |
| TOOL_EVENT_RECIPIENT_FINAL_GRACE_MS | 30,000 ms           | `server-chat.ts:266`           |

---

## 8. Breakpoint summary

| Breakpoint file + line                    | What you will see                                          |
| ----------------------------------------- | ---------------------------------------------------------- |
| `src/gateway/server-methods/agent.ts:309` | Idempotency check — inspect `idem` and `cached`            |
| `src/gateway/server-methods/agent.ts:461` | Session load — inspect `loadSessionEntry` return           |
| `src/gateway/server-methods/agent.ts:683` | Just before ack; inspect `accepted`, `runId`               |
| `src/gateway/server-methods/agent.ts:716` | `dispatchAgentRunFromGateway` call — final ingress opts    |
| `src/gateway/server-methods/agent.ts:187` | `agentCommandFromIngress` invocation inside dispatch       |
| `src/gateway/server-chat.ts:717`          | Handler entry — every agent event passes through here      |
| `src/gateway/server-chat.ts:566`          | Delta throttle gate — inspect `now - last` vs 150          |
| `src/gateway/server-chat.ts:646`          | `emitChatFinal` entry — final flush before terminal event  |
| `src/gateway/server-methods/chat.ts:1362` | `sanitizeChatSendMessageInput` — inspect raw vs sanitized  |
| `src/gateway/server-methods/chat.ts:1447` | Dedupe cache hit check for `chat.send`                     |
| `src/gateway/server-methods/chat.ts:1479` | `chat.send` initial ack — inspect `ackPayload`             |
| `src/gateway/server-methods/chat.ts:1174` | `chat.history` — inspect `rawMessages` before sanitization |
| `src/gateway/server-methods/chat.ts:1430` | `chat.send` stop-command path                              |

---

## 9. Exercises

### Exercise 1 — Trace an `agent` run to completion

Set breakpoints at `agent.ts:683` (ack), `agent.ts:187` (`agentCommandFromIngress`),
and `server-chat.ts:717` (event handler). Send an `agent` RPC from the CLI:

```
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js agent --message "say hello"
```

Verify: you see the ack breakpoint first, then the `agentCommandFromIngress` call,
then multiple event handler entries (one per LLM chunk), and finally `lifecyclePhase === "end"`.

### Exercise 2 — Idempotency replay

Send the same `agent` request twice with the same `idempotencyKey`. Set a breakpoint
at `agent.ts:309`. On the second call, observe `cached` is non-null and the handler
returns immediately without dispatching a second agent run.

### Exercise 3 — Delta throttle in action

Set a breakpoint at `server-chat.ts:566`. Watch `now - last` during a streaming LLM
response. Observe that most deltas are dropped (condition is true) and only ~1 event
per 150 ms passes. Confirm `emitChatFinal` (line 646) always flushes the latest
buffer text before sending `state: "final"`.

### Exercise 4 — `chat.history` size caps

Set a breakpoint at `chat.ts:1190` (`replaceOversizedChatHistoryMessages`). Inspect
a session with long messages. Observe how messages over 128 KB get replaced with the
placeholder string before the response is sent.

### Exercise 5 — Heartbeat suppression

Find a running gateway with a heartbeat-enabled agent. Set a breakpoint at
`server-chat.ts:44` (`shouldHideHeartbeatChatOutput`). Observe that heartbeat runs
set `isHeartbeat = true` in `getAgentRunContext`, and the function returns `true` to
suppress display output. Verify the agent still has full tool access during the run.

---

## 10. Security observations

**Model override gate (`agent.ts:284`):** The `provider`/`model` override fields
require `ADMIN_SCOPE`. A non-admin operator or node client that sends these fields
gets a hard 403. This prevents privilege escalation via model substitution.

**System provenance gate (`chat.ts:1347–1360`):** `systemInputProvenance` and
`systemProvenanceReceipt` require `ADMIN_SCOPE`. These fields inject system-level
attribution into the user turn, influencing audit and compliance trails. Non-admin
callers cannot forge provenance.

**No per-run auth re-check:** `agentCommandFromIngress` receives `senderIsOwner`
and `allowModelOverride` flags that were computed from the WS connection's `client`
object at handler entry. The embedded agent runtime runs with those flags for the
full duration of the run — there is no mid-run re-authorization.

**Dedupe store is per-connection:** The dedupe map (`context.dedupe`) lives on the
WS connection state, not on a shared global store. A reconnecting client cannot
replay the cached result from a previous connection's run.

**Stop command is a message, not a separate RPC:** `chat.send` with text `/stop`
triggers `abortChatRunsForSessionKeyWithPartials` — the same code path as
`chat.abort`. This means the abort path has the same requester authorization check
regardless of which entry point is used.
