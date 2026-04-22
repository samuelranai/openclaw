---
title: "Session 4 — Message Flow Hooks"
summary: "Source walkthrough of the five message flow hooks: inbound_claim (channel routing), message_received (observability), before_dispatch (pre-agent intercept), message_sending (outbound DLP gate), and message_sent (delivery confirmation) — with full dispatch-from-config.ts and deliver.ts pipeline traces"
read_when:
  - Building a channel plugin that claims inbound messages
  - Implementing a before_dispatch handler to intercept or short-circuit agent invocation
  - Adding outbound content policy via message_sending cancel or rewrite
  - Understanding the full message lifecycle from arrival to delivery
---

# Session 4 — Message Flow Hooks

This session covers the five hooks that sit on the message lifecycle path — from when
an inbound message arrives at the gateway to when the agent's reply is delivered to
the channel. Two of these hooks can intercept and stop the pipeline entirely
(`inbound_claim` and `before_dispatch`); one can cancel or rewrite outbound content
(`message_sending`).

**Prerequisite:** Sessions 1–2. Familiarity with `dispatch-from-config.ts` from the
[Gateway Session 7](../../gateway/security/study/sessions/session7-sessions-and-channels.md) is helpful but not required.

---

## 1. What this session covers

| File                                           | Role                                                                                    |
| ---------------------------------------------- | --------------------------------------------------------------------------------------- |
| `src/auto-reply/reply/dispatch-from-config.ts` | Main inbound dispatch pipeline — `inbound_claim`, `message_received`, `before_dispatch` |
| `src/infra/outbound/deliver.ts`                | Outbound delivery pipeline — `message_sending`, `message_sent`                          |
| `src/plugins/types.ts`                         | All event, context, and result type definitions                                         |
| `src/plugins/hooks.ts`                         | Runner function implementations (lines 579–681)                                         |

---

## 2. The full message lifecycle

```
Inbound message arrives at gateway
  │
  ├─► [Plugin-bound routing check]
  │     └─► inbound_claim (Claiming) ── Session 4 §3
  │               first { handled:true } claims the message
  │               → if claimed: message handled by plugin, pipeline stops
  │               → if not claimed: falls through to standard dispatch
  │
  ├─► message_received (Void) ────────── Session 4 §4
  │     fire-and-forget, logging only
  │
  ├─► [Command parsing, fast-abort check, ACP routing]
  │
  ├─► before_dispatch (Claiming) ─────── Session 4 §5
  │     first { handled:true } short-circuits model dispatch
  │     → if handled: reply with result.text (or no reply), pipeline stops
  │     → if not handled: agent run begins
  │
  └─► Agent run executes (Sessions 2, 3, 5)
        │
        └─► Outbound reply delivered
              │
              ├─► message_sending (Modifying) ── Session 4 §6
              │     can rewrite content or cancel delivery
              │
              └─► message_sent (Void) ────────── Session 4 §7
                    fire-and-forget confirmation
```

---

## 3. `inbound_claim`

**Hook name:** `inbound_claim`
**Execution model:** Claiming
**Call site:** [src/auto-reply/reply/dispatch-from-config.ts:359](../../../../../src/auto-reply/reply/dispatch-from-config.ts)

### 3a. What it does

Channel plugins can create persistent **conversation bindings** — a record that associates
a conversation ID with a plugin ID. When an inbound message arrives for a conversation
that has a plugin binding, `inbound_claim` fires to let that plugin's handler process
the message before the default agent dispatch occurs.

This is the mechanism by which plugins like `msteams` or `matrix` handle their own
threading, reactions, and replies without going through the standard OpenClaw agent loop.

### 3b. Call site walkthrough

The dispatch pipeline first checks whether the inbound conversation has a plugin binding:

```typescript
// src/auto-reply/reply/dispatch-from-config.ts:354
if (pluginOwnedBinding) {
  const targetedClaimOutcome = hookRunner?.runInboundClaimForPluginOutcome
    ? await hookRunner.runInboundClaimForPluginOutcome(
        pluginOwnedBinding.pluginId,   // only this plugin's handlers are invoked
        inboundClaimEvent,
        inboundClaimContext,
      )
    : /* fallback: check if plugin is loaded */;

  switch (targetedClaimOutcome.status) {
    case "handled":
      // message fully handled by plugin — stop pipeline
      recordProcessed("completed", { reason: "plugin-bound-handled" });
      return { queuedFinal: false, counts: ... };

    case "missing_plugin":
    case "no_handler":
      // plugin is gone or doesn't handle this — fall through with notice
      pluginFallbackReason = ...;
      break;

    case "declined":
      // plugin explicitly declined — send notice to user
      await sendBindingNotice(...);
      return ...;

    case "error":
      // plugin errored — send error notice, stop pipeline
      await sendBindingNotice(...);
      return ...;
  }
}
```

Three variants of the claiming runner exist:

| Variant                                                 | Used when                    | What it does                                                         |
| ------------------------------------------------------- | ---------------------------- | -------------------------------------------------------------------- | -------------- | ---------- | -------- | ------ |
| `runInboundClaim(event, ctx)`                           | No specific plugin target    | All registered `inbound_claim` handlers, first `{handled:true}` wins |
| `runInboundClaimForPlugin(pluginId, event, ctx)`        | Plugin ID known, need result | Only that plugin's handlers                                          |
| `runInboundClaimForPluginOutcome(pluginId, event, ctx)` | Need full status enum        | Returns `handled                                                     | missing_plugin | no_handler | declined | error` |

`runInboundClaimForPluginOutcome` is used in the main dispatch path because the full
status enum is needed to decide how to handle the fallback scenario — the simpler
`runInboundClaimForPlugin` only returns the `PluginHookInboundClaimResult` without the
additional status cases.

### 3c. Event shape

```typescript
// src/plugins/types.ts:1992
type PluginHookInboundClaimEvent = {
  content: string; // raw message text
  body?: string; // body after command parsing
  bodyForAgent?: string; // cleaned body for agent input
  transcript?: string;
  timestamp?: number;
  channel: string; // e.g. "telegram", "discord", "whatsapp"
  accountId?: string;
  conversationId?: string;
  parentConversationId?: string;
  senderId?: string;
  senderName?: string;
  senderUsername?: string;
  threadId?: string | number;
  messageId?: string;
  isGroup: boolean;
  commandAuthorized?: boolean; // whether sender is authorized for commands
  wasMentioned?: boolean;
  metadata?: Record<string, unknown>;
};
```

### 3d. Context shape

```typescript
// src/plugins/types.ts:1986
type PluginHookInboundClaimContext = PluginHookMessageContext & {
  parentConversationId?: string;
  senderId?: string;
  messageId?: string;
};

type PluginHookMessageContext = {
  channelId: string;
  accountId?: string;
  conversationId?: string;
};
```

### 3e. Result shape

```typescript
// src/plugins/types.ts:2013
type PluginHookInboundClaimResult = {
  handled: boolean;
};
```

Return `{ handled: true }` to claim the message. Return `{ handled: false }` or
`undefined` to pass. The first handler that returns `{ handled: true }` stops the
claiming chain — lower-priority handlers are not called.

---

## 4. `message_received`

**Hook name:** `message_received`
**Execution model:** Void
**Call site:** [src/auto-reply/reply/dispatch-from-config.ts:424](../../../../../src/auto-reply/reply/dispatch-from-config.ts)

### 4a. When it fires

Fires after the inbound claim phase, before agent dispatch. This is fire-and-forget — the
pipeline does not wait for handlers. Use it for logging, analytics, SIEM ingestion, or
side-effect notifications that do not need to influence the pipeline.

```typescript
// src/auto-reply/reply/dispatch-from-config.ts:423
if (hookRunner?.hasHooks("message_received")) {
  fireAndForgetHook(
    hookRunner.runMessageReceived(
      toPluginMessageReceivedEvent(hookContext),
      toPluginMessageContext(hookContext),
    ),
    "dispatch-from-config: message_received plugin hook failed",
  );
}
```

`fireAndForgetHook` is a wrapper that calls `void promise.catch(log.warn)`. The message
pipeline never awaits this — by the time all `message_received` handlers finish, the
agent run may already be complete.

### 4b. Event shape

```typescript
// src/plugins/types.ts:2051
type PluginHookMessageReceivedEvent = {
  from: string; // sender identifier
  content: string; // message content
  timestamp?: number;
  metadata?: Record<string, unknown>;
};
```

Notice this is a simplified view compared to `inbound_claim`. It does not include
`commandAuthorized`, `threadId`, or most of the channel-specific metadata. If you need
the richer context, use `inbound_claim` (though with the tradeoff that you can intercept
the pipeline).

---

## 5. `before_dispatch`

**Hook name:** `before_dispatch`
**Execution model:** Claiming
**Call site:** [src/auto-reply/reply/dispatch-from-config.ts:554](../../../../../src/auto-reply/reply/dispatch-from-config.ts)

### 5a. What it does

`before_dispatch` is the **last synchronization point before the agent LLM is invoked**.
A plugin that returns `{ handled: true }` from this hook:

1. Prevents the agent run from starting entirely
2. Can optionally provide a `text` reply to send back to the user
3. Causes the message to be recorded as `reason: "before_dispatch_handled"`

This is the primary hook for implementing:

- Rate limiting per sender or conversation
- Static response bots (FAQ bots, command handlers)
- Content policy gates that reject messages before they reach the LLM
- A/B routing to different agent configurations

### 5b. Call site walkthrough

```typescript
// src/auto-reply/reply/dispatch-from-config.ts:553
if (hookRunner?.hasHooks("before_dispatch")) {
  const beforeDispatchResult = await hookRunner.runBeforeDispatch(
    {
      content: hookContext.content,
      body: hookContext.bodyForAgent ?? hookContext.body,
      channel: hookContext.channelId,
      sessionKey: sessionStoreEntry.sessionKey ?? sessionKey,
      senderId: hookContext.senderId,
      isGroup: hookContext.isGroup,
      timestamp: hookContext.timestamp,
    },
    {
      channelId: hookContext.channelId,
      accountId: hookContext.accountId,
      conversationId: inboundClaimContext.conversationId,
      sessionKey: sessionStoreEntry.sessionKey ?? sessionKey,
      senderId: hookContext.senderId,
    },
  );

  if (beforeDispatchResult?.handled) {
    const text = beforeDispatchResult.text;
    let queuedFinal = false;
    if (text) {
      const handledReply = await sendFinalPayload({ text });
      queuedFinal = handledReply.queuedFinal;
    }
    recordProcessed("completed", { reason: "before_dispatch_handled" });
    markIdle("message_completed");
    return { queuedFinal, counts }; // pipeline stops here
  }
}
// agent run begins here if not handled
```

Unlike `message_received` (fire-and-forget), this call is fully awaited. The pipeline
blocks until all handlers return.

### 5c. Event shape

```typescript
// src/plugins/types.ts:2018
type PluginHookBeforeDispatchEvent = {
  content: string; // raw message content
  body?: string; // body after command parsing (the text the agent would see)
  channel?: string; // channel identifier
  sessionKey?: string;
  senderId?: string;
  isGroup?: boolean;
  timestamp?: number;
};
```

### 5d. Context shape

```typescript
// src/plugins/types.ts:2035
type PluginHookBeforeDispatchContext = {
  channelId?: string;
  accountId?: string;
  conversationId?: string;
  sessionKey?: string;
  senderId?: string;
};
```

### 5e. Result shape

```typescript
// src/plugins/types.ts:2043
type PluginHookBeforeDispatchResult = {
  handled: boolean; // true = skip agent dispatch
  text?: string; // optional reply text (only used if handled=true)
};
```

Return `{ handled: false }` or `undefined` to let the agent run proceed.

**Example — rate limit by sender:**

```typescript
runtime.hooks.register("before_dispatch", {
  priority: 200,
  handler: async (event, ctx) => {
    if (rateLimiter.isExceeded(ctx.senderId)) {
      return {
        handled: true,
        text: "You have reached your message limit. Please try again later.",
      };
    }
  },
});
```

---

## 6. `message_sending`

**Hook name:** `message_sending`
**Execution model:** Modifying
**Call site:** [src/infra/outbound/deliver.ts:443](../../../../../src/infra/outbound/deliver.ts)

### 6a. What it does

`message_sending` fires on the **outbound delivery path**, after the agent run has
produced a reply, before it is sent to the channel. Handlers can:

- **Rewrite** the outbound content (DLP scrubbing, format normalization)
- **Cancel** delivery entirely (policy violation, rate limiting, silent drop)

### 6b. Call site walkthrough

```typescript
// src/infra/outbound/deliver.ts:422
async function applyMessageSendingHook(params) {
  if (!params.enabled) {
    return { cancelled: false, payload: params.payload, ... };
  }
  try {
    const sendingResult = await params.hookRunner!.runMessageSending(
      {
        to: params.to,
        content: params.payloadSummary.text,    // outbound text content
        metadata: {
          channel: params.channel,
          accountId: params.accountId,
          mediaUrls: params.payloadSummary.mediaUrls,
        },
      },
      {
        channelId: params.channel,
        accountId: params.accountId ?? undefined,
      },
    );

    if (sendingResult?.cancel) {
      return {
        cancelled: true,
        payload: params.payload,
        payloadSummary: params.payloadSummary,
      };
    }

    if (sendingResult?.content !== undefined) {
      // apply rewritten content to the payload
      ...
    }
  } catch (err) {
    log.warn(`message_sending hook failed: ${String(err)}`);
    // failure is non-fatal — original payload is used
  }
}
```

Hook failures are non-fatal: if all handlers throw, the original payload is delivered
unchanged. This matches the general principle that hook errors should not disrupt message
delivery.

### 6c. Event shape

```typescript
// src/plugins/types.ts:2059
type PluginHookMessageSendingEvent = {
  to: string; // recipient/channel address
  content: string; // outbound text (what the agent said)
  metadata?: {
    channel: string; // e.g. "telegram", "discord"
    accountId?: string;
    mediaUrls?: string[]; // media attachments if any
  };
};
```

### 6d. Context shape

`PluginHookMessageContext`:

```typescript
{
  channelId: string;
  accountId?: string;
  conversationId?: string;
}
```

### 6e. Result shape and merge

```typescript
// src/plugins/types.ts:2065
type PluginHookMessageSendingResult = {
  content?: string; // replacement text — if set, replaces event.content
  cancel?: boolean; // if true, delivery is suppressed
};
```

Merge strategy from `hooks.ts:657`:

```typescript
mergeResults: (acc, next) => {
  if (acc?.cancel === true) return acc;  // cancel is sticky — cannot be unset
  return {
    content: lastDefined(acc?.content, next.content),  // last plugin to set content wins
    cancel: stickyTrue(acc?.cancel, next.cancel),
  };
},
shouldStop: (result) => result.cancel === true,  // chain stops on first cancel
```

**`cancel` is sticky:** once any handler sets `cancel: true`, the chain stops and no
lower-priority handler can un-cancel. This is intentional — a security policy that
cancels a message should not be overrideable by a lower-trust plugin.

**`content` uses `lastDefined`:** lower-priority handlers can override a content rewrite
from a higher-priority one. This asymmetry (cancel = high-priority wins, content =
low-priority wins) means your DLP plugin should be at a lower priority number than
formatting plugins.

---

## 7. `message_sent`

**Hook name:** `message_sent`
**Execution model:** Void
**Call site:** [src/infra/outbound/deliver.ts:390](../../../../../src/infra/outbound/deliver.ts)

### 7a. When it fires

Fires after the outbound channel delivery attempt — both on success and failure.
Fire-and-forget via `fireAndForgetHook`.

```typescript
// src/infra/outbound/deliver.ts:388
if (hasMessageSentHooks) {
  fireAndForgetHook(
    params.hookRunner!.runMessageSent(
      toPluginMessageSentEvent(canonical),
      toPluginMessageContext(canonical),
    ),
    "deliverOutboundPayloads: message_sent plugin hook failed",
    (message) => {
      log.warn(message);
    },
  );
}
```

`hasMessageSentHooks` is pre-computed as `hookRunner?.hasHooks("message_sent") ?? false`
before the delivery loop runs, avoiding repeated `hasHooks` calls for multi-recipient
messages.

### 7b. Event shape

```typescript
// src/plugins/types.ts:2071
type PluginHookMessageSentEvent = {
  to: string;
  content: string; // final text that was sent (after message_sending rewrites)
  success: boolean; // whether channel delivery succeeded
  error?: string; // delivery error if success=false
};
```

`content` reflects the **final delivered text** — if `message_sending` rewrote the
content, this hook sees the rewritten version, not the original. This allows `message_sent`
handlers to log exactly what was delivered.

---

## 8. Pipeline ordering and interaction between hooks

Understanding how these five hooks interact requires knowing their relative order in the
dispatch pipeline:

```
1. inbound_claim     — can claim and stop before agent runs
2. message_received  — fire-and-forget (does not block)
3. [fast-abort check, ACP routing, command parsing]
4. before_dispatch   — can handle and stop before agent runs (awaited)
5. [agent run + all agent lifecycle and tool hooks]
6. message_sending   — can cancel or rewrite outbound reply (awaited)
7. message_sent      — fire-and-forget confirmation
```

Two important interactions:

**`before_dispatch` does not fire if `inbound_claim` claimed the message.** If your
plugin claims a message in `inbound_claim`, the `before_dispatch` and `message_sending`
hooks for that message are never reached.

**`message_sending` sees the post-agent content.** If you want to gate a message before
the LLM sees it, use `before_dispatch`. If you want to gate or scrub the LLM's reply,
use `message_sending`.

---

## 9. Exercises

### Exercise 1 — Trace a message through the full 5-hook pipeline

1. Register all five hooks with handlers that log `"[HOOK_NAME] fired"`.
2. Set breakpoints at each of the five call sites listed in Section 1.
3. Send a message from a connected client.
4. Observe the order the breakpoints fire and which hooks are fire-and-forget vs awaited.
5. Note: `message_received` fires before `before_dispatch` is awaited, so the log may
   appear in a different order than the call sequence.

### Exercise 2 — Implement a rate limiter via `before_dispatch`

1. Register `before_dispatch` with a simple in-memory rate limiter keyed on
   `ctx.senderId`:
   ```typescript
   const counts = new Map<string, number>();
   runtime.hooks.register("before_dispatch", {
     priority: 100,
     handler: async (event, ctx) => {
       const key = ctx.senderId ?? "anon";
       const count = (counts.get(key) ?? 0) + 1;
       counts.set(key, count);
       if (count > 5) {
         return { handled: true, text: "Rate limit exceeded." };
       }
     },
   });
   ```
2. Send 6+ messages in quick succession.
3. Verify that messages 1–5 reach the agent and message 6+ gets the rate limit reply.
4. Check that `agent_end` is NOT called for the rate-limited messages (no agent run
   was started).

### Exercise 3 — DLP scrub via `message_sending`

1. Register `message_sending` with a handler that redacts any 9-digit sequences
   (simulating SSN scrubbing):
   ```typescript
   handler: async (event) => {
     const scrubbed = event.content.replace(/\b\d{9}\b/g, "[REDACTED]");
     if (scrubbed !== event.content) {
       return { content: scrubbed };
     }
   };
   ```
2. Send a message like "My SSN is 123456789, please help."
3. Observe that the agent's reply that contains the number is delivered with
   `[REDACTED]` substituted.
4. Verify in `message_sent` that `event.content` contains `[REDACTED]` (not the original).

### Exercise 4 — Compare `message_sending` cancel with `before_dispatch` intercept

1. Register `before_dispatch` returning `{ handled: true }` for messages containing "stop".
2. Register `message_sending` returning `{ cancel: true }` for replies containing "secret".
3. Send "stop this" — the agent never runs; `message_sending` does not fire.
4. Send "tell me a secret" — the agent runs, produces a reply; `message_sending` cancels
   delivery. The user sees no reply.
5. Check gateway metrics: the "stop" case shows `reason: "before_dispatch_handled"`;
   the "secret" case shows a delivered=false outcome.

---

## See Also

- [Session 5 — Tool Execution Hooks](session5-tool-execution-hooks.md)
- [Session 6 — Session, Subagent, and Gateway Hooks](session6-session-subagent-gateway-hooks.md)
- [Session 7 — Security Analysis](session7-security-analysis.md) — `before_dispatch` as a policy gate
- [Gateway Session 7 — Sessions and Channels](../../gateway/security/study/sessions/session7-sessions-and-channels.md) — full inbound routing context
