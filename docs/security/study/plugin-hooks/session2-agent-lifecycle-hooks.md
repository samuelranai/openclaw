---
title: "Session 2 — Agent Lifecycle Hooks"
summary: "Source walkthrough of the six agent lifecycle hooks: before_model_resolve, before_prompt_build, before_agent_start (legacy), llm_input, llm_output, and agent_end — covering their positions in the EmbeddedRunAttempt pipeline, event and context shapes, and merge semantics"
read_when:
  - Building a plugin that overrides the model, injects system prompt context, or observes LLM I/O
  - Investigating prompt injection risk via before_prompt_build or before_agent_start
  - Understanding when session messages are available in each hook phase
---

# Session 2 — Agent Lifecycle Hooks

This session walks through the six hooks that fire during a single agent run, from
model selection through LLM call to run completion. All six are invoked from the
embedded agent runner — the same code path used for every agent interaction
regardless of channel.

**Prerequisite:** Session 1 (execution models, priority, merge strategies).

---

## 1. What this session covers

| File                                                          | Role                                                                                 |
| ------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `src/agents/pi-embedded-runner/run/setup.ts`                  | `resolveHookModelSelection` — `before_model_resolve` + legacy `before_agent_start`   |
| `src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts` | `resolvePromptBuildHookResult` — `before_prompt_build` + legacy `before_agent_start` |
| `src/agents/pi-embedded-runner/run/attempt.ts`                | `llm_input`, `llm_output`, `agent_end` call sites                                    |
| `src/plugins/types.ts`                                        | All event, context, and result type definitions                                      |
| `src/plugins/hooks.ts`                                        | Runner function implementations                                                      |

---

## 2. Where these hooks fit in the agent pipeline

```
User message arrives
  │
  └─► EmbeddedRunAttempt begins
        │
        ├─ [Phase A] Model resolution
        │    └─► before_model_resolve  (Modifying)
        │    └─► before_agent_start    (Modifying, legacy — model fields only)
        │
        ├─ [Phase B] Session load + prompt assembly
        │    └─► before_prompt_build   (Modifying)
        │    └─► before_agent_start    (Modifying, legacy — prompt fields only)
        │
        ├─ [Phase C] LLM call
        │    └─► llm_input             (Void, fire-and-forget, before call)
        │    ← LLM responds →
        │    └─► llm_output            (Void, fire-and-forget, after call)
        │
        └─ [Phase D] Run teardown
             └─► agent_end             (Void, fire-and-forget)
```

Note that `before_agent_start` appears **twice** — once in Phase A (for model override
fields) and once in Phase B (for prompt fields). Session 1's discussion of merge
strategies explains why this does not double-apply fields (the prompt fields are stripped
after Phase A via `stripPromptMutationFieldsFromLegacyHookResult`).

---

## 3. `before_model_resolve`

**Hook name:** `before_model_resolve`
**Execution model:** Modifying
**Call site:** [src/agents/pi-embedded-runner/run/setup.ts:56](../../../../../src/agents/pi-embedded-runner/run/setup.ts)

### 3a. When it fires

Phase A — the **earliest point** in an agent run, before `resolveModel()` selects the
final API model object. No session messages are available yet.

### 3b. Call site walkthrough

```typescript
// src/agents/pi-embedded-runner/run/setup.ts:54
if (hookRunner?.hasHooks("before_model_resolve")) {
  try {
    modelResolveOverride = await hookRunner.runBeforeModelResolve(
      { prompt: params.prompt }, // event
      params.hookContext, // ctx (runId, agentId, sessionKey, ...)
    );
  } catch (hookErr) {
    log.warn(`before_model_resolve hook failed: ${String(hookErr)}`);
  }
}
```

The outer `try/catch` here is a belt-and-suspenders guard on top of the runner's internal
`catchErrors: true`. Hook failures at this level produce a `warn` (not `error`) and the
agent proceeds with the configured model.

After this block, the legacy `before_agent_start` call (line 65) can also contribute
model override fields — but `before_model_resolve` takes precedence because the override
values are merged with `firstDefined` (highest-priority-plugin-wins):

```typescript
// src/agents/pi-embedded-runner/run/setup.ts:71
modelResolveOverride = {
  providerOverride:
    modelResolveOverride?.providerOverride ?? legacyBeforeAgentStartResult?.providerOverride,
  modelOverride: modelResolveOverride?.modelOverride ?? legacyBeforeAgentStartResult?.modelOverride,
};
```

### 3c. Event shape

```typescript
// src/plugins/types.ts:1831
type PluginHookBeforeModelResolveEvent = {
  prompt: string; // user message for this run — no session history yet
};
```

### 3d. Context shape

```typescript
// src/plugins/types.ts:1816
type PluginHookAgentContext = {
  runId?: string; // stable identifier for this agent invocation
  agentId?: string;
  sessionKey?: string;
  sessionId?: string;
  workspaceDir?: string;
  messageProvider?: string; // channel the trigger arrived on
  trigger?: string; // "user" | "heartbeat" | "cron" | "memory"
  channelId?: string;
};
```

### 3e. Result shape and merge

```typescript
// src/plugins/types.ts:1836
type PluginHookBeforeModelResolveResult = {
  modelOverride?: string; // e.g. "llama3.3:8b"
  providerOverride?: string; // e.g. "ollama"
};
```

Merge strategy from `hooks.ts:185`:

```typescript
const mergeBeforeModelResolve = (acc, next) => ({
  modelOverride: firstDefined(acc?.modelOverride, next.modelOverride),
  providerOverride: firstDefined(acc?.providerOverride, next.providerOverride),
});
```

**`firstDefined`**: the highest-priority plugin to set a field wins. A plugin at
`priority: 100` that returns `{ modelOverride: "claude-opus-4-6" }` cannot be
overridden by a plugin at `priority: 50` that returns `{ modelOverride: "gpt-5.4" }`.

**Set a breakpoint at `setup.ts:82`** (the `if (modelResolveOverride?.providerOverride)` block)
to observe the final resolved provider and model IDs after hooks run.

---

## 4. `before_prompt_build`

**Hook name:** `before_prompt_build`
**Execution model:** Modifying
**Call site:** [src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts:37](../../../../../src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts)

### 4a. When it fires

Phase B — after session messages are loaded from the JSONL transcript, but **before** the
system prompt and user context are assembled into the final LLM payload. This is the
primary injection point for dynamic context (RAG results, user preferences, per-run
instructions).

### 4b. Call site walkthrough

```typescript
// src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts:35
const promptBuildResult = params.hookRunner?.hasHooks("before_prompt_build")
  ? await params.hookRunner
      .runBeforePromptBuild(
        { prompt: params.prompt, messages: params.messages }, // event
        params.hookCtx, // ctx
      )
      .catch((hookErr: unknown) => {
        log.warn(`before_prompt_build hook failed: ${String(hookErr)}`);
        return undefined;
      })
  : undefined;
```

After this, the legacy `before_agent_start` call runs for its prompt fields (line 51),
and the two results are merged:

```typescript
// src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts:67
return {
  systemPrompt: promptBuildResult?.systemPrompt ?? legacyResult?.systemPrompt,
  prependContext: joinPresentTextSegments([
    promptBuildResult?.prependContext,
    legacyResult?.prependContext,
  ]),
  prependSystemContext: joinPresentTextSegments([...]),
  appendSystemContext:  joinPresentTextSegments([...]),
};
```

The `??` for `systemPrompt` means the newer `before_prompt_build` hook's value takes
precedence over the legacy `before_agent_start` value. For the context fields, both are
concatenated.

### 4c. Event shape

```typescript
// src/plugins/types.ts:1844
type PluginHookBeforePromptBuildEvent = {
  prompt: string; // current user turn
  messages: unknown[]; // full session history loaded from JSONL
};
```

This is the first hook where `messages` is available. A plugin can inspect prior turns,
count tool calls, or analyze conversation state before deciding what context to inject.

### 4d. Result shape and merge

```typescript
// src/plugins/types.ts:1850
type PluginHookBeforePromptBuildResult = {
  systemPrompt?: string; // replaces the entire system prompt
  prependContext?: string; // prepended to the user turn body
  prependSystemContext?: string; // prepended to system prompt (cacheable)
  appendSystemContext?: string; // appended to system prompt (cacheable)
};
```

Four fields, two distinct merge strategies:

| Field                  | Strategy      | Winner                                      |
| ---------------------- | ------------- | ------------------------------------------- |
| `systemPrompt`         | `lastDefined` | Lowest-priority plugin that sets it         |
| `prependContext`       | concatenate   | All plugins contribute, high-priority first |
| `prependSystemContext` | concatenate   | All plugins contribute, high-priority first |
| `appendSystemContext`  | concatenate   | All plugins contribute, high-priority first |

The `prependSystemContext` / `appendSystemContext` fields exist specifically for **static
content** that does not change per turn. Placing static plugin guidance here instead of in
`prependContext` allows the LLM provider's prompt-caching mechanism to cache those tokens
— reducing per-turn cost. Use `prependContext` for dynamic content (RAG results, timestamps,
per-turn state).

**Security note:** This hook is in `PROMPT_INJECTION_HOOK_NAMES`. A plugin registered
here can silently inject any text into the system prompt or user turn. There is no
content inspection of what the plugin returns. See Session 7 for the full attack surface
analysis.

**Set a breakpoint at `attempt.prompt-helpers.ts:67`** (the `return` statement). Inspect
the returned object to see the assembled prompt injection fields that will be applied to
the next LLM call.

---

## 5. `before_agent_start` (legacy)

**Hook name:** `before_agent_start`
**Execution model:** Modifying
**Called from two places:**

- [src/agents/pi-embedded-runner/run/setup.ts:67](../../../../../src/agents/pi-embedded-runner/run/setup.ts) (Phase A — model fields)
- [src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts:52](../../../../../src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts) (Phase B — prompt fields)

### 5a. Why this hook exists

`before_agent_start` is a legacy compatibility hook that predates the Phase A / Phase B
split. It combines what are now `before_model_resolve` (model/provider override) and
`before_prompt_build` (prompt injection) into a single handler. New plugins should prefer
the two dedicated hooks.

### 5b. The two-call problem and `stripPromptMutationFieldsFromLegacyHookResult`

Because `before_agent_start` is called in Phase A (for model fields) and Phase B (for
prompt fields), the runner must ensure prompt-mutation fields are not applied twice.
Phase A calls `runBeforeAgentStart` at `setup.ts:67`, but then discards the prompt
fields:

```typescript
// src/agents/pi-embedded-runner/run/setup.ts:71
// Only use model override fields from the legacy result:
modelResolveOverride = {
  providerOverride:
    modelResolveOverride?.providerOverride ?? legacyBeforeAgentStartResult?.providerOverride,
  modelOverride: modelResolveOverride?.modelOverride ?? legacyBeforeAgentStartResult?.modelOverride,
};
// legacyBeforeAgentStartResult still holds systemPrompt, prependContext, etc.
// but those are not used here.
```

Phase B then receives `legacyBeforeAgentStartResult` from Phase A as a parameter and
uses its prompt fields in the final merge. The pattern ensures each field class is applied
exactly once.

### 5c. Event shape

```typescript
// src/plugins/types.ts:1882
type PluginHookBeforeAgentStartEvent = {
  prompt: string;
  messages?: unknown[]; // optional — not present in the Phase A call
};
```

The `messages` field is `optional` because Phase A runs before session messages are
loaded. Your handler must guard against `messages` being undefined if it registers
`before_agent_start` and inspects history.

### 5d. Result shape

```typescript
// src/plugins/types.ts:1888
type PluginHookBeforeAgentStartResult = PluginHookBeforePromptBuildResult &
  PluginHookBeforeModelResolveResult;
// = { systemPrompt?, prependContext?, prependSystemContext?, appendSystemContext?,
//     modelOverride?, providerOverride? }
```

All six fields in one return object. The runner merges them using the combined strategy
from both Phase A and Phase B merge functions.

---

## 6. `llm_input`

**Hook name:** `llm_input`
**Execution model:** Void
**Call site:** [src/agents/pi-embedded-runner/run/attempt.ts:1454](../../../../../src/agents/pi-embedded-runner/run/attempt.ts)

### 6a. When it fires

Phase C — immediately **before** the LLM API call, after system prompt assembly and image
processing. The LLM call proceeds without waiting for this hook (fire-and-forget via
`void ... .catch(...)`).

### 6b. Call site walkthrough

```typescript
// src/agents/pi-embedded-runner/run/attempt.ts:1452
if (hookRunner?.hasHooks("llm_input")) {
  hookRunner
    .runLlmInput(
      {
        runId: params.runId,
        sessionId: params.sessionId,
        provider: params.provider,
        model: params.modelId,
        systemPrompt: systemPromptText, // final assembled system prompt
        prompt: effectivePrompt, // effective user turn (may differ from raw prompt)
        historyMessages: activeSession.messages,
        imagesCount: imageResult.images.length,
      },
      {
        runId: params.runId,
        agentId: hookAgentId,
        sessionKey: params.sessionKey,
        sessionId: params.sessionId,
        workspaceDir: params.workspaceDir,
        messageProvider: params.messageProvider ?? undefined,
        trigger: params.trigger,
        channelId: params.messageChannel ?? params.messageProvider ?? undefined,
      },
    )
    .catch((err) => {
      log.warn(`llm_input hook failed: ${String(err)}`);
    });
}
```

### 6c. Event shape

```typescript
// src/plugins/types.ts:1912
type PluginHookLlmInputEvent = {
  runId: string;
  sessionId: string;
  provider: string; // e.g. "anthropic"
  model: string; // e.g. "claude-sonnet-4-6"
  systemPrompt?: string; // final assembled system prompt after before_prompt_build
  prompt: string; // effective user turn
  historyMessages: unknown[]; // session messages fed to LLM
  imagesCount: number; // images in this turn
};
```

This hook receives the **final assembled state** going to the LLM — `systemPrompt` here
is the result of all `before_prompt_build` injections. This is the best place to log
the exact payload for auditability or SIEM ingestion.

**What this hook cannot do:** it cannot modify any of these fields. The LLM call has
already been dispatched. Use `before_prompt_build` if you need to alter the input.

---

## 7. `llm_output`

**Hook name:** `llm_output`
**Execution model:** Void
**Call site:** [src/agents/pi-embedded-runner/run/attempt.ts:1748](../../../../../src/agents/pi-embedded-runner/run/attempt.ts)

### 7a. When it fires

Phase C — after the LLM response stream completes. Like `llm_input`, this is
fire-and-forget. The agent run (tool dispatching, response delivery) does not wait for
this hook.

### 7b. Call site walkthrough

```typescript
// src/agents/pi-embedded-runner/run/attempt.ts:1747
if (hookRunner?.hasHooks("llm_output")) {
  hookRunner
    .runLlmOutput(
      {
        runId: params.runId,
        sessionId: params.sessionId,
        provider: params.provider,
        model: params.modelId,
        assistantTexts, // all text blocks from this run
        lastAssistant, // last raw assistant message object
        usage: getUsageTotals(),
      },
      {
        /* same PluginHookAgentContext */
      },
    )
    .catch((err) => {
      log.warn(`llm_output hook failed: ${String(err)}`);
    });
}
```

### 7c. Event shape

```typescript
// src/plugins/types.ts:1924
type PluginHookLlmOutputEvent = {
  runId: string;
  sessionId: string;
  provider: string;
  model: string;
  assistantTexts: string[]; // one entry per text block in the response
  lastAssistant?: unknown; // raw assistant message (provider-specific format)
  usage?: {
    input?: number;
    output?: number;
    cacheRead?: number;
    cacheWrite?: number;
    total?: number;
  };
};
```

`assistantTexts` is an array because a single run can produce multiple text blocks
(for example when the agent runs multiple tool-use/response cycles or during compaction).

`usage` contains token counts including cache read/write hits — useful for building cost
attribution systems.

---

## 8. `agent_end`

**Hook name:** `agent_end`
**Execution model:** Void
**Call site:** [src/agents/pi-embedded-runner/run/attempt.ts:1688](../../../../../src/agents/pi-embedded-runner/run/attempt.ts)

### 8a. When it fires

Phase D — at the very end of the agent run, inside the `finally` block. Fires even
when the run was aborted, timed out, or failed. This makes it reliable for cleanup and
audit logging.

### 8b. Call site walkthrough

```typescript
// src/agents/pi-embedded-runner/run/attempt.ts:1687
if (hookRunner?.hasHooks("agent_end")) {
  hookRunner
    .runAgentEnd(
      {
        messages: messagesSnapshot, // final session state
        success: !aborted && !promptError,
        error: promptError ? describeUnknownError(promptError) : undefined,
        durationMs: Date.now() - promptStartedAt,
      },
      {
        /* PluginHookAgentContext */
      },
    )
    .catch((err) => {
      log.warn(`agent_end hook failed: ${err}`);
    });
}
```

`messagesSnapshot` is taken just before the `finally` block — it captures the full
session state at the end of the run including all tool results and assistant responses.

### 8c. Event shape

```typescript
// src/plugins/types.ts:1941
type PluginHookAgentEndEvent = {
  messages: unknown[]; // full session snapshot
  success: boolean; // false if aborted, timed out, or LLM error
  error?: string; // human-readable error if not success
  durationMs?: number; // wall-clock time for this agent run
};
```

**Key difference from `llm_output`:** `agent_end` includes **all messages** (user, tool,
assistant) whereas `llm_output` only includes the LLM response text blocks. If you need
the complete conversation transcript at run end, use `agent_end`. If you need token usage
details, use `llm_output`.

---

## 9. Full pipeline trace — putting the six hooks together

Here is the complete sequence for a single user message that triggers an agent run with
two tool calls:

```
1. message arrives
2. dispatch-from-config routes to EmbeddedRunAttempt

Phase A (setup.ts):
3. before_model_resolve fires — plugin can override model/provider
4. before_agent_start fires (legacy, Phase A variant) — model fields only

Phase B (attempt.prompt-helpers.ts):
5. before_prompt_build fires — plugin can inject systemPrompt, prependContext, etc.
6. before_agent_start fires (legacy, Phase B variant) — prompt fields only

Phase C (attempt.ts, first LLM call):
7. [system prompt assembled from hook results]
8. llm_input fires (fire-and-forget) — full LLM payload observable
9. LLM call executes
10. LLM returns (tool_use + text)
11. [tool calls processed — see Session 5 for before_tool_call]
12. LLM call executes (second turn with tool results)
13. LLM returns (final text)
14. llm_output fires (fire-and-forget) — response + usage observable

Phase D:
15. agent_end fires (fire-and-forget) — final session state, success/error, duration
```

**Set a breakpoint at `setup.ts:54`** to start a live trace from the beginning of Phase A.

---

## 10. Exercises

### Exercise 1 — Trace model selection under a hook override

1. Install a test plugin that registers `before_model_resolve` returning
   `{ modelOverride: "gpt-5.4", providerOverride: "openai" }`.
2. Set a breakpoint at `setup.ts:82` (`if (modelResolveOverride?.providerOverride)`).
3. Send a message. The breakpoint hits — inspect `provider` and `modelId` before and
   after the override is applied.
4. Observe the `[hooks] provider overridden to openai` log line in the gateway output.

### Exercise 2 — Observe the two-phase `before_agent_start` dispatch

1. Install a plugin that registers `before_agent_start` with a handler that logs
   `event.messages?.length` on each call.
2. Set breakpoints at `setup.ts:67` and `attempt.prompt-helpers.ts:52`.
3. Send a message. Observe that `setup.ts:67` fires first with `messages: undefined`
   (Phase A — no session yet), then `attempt.prompt-helpers.ts:52` fires with the
   loaded `messages` array (Phase B).

### Exercise 3 — Compare `llm_input` systemPrompt vs configured base prompt

1. Register `llm_input` and log `event.systemPrompt`.
2. Also register `before_prompt_build` returning `{ prependSystemContext: "INJECTED_BY_PLUGIN\n" }`.
3. Send a message and inspect the `llm_input` event. Verify that `systemPrompt` begins
   with `INJECTED_BY_PLUGIN` — confirming that `before_prompt_build` injections are
   visible in `llm_input`.

### Exercise 4 — Measure agent run duration via `agent_end`

1. Register `agent_end` and log `event.durationMs` and `event.success`.
2. Send a message that triggers at least one tool call.
3. Compare the logged `durationMs` with the wall-clock time you observe in the UI.
4. Force an error (e.g. set an invalid model ID) and observe `event.success = false` and
   `event.error` in the hook.

---

## See Also

- [Session 1 — Hook System Architecture](session1-hook-system-architecture.md) — execution models and merge strategies
- [Session 3 — Compaction and Reset Hooks](session3-compaction-reset-hooks.md)
- [Session 7 — Security Analysis](session7-security-analysis.md) — prompt injection risk via `before_prompt_build`
- [plugin-security-hooks.md](../plugin-security-hooks.md) — `PROMPT_INJECTION_HOOK_NAMES` discussion
