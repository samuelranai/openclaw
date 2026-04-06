---
title: "Proposal: Content Inspection and Interception at before_dispatch"
summary: "Analysis and solution design for inbound message content inspection with LLM-assisted blocking — covering hook architecture constraints, group-chat discrimination, and async-safe interception semantics"
status: draft
---

# Proposal: Content Inspection and Interception at `before_dispatch`

## 1. Background

A plugin needs to inspect inbound user messages for sensitive content leakage and prompt
injection, intercept policy-violating messages before they reach the agent, and return an
LLM-generated explanation with remediation advice to the sender.

**Original requirements (verbatim):**

> (1) 当前实现是在 `msg_received` hook 点，但该 hook 点不支持拦截，需要在其他地方实现拦截，
> 会出现异步时序问题，导致拦截漏水。
> (2) 在 `msg_received` 点无法区分消息是单点发送还是群聊，导致部分防护场景失效。
> (3) 拦截时还希望走大模型分析，给出原因分析 + 操作建议等。

---

## 2. Architecture Constraints — Why `message_received` Cannot Be Fixed

### 2.1 Hook execution model

OpenClaw defines three hook execution models:

| Model              | Function                               | Return value                | Can block?              |
| ------------------ | -------------------------------------- | --------------------------- | ----------------------- |
| `runVoidHook`      | All handlers in parallel               | `Promise<void>` — discarded | No                      |
| `runClaimingHook`  | Sequential, first `handled: true` wins | `TResult \| undefined`      | Yes — short-circuits    |
| `runModifyingHook` | Sequential, results merged             | `TResult \| undefined`      | Yes — with `shouldStop` |

### 2.2 `message_received` is structurally void

[src/plugins/hooks.ts:617-624](../../../src/plugins/hooks.ts#L617-L624):

```ts
/**
 * Run message_received hook.
 * Runs in parallel (fire-and-forget).
 */
async function runMessageReceived(event, ctx): Promise<void> {
  return runVoidHook("message_received", event, ctx); // returns void
}
```

[src/auto-reply/reply/dispatch-from-config.ts:422-431](../../../src/auto-reply/reply/dispatch-from-config.ts#L422-L431):

```ts
// Trigger plugin hooks (fire-and-forget)
fireAndForgetHook(
  hookRunner.runMessageReceived(...),
  "dispatch-from-config: message_received plugin hook failed",
);
// pipeline continues immediately — no await, no result check
```

Two independent layers prevent interception:

1. `runVoidHook` discards all handler return values — there is no channel for a handler to
   signal "block this message."
2. The call site is `fireAndForgetHook` — the promise is not awaited before the pipeline
   continues. Even if the hook contract were changed to return a result, the call site would
   discard it.

**Consequence of the current workaround:** a plugin that sets an external flag in
`message_received` and checks it elsewhere races against the pipeline continuation.
The window between `fireAndForgetHook` and the point where the flag is checked is the
source of the reported "interception leak."

### 2.3 `llm_input` also cannot intercept

[src/agents/pi-embedded-runner/run/attempt.ts:1452-1478](../../../src/agents/pi-embedded-runner/run/attempt.ts#L1452-L1478):

```ts
if (hookRunner?.hasHooks("llm_input")) {
  hookRunner
    .runLlmInput(...)
    .catch((err) => { log.warn(`llm_input hook failed: ${String(err)}`); });
  // fire-and-forget via .catch() — not awaited
}
// LLM call proceeds immediately
```

Additionally, `llm_input` fires deep inside the agent loop — after ACP dispatch has
already accepted the message and started a run. Blocking at this point would require
aborting an in-progress run, which is a much heavier operation than declining at the
message ingress.

### 2.4 `before_dispatch` is the correct hook point

[src/auto-reply/reply/dispatch-from-config.ts:553-588](../../../src/auto-reply/reply/dispatch-from-config.ts#L553-L588):

```ts
if (hookRunner?.hasHooks("before_dispatch")) {
  const beforeDispatchResult = await hookRunner.runBeforeDispatch(
    // true await
    {
      content: hookContext.content, // raw message text
      body: hookContext.bodyForAgent ?? hookContext.body,
      channel: hookContext.channelId,
      sessionKey: sessionStoreEntry.sessionKey ?? sessionKey,
      senderId: hookContext.senderId,
      isGroup: hookContext.isGroup, // group/DM discrimination
      timestamp: hookContext.timestamp,
    },
    { channelId, accountId, conversationId, sessionKey, senderId },
  );
  if (beforeDispatchResult?.handled) {
    const text = beforeDispatchResult.text;
    if (text) {
      await sendFinalPayload({ text }); // reply sent to user
    }
    return { queuedFinal, counts }; // pipeline terminates here
  }
}
// only reaches here if handled === false/undefined
```

[src/plugins/hooks.ts:627-641](../../../src/plugins/hooks.ts#L627-L641):

```ts
async function runBeforeDispatch(event, ctx): Promise<PluginHookBeforeDispatchResult | undefined> {
  return runClaimingHook<"before_dispatch", PluginHookBeforeDispatchResult>(
    "before_dispatch",
    event,
    ctx,
  );
}
```

`runClaimingHook` runs handlers sequentially; the first to return `{ handled: true }` wins
and the remaining handlers are skipped. Because the call site `await`s the result, there is
no race window.

---

## 3. Solution Design

### 3.1 Hook choice

Use `before_dispatch` for all three requirements. No new hook point is needed. No source
changes are required for the core pipeline — the capability already exists.

### 3.2 Requirement (1) — Reliable interception

`before_dispatch` provides deterministic, synchronous-from-the-pipeline's-perspective
interception:

```
inbound message
      │
      ▼
fireAndForgetHook(message_received)  ← observation only, no block capability
      │
      ▼
await runBeforeDispatch(event, ctx)   ← ✅ awaited claiming hook
      │
      ├─ handler returns { handled: true, text }
      │       │
      │       └─ sendFinalPayload(text) → reply to user
      │          return early — agent loop never starts
      │
      └─ no handler claims → agent loop starts normally
```

**No timing gap.** The pipeline cannot proceed past `before_dispatch` until all registered
handlers have run and none has claimed the message.

### 3.3 Requirement (2) — Group vs single-chat discrimination

`PluginHookBeforeDispatchEvent` already carries `isGroup`:

[src/plugins/types.ts:2018-2033](../../../src/plugins/types.ts#L2018-L2033):

```ts
export type PluginHookBeforeDispatchEvent = {
  content: string;
  body?: string;
  channel?: string;
  sessionKey?: string;
  senderId?: string;
  isGroup?: boolean; // ← already present
  timestamp?: number;
};
```

`isGroup` is derived at
[src/hooks/message-hook-mappers.ts:79](../../../src/hooks/message-hook-mappers.ts#L79)
from `ctx.GroupSubject || ctx.GroupChannel`, which is populated by every channel adapter
(WhatsApp, Telegram, Discord, etc.) before reaching the dispatch layer. The field is
cross-channel and reliable.

**Minor gap — `groupId` not forwarded.** The internal `hookContext` has `groupId`
(the conversation ID when `isGroup=true`), but the current `PluginHookBeforeDispatchEvent`
does not expose it. See Section 4.1 for the patch.

### 3.4 Requirement (3) — LLM analysis within the handler

Because `runClaimingHook` awaits each handler as an async function, the plugin can make
its own LLM API call inside the handler and the pipeline will wait for it to complete:

```ts
// Plugin registration — conceptual skeleton
openclaw.hooks.register(
  "before_dispatch",
  async (event, ctx) => {
    // Step 1: fast local scan (regex / keyword list)
    const localRisk = localContentScan(event.content);
    if (!localRisk.flagged) return; // pass through — no allocation

    // Step 2: LLM-assisted deep analysis
    const analysis = await callAnalysisLlm({
      systemPrompt: INSPECTION_SYSTEM_PROMPT,
      userContent: event.content,
      riskSignals: localRisk.signals,
      isGroup: event.isGroup,
      channel: event.channel,
    });

    if (!analysis.shouldBlock) return; // LLM determined it's safe

    // Step 3: intercept — return analysis as reply
    return {
      handled: true,
      text: formatBlockReply({
        reason: analysis.reason,
        suggestions: analysis.suggestions,
        riskClass: analysis.riskClass,
      }),
    };
  },
  { priority: 100 },
); // high priority — runs before other handlers
```

The handler is async and awaited by the framework. The LLM call completes before the
pipeline either terminates (blocked) or continues (not flagged). No race condition.

---

## 4. Required Source Changes

### 4.1 Expose `groupId` in `before_dispatch` event (optional but recommended)

**Why:** plugins may need to apply different policies per group (e.g., stricter rules in
certain Telegram groups than others).

**Change 1 — type definition** ([src/plugins/types.ts](../../../src/plugins/types.ts)):

```ts
export type PluginHookBeforeDispatchEvent = {
  content: string;
  body?: string;
  channel?: string;
  sessionKey?: string;
  senderId?: string;
  isGroup?: boolean;
  groupId?: string; // ← add: conversation ID when isGroup=true
  timestamp?: number;
};
```

**Change 2 — call site** ([src/auto-reply/reply/dispatch-from-config.ts:553-572](../../../src/auto-reply/reply/dispatch-from-config.ts#L553-L572)):

```ts
const beforeDispatchResult = await hookRunner.runBeforeDispatch(
  {
    content: hookContext.content,
    body: hookContext.bodyForAgent ?? hookContext.body,
    channel: hookContext.channelId,
    sessionKey: sessionStoreEntry.sessionKey ?? sessionKey,
    senderId: hookContext.senderId,
    isGroup: hookContext.isGroup,
    groupId: hookContext.groupId,    // ← add
    timestamp: hookContext.timestamp,
  },
  ...
);
```

`hookContext.groupId` is already set at
[src/hooks/message-hook-mappers.ts:126](../../../src/hooks/message-hook-mappers.ts#L126)
as `isGroup ? conversationId : undefined`.

This is a **non-breaking, additive change** — existing handlers that don't use `groupId`
are unaffected.

### 4.2 No other source changes required

`before_dispatch` is already a `runClaimingHook`, already awaited, and the result
already drives `sendFinalPayload`. The plugin can implement all three requirements
without any further framework changes.

---

## 5. Plugin Implementation Guidelines

### 5.1 Recommended two-phase detection

Running a full LLM call on every message is expensive. A two-phase approach is preferred:

```
Phase 1 — Local scan (< 1 ms)
  ├── keyword/regex matching
  ├── injection pattern detection (ignore instructions, act as, etc.)
  └── if clean → return undefined (no interception, no cost)

Phase 2 — LLM analysis (only when Phase 1 flags)
  ├── structured prompt with risk signals as context
  ├── request: reason + risk class + user-facing suggestions
  └── if LLM clears → return undefined
      if LLM confirms → return { handled: true, text: formattedReply }
```

### 5.2 Group-aware policy

```ts
async (event, ctx) => {
  if (event.isGroup) {
    // Group messages: apply group-specific policy
    // - may use event.groupId to load per-group config
    // - consider: only scan @-mentioned messages vs all group traffic
    const policy = await loadGroupPolicy(event.groupId);
    if (!policy.inspectAll && !wasMentioned(event)) return;
  }
  // ... detection logic
};
```

### 5.3 Reply format for blocked messages

The `text` field in `PluginHookBeforeDispatchResult` is sent directly to the user via
`sendFinalPayload`. Structure the LLM output so the reply is actionable:

```
⚠️ 消息被安全策略拦截

触发原因：[LLM 生成的具体原因]

风险类别：[敏感信息泄露 / 提示词注入 / ...]

操作建议：
• [建议1]
• [建议2]

如有疑问，请联系管理员。
```

### 5.4 Timeout and failure posture

`before_dispatch` handlers that throw are caught by `runClaimingHooksList`
([src/plugins/hooks.ts:391-394](../../../src/plugins/hooks.ts#L391-L394)):

```ts
} catch (err) {
  handleHookError({ hookName, pluginId: hook.pluginId, error: err });
}
```

On error, the hook framework logs the error and moves to the next handler — it does
**not** block the message. This is a **fail-open** posture for the detection plugin.

If the requirement is **fail-closed** (block on detection error), the plugin must handle
its own errors and return `{ handled: true, text: errorReply }` rather than throwing:

```ts
async (event, ctx) => {
  try {
    const result = await runDetection(event);
    if (result.block) return { handled: true, text: result.reply };
  } catch (err) {
    log.error("content-inspection failed", err);
    // fail-closed: block the message with a generic notice
    return { handled: true, text: GENERIC_BLOCK_NOTICE };
    // fail-open alternative: return undefined (let message through)
  }
};
```

Document the chosen posture explicitly in the plugin.

---

## 6. Architecture Comparison

| Dimension               | `message_received` (current)           | `before_dispatch` (proposed)                     |
| ----------------------- | -------------------------------------- | ------------------------------------------------ |
| Hook execution          | `runVoidHook` — parallel, void         | `runClaimingHook` — sequential, awaited          |
| Call site               | `fireAndForgetHook` — not awaited      | `await hookRunner.runBeforeDispatch(...)`        |
| Can intercept           | ❌ No                                  | ✅ Yes — `{ handled: true }` terminates pipeline |
| Timing safety           | Race condition (flag-based workaround) | No race — pipeline blocked until hook returns    |
| `isGroup` available     | ❌ Not in event type                   | ✅ Already in event type                         |
| `groupId` available     | ❌ Not in event type                   | ⚠️ Requires 2-line patch (Section 4.1)           |
| LLM call inside handler | Unsafe (result discarded)              | ✅ Safe — handler awaited                        |
| Reply to user           | Not supported natively                 | ✅ `text` field routed to `sendFinalPayload`     |
| Failure posture         | Fail-open by design                    | Fail-open by default; fail-closed possible       |

---

## 7. Limitations and Future Work

### 7.1 Binary dispatch semantics

`before_dispatch` is either-or: intercept (`handled: true`) or pass through. There is
currently no way to **pass the message through while injecting risk context into the
agent's system prompt** (e.g., "this message was scanned and found low-risk but contains
pattern X — please be cautious").

If this "soft annotation" use case is needed, `PluginHookBeforeDispatchResult` would need
a new field such as `systemContextAnnotation?: string`, and the dispatch pipeline would
need to propagate it into the agent's context window. This is a separate feature request.

### 7.2 First-claim-wins vs. multi-inspector composition

Multiple plugins can register `before_dispatch` handlers. `runClaimingHook` stops at the
first `{ handled: true }` result — subsequent plugins do not run. If multiple independent
inspectors must all have a chance to examine the message (e.g., a DLP plugin and a prompt
injection plugin), they should either:

- Be combined into a single plugin handler, or
- Use `runModifyingHook` semantics (accumulating results), which would require promoting
  `before_dispatch` from claiming to modifying — a more significant change.

For the immediate requirement, a single plugin covering both detection cases in one
handler is the simplest path.

### 7.3 No access to session history at `before_dispatch`

Unlike `before_tool_call` (which could be extended with cross-call history for STAC
detection), `before_dispatch` sees only the current inbound message. Stateful detection
(e.g., detecting a multi-turn prompt injection that spans several messages) would require
the plugin to maintain its own session-scoped state, keyed on `event.sessionKey`.

---

## 8. Implementation Checklist

- [ ] Plugin: implement Phase 1 local scan (keyword list, injection patterns)
- [ ] Plugin: implement Phase 2 LLM analysis prompt and response parser
- [ ] Plugin: register `before_dispatch` handler with appropriate priority
- [ ] Plugin: decide and document fail-open vs fail-closed posture
- [ ] Plugin: implement group-aware policy gating (`event.isGroup`, `event.groupId`)
- [ ] Core (optional): land `groupId` addition to `PluginHookBeforeDispatchEvent` — 2 files, additive
- [ ] Plugin: write tests covering: flagged message blocked, clean message passes through, LLM call failure behavior, group vs DM policy
- [ ] Ops: configure LLM provider credentials for the detection plugin separately from the main agent provider

---

## 9. References

| File                                                                                                                        | Relevance                                                                             |
| --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| [src/plugins/hooks.ts:617-641](../../../src/plugins/hooks.ts#L617-L641)                                                     | `runMessageReceived` (void) and `runBeforeDispatch` (claiming) implementations        |
| [src/plugins/hooks.ts:264-334](../../../src/plugins/hooks.ts#L264-L334)                                                     | `runVoidHook` and `runClaimingHook` execution models                                  |
| [src/auto-reply/reply/dispatch-from-config.ts:422-588](../../../src/auto-reply/reply/dispatch-from-config.ts#L422-L588)     | Call sites: `fireAndForgetHook(message_received)` and `await runBeforeDispatch`       |
| [src/plugins/types.ts:2017-2048](../../../src/plugins/types.ts#L2017-L2048)                                                 | `PluginHookBeforeDispatchEvent` and `PluginHookBeforeDispatchResult` type definitions |
| [src/hooks/message-hook-mappers.ts:79](../../../src/hooks/message-hook-mappers.ts#L79)                                      | `isGroup` derivation from `GroupSubject \|\| GroupChannel`                            |
| [src/agents/pi-embedded-runner/run/attempt.ts:1452-1478](../../../src/agents/pi-embedded-runner/run/attempt.ts#L1452-L1478) | `llm_input` call site (fire-and-forget — not suitable for interception)               |
| [docs/security/study/plugin-security-hooks.md](plugin-security-hooks.md)                                                    | `before_tool_call` and `before_skill_install` design reference                        |
