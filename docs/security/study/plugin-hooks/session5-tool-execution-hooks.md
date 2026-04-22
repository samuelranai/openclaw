---
title: "Session 5 — Tool Execution Hooks"
summary: "Deep dive into the four tool execution hooks: before_tool_call (param rewrite, block, and the requireApproval flow with gateway RPC), after_tool_call, tool_result_persist (sync transcript transform), and before_message_write (sync per-message block/rewrite) — with full source walkthrough of pi-tools.before-tool-call.ts and session-tool-result-guard-wrapper.ts"
read_when:
  - Building a security plugin that blocks or approves dangerous tool calls
  - Understanding the requireApproval gateway RPC flow and AbortSignal race
  - Implementing transcript scrubbing via tool_result_persist or before_message_write
  - Investigating why a Sync hook handler's result was silently ignored
---

# Session 5 — Tool Execution Hooks

This session covers the four hooks that surround tool call execution. Two are Modifying
(`before_tool_call`) and Void (`after_tool_call`) hooks on the async tool dispatch path.
Two are Sync hooks (`tool_result_persist`, `before_message_write`) on the synchronous
session transcript write path. These are the two most security-sensitive hook positions
in the gateway.

**Prerequisite:** Sessions 1–2. [Gateway Session 6 — Tool Dispatch](../../gateway/security/study/sessions/session6-tool-dispatch.md)
for the `runBeforeToolCallHook` wrapper context.

---

## 1. What this session covers

| File                                              | Role                                                                             |
| ------------------------------------------------- | -------------------------------------------------------------------------------- |
| `src/agents/pi-tools.before-tool-call.ts`         | `runBeforeToolCallHook` — the wrapper called by all three tool dispatch surfaces |
| `src/agents/session-tool-result-guard-wrapper.ts` | `guardSessionManager` — wires `tool_result_persist` and `before_message_write`   |
| `src/plugins/types.ts`                            | All event, context, and result type definitions                                  |
| `src/plugins/hooks.ts`                            | Runner implementations (lines 692–872)                                           |

---

## 2. The three tool dispatch surfaces

All three paths that can invoke a tool share the same `runBeforeToolCallHook` function:

```
LLM-initiated tool call (in agent loop)
  └─► pi-tools.before-tool-call.ts: runBeforeToolCallHook()

HTTP POST /tools/invoke
  └─► gateway/tools-invoke-http.ts: runBeforeToolCallHook()

WS node.invoke
  └─► gateway/server-methods/nodes.ts: runBeforeToolCallHook() (via sanitizeSystemRunParamsForForwarding)
```

Registering `before_tool_call` gives **universal coverage** — there is no tool dispatch
path that bypasses this hook.

---

## 3. `before_tool_call`

**Hook name:** `before_tool_call`
**Execution model:** Modifying
**Call site:** [src/agents/pi-tools.before-tool-call.ts:194](../../../../../src/agents/pi-tools.before-tool-call.ts)

### 3a. The `runBeforeToolCallHook` wrapper

The public-facing entry point is not `hookRunner.runBeforeToolCall` but the higher-level
`runBeforeToolCallHook` wrapper in `pi-tools.before-tool-call.ts`. This wrapper handles
the full flow: running the hook, processing the result, executing the approval flow,
and returning a `{ blocked, params }` object to the dispatcher.

```typescript
// src/agents/pi-tools.before-tool-call.ts:180
if (!hookRunner?.hasHooks("before_tool_call")) {
  return { blocked: false, params: args.params };
}

const normalizedParams = isPlainObject(params) ? params : {};
const toolContext = {
  toolName,
  ...(args.ctx?.agentId && { agentId: args.ctx.agentId }),
  ...(args.ctx?.sessionKey && { sessionKey: args.ctx.sessionKey }),
  ...(args.ctx?.sessionId && { sessionId: args.ctx.sessionId }),
  ...(args.ctx?.runId && { runId: args.ctx.runId }),
  ...(args.toolCallId && { toolCallId: args.toolCallId }),
};

const hookResult = await hookRunner.runBeforeToolCall(
  {
    toolName,
    params: normalizedParams,
    ...(args.ctx?.runId && { runId: args.ctx.runId }),
    ...(args.toolCallId && { toolCallId: args.toolCallId }),
  },
  toolContext,
);
```

The `isPlainObject(params) ? params : {}` guard normalizes non-object params (e.g. null
or a primitive) to an empty object before passing them to hooks. Hooks always receive a
`Record<string, unknown>`, never a raw primitive.

### 3b. Event shape

```typescript
// src/plugins/types.ts:2092
type PluginHookBeforeToolCallEvent = {
  toolName: string;
  params: Record<string, unknown>; // all tool arguments (normalized)
  runId?: string; // stable agent run identifier
  toolCallId?: string; // provider-specific tool call ID (Anthropic, etc.)
};
```

`toolCallId` is the LLM provider's identifier for this specific tool invocation within a
response turn. It is `"http-" + Date.now()` for HTTP-initiated calls and absent for
node-initiated calls.

### 3c. Context shape

```typescript
// src/plugins/types.ts:2079
type PluginHookToolContext = {
  agentId?: string;
  sessionKey?: string;
  sessionId?: string; // ephemeral — regenerated on /new and /reset
  runId?: string;
  toolName: string;
  toolCallId?: string;
};
```

### 3d. Result shape

```typescript
// src/plugins/types.ts:2112
type PluginHookBeforeToolCallResult = {
  params?: Record<string, unknown>; // param rewrite — passed to tool instead of originals
  block?: boolean; // hard block, returns error to LLM
  blockReason?: string;
  requireApproval?: {
    title: string;
    description: string;
    severity?: "info" | "warning" | "critical";
    timeoutMs?: number; // default: 120_000ms
    timeoutBehavior?: "allow" | "deny"; // what happens if no decision arrives
    pluginId?: string; // set by runner — do NOT set in plugin code
    onResolution?: (decision: PluginApprovalResolution) => Promise<void> | void;
  };
};

type PluginApprovalResolution = "allow-once" | "allow-always" | "deny" | "timeout" | "cancelled";
```

### 3e. Merge logic in detail

The merge function at `hooks.ts:701` has the most nuanced logic of any hook in the
system:

```typescript
// src/plugins/hooks.ts:701
mergeResults: (acc, next, reg) => {
  // 1. If already blocked, short-circuit — ignore next handler's result entirely
  if (acc?.block === true) {
    return acc;
  }

  // 2. Freeze params if a different plugin already owns the approval
  const approvalPluginId = acc?.requireApproval?.pluginId;
  const freezeParamsForDifferentPlugin =
    Boolean(approvalPluginId) && approvalPluginId !== reg.pluginId;

  return {
    params: freezeParamsForDifferentPlugin
      ? acc?.params                           // frozen: ignore this plugin's param rewrite
      : lastDefined(acc?.params, next.params), // last plugin's rewrite wins

    block: stickyTrue(acc?.block, next.block),
    blockReason: lastDefined(acc?.blockReason, next.blockReason),

    // 3. First requireApproval wins — subsequent plugins cannot add a second gate
    requireApproval:
      acc?.requireApproval ??
      (next.requireApproval
        ? { ...next.requireApproval, pluginId: reg.pluginId }  // stamp pluginId
        : undefined),
  };
},
shouldStop: (result) => result.block === true,
```

Four behaviors to internalize:

1. **`block` is sticky and short-circuits.** Once any plugin blocks, `shouldStop`
   returns true and the chain terminates. No lower-priority handlers run.

2. **`requireApproval` is first-set-wins.** `acc?.requireApproval ?? next.requireApproval`
   means once a plugin claims the approval, lower-priority plugins cannot add a second
   gate. This is the primary limitation in the current design (see Session 7).

3. **`pluginId` is stamped by the runner, not the plugin.** `reg.pluginId` (from the
   `PluginHookRegistration`) overrides any `pluginId` the plugin might have tried to
   set. Plugins cannot impersonate other plugins in approval records.

4. **Params are frozen after an approval is claimed by another plugin.** If Plugin A
   (`priority: 100`) sets `requireApproval`, Plugin B (`priority: 50`) cannot rewrite
   `params` — the `freezeParamsForDifferentPlugin` guard prevents it. This ensures the
   approved params match what was presented in the approval dialog.

**Set a breakpoint at `hooks.ts:701`** (the `mergeResults` function body) to observe each
merge step when multiple plugins are registered.

---

## 4. The `requireApproval` flow

When a handler returns `requireApproval`, the wrapper in `pi-tools.before-tool-call.ts`
enters a two-phase gateway RPC flow:

### 4a. Phase 1 — Request approval (line 227)

```typescript
// src/agents/pi-tools.before-tool-call.ts:227
const requestResult = await callGatewayTool<{
  id?: string;
  status?: string;
  decision?: string | null;
}>(
  "plugin.approval.request",
  { timeoutMs: (approval.timeoutMs ?? 120_000) + 10_000 }, // +10s buffer for cleanup
  {
    pluginId: approval.pluginId,
    title: approval.title,
    description: approval.description,
    severity: approval.severity,
    toolName,
    toolCallId: args.toolCallId,
    agentId: args.ctx?.agentId,
    sessionKey: args.ctx?.sessionKey,
    timeoutMs: approval.timeoutMs ?? 120_000,
    twoPhase: true,
  },
);
const id = requestResult?.id;
```

`plugin.approval.request` is a gateway RPC method that:

1. Creates an in-flight approval record with the UUID `id`
2. Broadcasts the approval request to all connected clients (web UI, Telegram, Discord)
3. Returns the UUID immediately (two-phase: does not wait for a decision)

If `requestResult.decision` is present in the immediate response, an immediate decision
was made (e.g. an existing `allow-always` permission matched). A `null` decision means
the approval route is unavailable — the call is treated as blocked.

### 4b. Phase 2 — Wait for decision (line 275)

```typescript
// src/agents/pi-tools.before-tool-call.ts:275
const waitPromise = callGatewayTool<{ id?: string; decision?: string | null }>(
  "plugin.approval.waitDecision",
  { timeoutMs: (approval.timeoutMs ?? 120_000) + 10_000 },
  { id },
);

// Race the wait against the agent run's AbortSignal
let waitResult;
if (args.signal) {
  let onAbort;
  const abortPromise = new Promise<never>((_, reject) => {
    if (args.signal!.aborted) {
      reject(new Error("aborted"));
      return;
    }
    onAbort = () => reject(new Error("aborted"));
    args.signal!.addEventListener("abort", onAbort);
  });
  try {
    waitResult = await Promise.race([waitPromise, abortPromise]);
  } finally {
    if (onAbort) args.signal!.removeEventListener("abort", onAbort);
  }
} else {
  waitResult = await waitPromise;
}
```

The `Promise.race` with `AbortSignal` is the key safety feature here. If the agent run
is cancelled (user sends a new message, `/stop` command, timeout), the approval wait is
interrupted rather than holding the run open for the full `timeoutMs`. Without this race,
a stale approval request could keep a dead run's resources alive for up to 2 minutes.

### 4c. Resolution handling (line ~330)

```typescript
const decision = waitResult?.decision;
switch (decision) {
  case "allow-once":
  case "allow-always":
    safeOnResolution(PluginApprovalResolutions.ALLOW_ONCE or ALLOW_ALWAYS);
    return { blocked: false, params: hookResult.params ?? args.params };

  case "deny":
    safeOnResolution(PluginApprovalResolutions.DENY);
    return { blocked: true, reason: approval.description || "Plugin approval denied" };

  case null:
  case undefined:
    // timeout or no decision
    if (approval.timeoutBehavior === "allow") {
      safeOnResolution(PluginApprovalResolutions.TIMEOUT);
      return { blocked: false, params: hookResult.params ?? args.params };
    }
    safeOnResolution(PluginApprovalResolutions.TIMEOUT);
    return { blocked: true, reason: "Plugin approval timed out" };
}
```

`safeOnResolution` calls the plugin's `onResolution` callback in a fire-and-forget
manner — it is not awaited before the tool proceed/block decision is made. If
`onResolution` throws or rejects, the error is logged via `log.warn` and silently dropped.

**`timeoutBehavior: "allow"`** is the footgun: if the user is unreachable for 2 minutes,
the tool proceeds automatically. Always prefer `"deny"` (the implicit default) for
security gates.

---

## 5. `after_tool_call`

**Hook name:** `after_tool_call`
**Execution model:** Void
**Call site:** Tool execution completion in the agent runner

### 5a. Event shape

```typescript
// src/plugins/types.ts:2133
type PluginHookAfterToolCallEvent = {
  toolName: string;
  params: Record<string, unknown>; // as actually called (after before_tool_call rewrites)
  runId?: string;
  toolCallId?: string;
  result?: unknown; // tool output (undefined if error)
  error?: string; // error message if tool threw
  durationMs?: number; // tool execution time
};
```

`params` here reflects the **post-rewrite** params (after `before_tool_call` modifications).
If you need the original params for comparison, you must capture them in your
`before_tool_call` handler and correlate by `toolCallId`.

`result` is the raw tool output. For file-reading tools it may contain large amounts of
text; for exec tools it contains the command output. This hook is read-only — to modify
what gets persisted, use `tool_result_persist`.

---

## 6. `tool_result_persist`

**Hook name:** `tool_result_persist`
**Execution model:** Sync (inline sequential loop)
**Call site:** [src/agents/session-tool-result-guard-wrapper.ts:47](../../../../../src/agents/session-tool-result-guard-wrapper.ts)

### 6a. The `guardSessionManager` wiring

`tool_result_persist` is not called through the standard hook runner from a top-level
pipeline function. It is wired into the `SessionManager`'s `transformToolResultForPersistence`
callback via `guardSessionManager`:

```typescript
// src/agents/session-tool-result-guard-wrapper.ts:44
const transform = hookRunner?.hasHooks("tool_result_persist")
  ? (message, meta) => {
      const out = hookRunner.runToolResultPersist(
        {
          toolName: meta.toolName,
          toolCallId: meta.toolCallId,
          message, // raw AgentMessage about to be written to JSONL
          isSynthetic: meta.isSynthetic,
        },
        {
          agentId: opts?.agentId,
          sessionKey: opts?.sessionKey,
          toolName: meta.toolName,
          toolCallId: meta.toolCallId,
        },
      );
      return out?.message ?? message; // return modified or original
    }
  : undefined;
```

The `SessionManager` calls this transform callback **synchronously** during the JSONL
append operation. The entire transcript write path is synchronous — async transforms would
require restructuring the session storage layer.

### 6b. Event shape

```typescript
// src/plugins/types.ts:2153
type PluginHookToolResultPersistEvent = {
  toolName?: string;
  toolCallId?: string;
  message: AgentMessage; // the toolResult message object about to be written to JSONL
  isSynthetic?: boolean; // true if synthesized by a guard/repair step (not a real tool result)
};
```

### 6c. Context shape

```typescript
// src/plugins/types.ts:2146
type PluginHookToolResultPersistContext = {
  agentId?: string;
  sessionKey?: string;
  toolName?: string;
  toolCallId?: string;
};
```

### 6d. Result shape

```typescript
// src/plugins/types.ts:2165
type PluginHookToolResultPersistResult = {
  message?: AgentMessage; // replacement message; if omitted, original is used
};
```

### 6e. Chain threading

Unlike `before_tool_call`'s merge function, `tool_result_persist` threads the message
through handlers sequentially:

```typescript
// src/plugins/hooks.ts:757
let current = event.message;

for (const hook of hooks) {
  const out = (hook.handler as any)({ ...event, message: current }, ctx);
  // ... async guard ...
  const next = (out as PluginHookToolResultPersistResult | undefined)?.message;
  if (next) current = next;
}

return { message: current };
```

Each handler receives the message as it was modified by the previous handler. If Plugin A
(priority 100) strips a field, Plugin B (priority 50) sees the stripped version. This
allows a chain of transforms: normalize → redact → truncate.

**Returning `undefined` (or `{}`) from a handler** passes the current message unchanged
to the next handler. Only return a `message` when you want to replace the content.

### 6f. The async guard

```typescript
// src/plugins/hooks.ts:769
if (out && typeof (out as any).then === "function") {
  logger?.warn?.(
    `[hooks] tool_result_persist handler from ${hook.pluginId} returned a Promise; ` +
      `this hook is synchronous and the result was ignored.`,
  );
  continue; // skip this handler's result, continue with current message
}
```

An async handler does NOT block the JSONL write — it simply has its result dropped. The
message is written to disk as-is, and the async operation completes later with no effect.
This is a **silent correctness failure**: the handler appeared to run but its transform
was ignored.

**Set a breakpoint at `hooks.ts:769`** to catch async handlers during development.

---

## 7. `before_message_write`

**Hook name:** `before_message_write`
**Execution model:** Sync (inline sequential loop)
**Call site:** [src/agents/session-tool-result-guard-wrapper.ts:36](../../../../../src/agents/session-tool-result-guard-wrapper.ts)

### 7a. What it does

`before_message_write` fires for **every message** (any role: user, assistant, tool,
system) about to be appended to the session JSONL. It can:

- **Block** the message — it is never written to disk (disappears from history)
- **Replace** the message — a modified version is written instead

This is broader than `tool_result_persist` (which only covers tool results). Use
`before_message_write` when you need to intercept user messages, assistant responses, or
any other message type.

### 7b. Wiring in `guardSessionManager`

```typescript
// src/agents/session-tool-result-guard-wrapper.ts:34
const beforeMessageWrite = hookRunner?.hasHooks("before_message_write")
  ? (event) => {
      return hookRunner.runBeforeMessageWrite(event, {
        agentId: opts?.agentId,
        sessionKey: opts?.sessionKey,
      });
    }
  : undefined;
```

The `SessionManager` calls this callback synchronously as part of its message append
path. The same async guard that applies to `tool_result_persist` applies here.

### 7c. Event shape

```typescript
// src/plugins/types.ts:2170
type PluginHookBeforeMessageWriteEvent = {
  message: AgentMessage; // any message type: user, assistant, tool, system
  sessionKey?: string;
  agentId?: string;
};
```

### 7d. Result shape

```typescript
// src/plugins/types.ts:2176
type PluginHookBeforeMessageWriteResult = {
  block?: boolean; // if true, message is NOT written to JSONL
  message?: AgentMessage; // replacement message (if set, written instead of original)
};
```

### 7e. Block short-circuits the chain

```typescript
// src/plugins/hooks.ts:848
const result = out as PluginHookBeforeMessageWriteResult | undefined;

// If any handler blocks, return immediately — do not continue chain
if (result?.block) {
  return { block: true };
}

// If handler provided a modified message, thread it forward
if (result?.message) {
  current = result.message;
}
```

Once any handler returns `{ block: true }`, the loop exits and `{ block: true }` is
returned to the `SessionManager`, which skips the write. Lower-priority handlers never
run after a block — there is no way to unblock a message once blocked.

**Security implication:** a malicious plugin at any priority can silently suppress
messages from the session transcript. An audit log plugin that registers `before_message_write`
at `priority: -1` (after all other plugins) will never see the suppressed messages. There
is no out-of-band notification that a message was blocked.

---

## 8. Comparison: `tool_result_persist` vs `before_message_write`

|                    | `tool_result_persist`               | `before_message_write`     |
| ------------------ | ----------------------------------- | -------------------------- |
| Scope              | Tool result messages only           | All messages (any role)    |
| Can block?         | No (no block field)                 | Yes (`block: true`)        |
| Can rewrite?       | Yes (`message` field)               | Yes (`message` field)      |
| Chain behavior     | Thread through all handlers         | Block short-circuits chain |
| `isSynthetic` flag | Yes (guard/repair results)          | No                         |
| Call site          | `transformToolResultForPersistence` | `beforeMessageWriteHook`   |

Use `tool_result_persist` when you want to normalize or redact tool output without
blocking. Use `before_message_write` when you need to suppress messages entirely or
intercept non-tool messages.

---

## 9. Exercises

### Exercise 1 — Block a dangerous tool via `before_tool_call`

1. Register `before_tool_call` with a handler that blocks `exec` calls containing
   `rm -rf`:
   ```typescript
   handler: async (event) => {
     if (event.toolName === "exec") {
       const cmd = String(event.params.command ?? "");
       if (cmd.includes("rm -rf")) {
         return { block: true, blockReason: "Destructive command blocked by policy" };
       }
     }
   };
   ```
2. Ask the agent to run `rm -rf /tmp/test`.
3. Verify the agent receives a tool-blocked error and the command is never executed.
4. Set a breakpoint at `pi-tools.before-tool-call.ts:204` (`if (hookResult?.block)`) to
   observe the blocked result.

### Exercise 2 — Trace the `requireApproval` two-phase RPC flow

1. Register `before_tool_call` returning `requireApproval` for `exec` tool calls.
2. Set breakpoints at:
   - `pi-tools.before-tool-call.ts:227` (Phase 1 — `plugin.approval.request`)
   - `pi-tools.before-tool-call.ts:275` (Phase 2 — `plugin.approval.waitDecision`)
3. Ask the agent to run `ls /tmp`.
4. Observe Phase 1 firing — inspect `requestResult.id` (the approval UUID).
5. The agent suspends — resolve the approval from the web UI or via the gateway API.
6. Observe Phase 2 completing — inspect `waitResult.decision`.

### Exercise 3 — Scrub sensitive data via `tool_result_persist`

1. Register `tool_result_persist` with a handler that redacts API keys
   (patterns like `sk-...` or `eyJ...`):
   ```typescript
   handler: (event) => {
     const msg = event.message;
     // Assume msg.content is a string or array — adapt to AgentMessage shape
     const scrubbed = JSON.parse(
       JSON.stringify(msg).replace(/sk-[a-zA-Z0-9]{20,}/g, "[REDACTED_API_KEY]"),
     );
     return { message: scrubbed };
   };
   ```
2. Execute a tool that returns content containing an API key pattern.
3. Verify the JSONL transcript contains `[REDACTED_API_KEY]` instead of the raw key.
4. Verify `after_tool_call` still receives the original (non-scrubbed) result —
   confirming the redaction only affects persistence.

### Exercise 4 — Suppress user messages via `before_message_write`

1. Register `before_message_write` with a handler that blocks messages from role "user"
   that contain the word "debug":
   ```typescript
   handler: (event) => {
     const msg = event.message as any;
     if (msg.role === "user" && String(msg.content).includes("debug")) {
       return { block: true };
     }
   };
   ```
2. Send "debug this for me".
3. Check the session JSONL — the user message should not appear.
4. Observe that the agent received no message and produced no response (no JSONL entry
   for either side).

---

## See Also

- [Session 1 — Hook System Architecture](session1-hook-system-architecture.md) — Sync execution model and async guard
- [Session 7 — Security Analysis](session7-security-analysis.md) — `before_tool_call` trust model and `requireApproval` footguns
- [Gateway Session 6 — Tool Dispatch](../../gateway/security/study/sessions/session6-tool-dispatch.md) — the three tool dispatch surfaces
- [plugin-security-hooks.md](../plugin-security-hooks.md) — design analysis of `before_tool_call` + `requireApproval`
