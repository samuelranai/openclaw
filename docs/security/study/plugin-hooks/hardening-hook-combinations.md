---
title: "Security Hardening — Hook Combination Analysis and General Proposal"
summary: "Analysis of why after_tool_call cannot intercept or modify prompts alone, the architectural constraints of each tool-adjacent hook, and the general multi-hook combination pattern that achieves post-tool detection with interception or prompt modification"
read_when:
  - Designing a security plugin that needs to react to tool results
  - Understanding why after_tool_call, before_message_write, tool_result_persist, and message_sending do not individually solve post-tool prompt hardening
  - Choosing which hook combination to use for a given security goal
---

# Security Hardening — Hook Combination Analysis and General Proposal

This document answers a common question from plugin authors attempting prompt security
hardening: **why do the intuitively obvious hook points not work for post-tool
interception or prompt modification, and what does work?**

It is a prerequisite for the two companion implementation documents:

- [Combination: Post-Tool Detection with Interception](hardening-combo-intercept.md)
- [Combination: Post-Tool Detection with Prompt Modification](hardening-combo-prompt-modify.md)

---

## 1. The question

> "I want to inspect tool results in `after_tool_call` and then either intercept the
> result or modify the prompt. I also tried `before_message_write`, `tool_result_persist`,
> and `message_sending` — none of them worked for this. What is the correct approach?"

This is the right instinct — `after_tool_call` is the natural observation point — but
the wrong mental model of how hooks compose. The answer requires understanding three
separate constraints.

---

## 2. Why each hook alone is insufficient

### 2.1 `after_tool_call` — void, fire-and-forget, return value discarded

**Type signature** ([src/plugins/types.ts:2355](../../../../../src/plugins/types.ts)):

```typescript
after_tool_call: (
  event: PluginHookAfterToolCallEvent,
  ctx: PluginHookToolContext,
) => Promise<void> | void;
```

The return type is `void`. The runner calls it with `runVoidHook`, which discards all
return values. It is not awaited before subsequent pipeline stages proceed. Any value
you return is silently ignored.

**Call site** ([src/agents/pi-embedded-subscribe.handlers.tools.ts](../../../../../src/agents/pi-embedded-subscribe.handlers.tools.ts)):

```typescript
// fire-and-forget
void hookRunnerAfter.runAfterToolCall(hookEvent, {...}).catch((err) => {...});
```

The `void` keyword is explicit: the pipeline does not wait for this hook. You can
observe the tool result here, but you cannot change anything from here.

---

### 2.2 `tool_result_persist` — synchronous only, no async handlers

**Type signature** ([src/plugins/types.ts:2359](../../../../../src/plugins/types.ts)):

```typescript
tool_result_persist: (
  event: PluginHookToolResultPersistEvent,
  ctx: PluginHookToolResultPersistContext,
) => PluginHookToolResultPersistResult | void;   // no Promise
```

The handler signature has **no `Promise` return**. If you write an `async` handler, it
returns a `Promise` object immediately — which is truthy and non-void — and the runner
may misinterpret it or silently drop the result. Async logic inside a `tool_result_persist`
handler is unreliable.

This hook **can** modify the tool result (the `message` field it returns is what gets
written to the session transcript and fed back to the LLM next turn). But all logic must
be synchronous: regex, string manipulation, in-process lookup tables. No `await`, no
network calls, no file I/O.

---

### 2.3 `before_message_write` — also synchronous only

Same constraint as `tool_result_persist`. Operates on individual messages being written
to the session JSONL. Can block or rewrite a message, but handler must be synchronous.
Fires on every message type (user, assistant, tool) — not scoped to tool results.

---

### 2.4 `message_sending` — wrong layer entirely

**Type signature** ([src/plugins/types.ts:2343](../../../../../src/plugins/types.ts)):

```typescript
message_sending: (
  event: PluginHookMessageSendingEvent,
  ctx: PluginHookMessageContext,
) => Promise<PluginHookMessageSendingResult | void> | PluginHookMessageSendingResult | void;
```

This hook fires on **outbound channel messages** — replies going to Telegram, Discord,
WhatsApp, and so on. It has no connection to tool results or LLM prompt context. The
`content` field is the final text reply being delivered to the user, not the tool result
or the prompt.

---

## 3. The architectural constraint in one diagram

```
Tool call dispatched
        │
        ▼
  Tool executes
        │
        ├──► after_tool_call (fire-and-forget, void)
        │         Can OBSERVE. Cannot modify anything.
        │
        ▼
  tool_result_persist (SYNC)
        │         Can MODIFY the message written to session.
        │         Must be synchronous. No await.
        │
        ▼
  before_message_write (SYNC)
        │         Can BLOCK or REWRITE any message being written.
        │         Must be synchronous. No await.
        │
        ▼
  Session transcript updated
        │
        ▼
  Next LLM turn begins
        │
        ▼
  before_prompt_build (async, awaited)
        │         Can MODIFY what the LLM is told.
        │         Fully async. Returns prependContext / prependSystemContext /
        │         appendSystemContext / systemPrompt.
        │
        ▼
  LLM call
        │
        ▼
  LLM proposes next tool call
        │
        ▼
  before_tool_call (async, awaited)
                  Can BLOCK or APPROVE the next tool call.
                  Fully async. Returns block / blockReason / requireApproval / params.
```

The key insight: `after_tool_call` fires in the same window as `tool_result_persist`,
but `before_prompt_build` and `before_tool_call` fire in the **next window** (next LLM
turn). You cannot have a single hook that both observes the tool result and modifies the
prompt — those are in different pipeline windows.

---

## 4. The general combination pattern

The solution is a **shared-state bridge**: store detection results in an in-process Map
keyed by `runId`, then read that state in the appropriate action hook.

```typescript
// Shared state — in-process, keyed by runId
const detectionResults = new Map<string, DetectionResult>();

// Window 1: Observe
registerHook("after_tool_call", async (event, ctx) => {
  const result = detectCondition(event.toolName, event.result); // sync logic
  if (result) detectionResults.set(runId, result);
});

// Window 1 (same tick): Act on the transcript
registerHook("tool_result_persist", (event, ctx) => {
  const result = detectionResults.get(runId);
  if (result) return { message: scrubOrAnnotate(event.message, result) };
});

// Window 2: Act on the next LLM prompt
registerHook("before_prompt_build", async (_event, ctx) => {
  const result = detectionResults.get(runId);
  if (result) return { prependContext: buildDirective(result) };
});

// Window 2: Act on the next tool call
registerHook("before_tool_call", async (event, ctx) => {
  const result = detectionResults.get(runId);
  if (result) return { block: true, blockReason: buildBlockReason(result) };
});
```

---

## 5. Which hooks to use for which goal

| Security goal                           | Primary hook                              | Supporting hooks                                                 |
| --------------------------------------- | ----------------------------------------- | ---------------------------------------------------------------- |
| Detect condition in tool result         | `after_tool_call`                         | —                                                                |
| Scrub / replace tool result content     | `tool_result_persist` (sync)              | —                                                                |
| Block next tool call                    | `before_tool_call`                        | `after_tool_call` to set flag                                    |
| Modify LLM instructions for next turn   | `before_prompt_build`                     | `after_tool_call` to set flag                                    |
| Intercept (block + replace + guide LLM) | All four combined                         | See [hardening-combo-intercept.md](hardening-combo-intercept.md) |
| Prompt rewrite based on detection       | `after_tool_call` + `before_prompt_build` | `tool_result_persist` to annotate                                |

---

## 6. Critical ordering constraint

`after_tool_call` is fire-and-forget — not awaited. `tool_result_persist` fires
synchronously in the same pipeline window. Because of this, any logic in
`after_tool_call` that sets shared state must complete **synchronously** (before the
first `await`) for the flag to be visible to `tool_result_persist`.

```typescript
// ✅ Safe — state is set synchronously before first await
registerHook("after_tool_call", async (event, ctx) => {
  const result = detectCondition(event.toolName, event.result); // sync
  if (result) detectionResults.set(runId, result); // sync
  await logToAuditService(result); // async after set
});

// ❌ Unsafe — state set after await, may miss tool_result_persist window
registerHook("after_tool_call", async (event, ctx) => {
  const result = await callExternalClassifier(event.result); // async first
  if (result) detectionResults.set(runId, result); // too late for tool_result_persist
});
```

If your detection logic requires async work (network call, LLM classifier), move the
detection into `before_prompt_build` instead — which is fully async, always fires after
session write, and receives the full session history including all tool results.

---

## 7. The `allowPromptInjection` policy guard

`before_prompt_build` is classified as a prompt-injection hook
([src/plugins/registry.ts:869](../../../../../src/plugins/registry.ts)). An operator
can set:

```json
{
  "plugins": {
    "entries": {
      "my-security-plugin": {
        "hooks": { "allowPromptInjection": false }
      }
    }
  }
}
```

This will **silently block** your `before_prompt_build` registration and log a warning.
If your security plugin depends on this hook, document this requirement and verify at
startup (e.g. by logging a confirmation from inside the hook on first run).

---

## 8. Summary

The reason `after_tool_call` + any single hook does not work is that no single hook
spans both pipeline windows (observation and action). The correct architecture is:

1. **Detect** in `after_tool_call` using synchronous logic, storing results in a
   `Map<runId, DetectionResult>`.
2. **Act on the transcript** (Window 1) using `tool_result_persist` (sync scrub) or
   `before_message_write` (sync block).
3. **Act on the next LLM turn** (Window 2) using `before_prompt_build` (async prompt
   injection) and/or `before_tool_call` (async block/approval).

See the companion documents for complete implementations of each goal.
