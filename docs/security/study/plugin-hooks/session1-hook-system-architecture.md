---
title: "Session 1 — Hook System Architecture"
summary: "Deep dive into the plugin hook runner machinery: PluginHookRegistration, the four execution models (Void, Modifying, Claiming, Sync), priority sorting, the hasHooks fast path, error handling, and the global singleton lifecycle"
read_when:
  - Starting the plugin hook study series
  - Understanding how hook handlers are registered, ordered, and invoked
  - Debugging unexpected hook behavior (wrong order, silent errors, async ignored)
---

# Session 1 — Hook System Architecture

This session covers the machinery underneath all 26 plugin hooks: how handlers are
stored, how the runner decides execution order, and what each execution model
guarantees. Every later session builds on the concepts introduced here.

**Prerequisite:** Read [plugin-architecture-today.md](../plugin-architecture-today.md)
for the plugin manifest format and `PluginRuntime` SDK surface overview.

---

## 1. What this session covers

| File                                | Lines | Role                                                                       |
| ----------------------------------- | ----- | -------------------------------------------------------------------------- |
| `src/plugins/types.ts`              | ~2411 | All hook names, event/context/result types, `PluginHookRegistration`       |
| `src/plugins/hooks.ts`              | ~1040 | `createHookRunner` — all four execution models, 26 public runner functions |
| `src/plugins/hook-runner-global.ts` | ~105  | Global singleton: `initializeGlobalHookRunner`, `getGlobalHookRunner`      |

---

## 2. The 26 hook names

The canonical list lives as a TypeScript union in
[src/plugins/types.ts:1736](../../../../../src/plugins/types.ts):

```typescript
// src/plugins/types.ts:1736
export type PluginHookName =
  | "before_model_resolve"
  | "before_prompt_build"
  | "before_agent_start"
  | "llm_input"
  | "llm_output"
  | "agent_end"
  | "before_compaction"
  | "after_compaction"
  | "before_reset"
  | "inbound_claim"
  | "message_received"
  | "message_sending"
  | "message_sent"
  | "before_tool_call"
  | "after_tool_call"
  | "tool_result_persist"
  | "before_message_write"
  | "session_start"
  | "session_end"
  | "subagent_spawning"
  | "subagent_delivery_target"
  | "subagent_spawned"
  | "subagent_ended"
  | "gateway_start"
  | "gateway_stop"
  | "before_dispatch";
```

The matching `PLUGIN_HOOK_NAMES` constant at line 1764 is a `readonly` array used to
validate hook names at runtime via `isPluginHookName`. A compile-time exhaustiveness
check (`MissingPluginHookNames`, line 1793) ensures the array always matches the union —
the file will not compile if you add a hook name to the union without adding it to the
array.

Two hooks have a special designation — `PROMPT_INJECTION_HOOK_NAMES` (line 1803):

```typescript
// src/plugins/types.ts:1803
export const PROMPT_INJECTION_HOOK_NAMES = [
  "before_prompt_build",
  "before_agent_start",
] as const satisfies readonly PluginHookName[];
```

These are the only hooks that can inject text into the system prompt or user context.
The designation is used by security analysis tooling and documentation; at runtime, the
hooks themselves behave identically to other Modifying hooks.

---

## 3. `PluginHookRegistration` — the stored record

When a plugin calls `runtime.hooks.register(hookName, { priority, handler })`, the
registration is stored as a `PluginHookRegistration<K>` struct
([src/plugins/types.ts:2404](../../../../../src/plugins/types.ts)):

```typescript
// src/plugins/types.ts:2404
export type PluginHookRegistration<K extends PluginHookName = PluginHookName> = {
  pluginId: string; // set by the registration infra, not by the plugin
  hookName: K;
  handler: PluginHookHandlerMap[K];
  priority?: number; // higher = runs first; undefined treated as 0
  source: string; // file path of the plugin entry that registered this hook
};
```

Key points:

- **`pluginId` is set by the framework**, not the plugin. A plugin cannot claim another
  plugin's ID. This matters most for `before_tool_call`, where the `pluginId` is stamped
  onto `requireApproval` records so the gateway UI can attribute approvals correctly.
- **`priority` defaults to 0** when absent. The runner sorts by `(b.priority ?? 0) - (a.priority ?? 0)`,
  so higher numbers run first.
- **`source`** is the resolved module path. This is used in error messages and debug logs
  to identify which file registered a failing handler.

All registrations for all hooks from all plugins are stored in a single flat array —
`registry.typedHooks` — on the `PluginRegistry`. The registry itself is created during
plugin loading and passed to `createHookRunner`.

---

## 4. The global singleton

Plugins run in-process, but the hook runner must be accessible from anywhere in the
codebase — agent loops, outbound delivery, session management — without threading the
runner through every call site. This is handled by a global singleton in
[src/plugins/hook-runner-global.ts](../../../../../src/plugins/hook-runner-global.ts).

### 4a. Initialization (line 32)

```typescript
// src/plugins/hook-runner-global.ts:32
export function initializeGlobalHookRunner(registry: PluginRegistry): void {
  const state = getState();
  state.registry = registry;
  state.hookRunner = createHookRunner(registry, {
    logger: {
      debug: (msg) => log.debug(msg),
      warn: (msg) => log.warn(msg),
      error: (msg) => log.error(msg),
    },
    catchErrors: true, // <-- hook errors never propagate to callers
  });

  const hookCount = registry.hooks.length;
  if (hookCount > 0) {
    log.info(`hook runner initialized with ${hookCount} registered hooks`);
  }
}
```

`initializeGlobalHookRunner` is called exactly once during gateway startup, after plugins
are loaded. `catchErrors: true` means that any exception thrown inside a hook handler is
caught, logged at `error` level, and silently discarded — the pipeline continues as if the
handler had returned `undefined`. This is a deliberate resilience choice: a broken plugin
should not bring down the gateway.

### 4b. Access (line 55)

```typescript
// src/plugins/hook-runner-global.ts:55
export function getGlobalHookRunner(): HookRunner | null {
  return getState().hookRunner;
}
```

Returns `null` if no plugins are loaded. Every call site guards with `hookRunner?.hasHooks(...)` before calling any runner method — this is the canonical pattern throughout the codebase.

### 4c. The `hasHooks` fast path (hooks.ts:987)

```typescript
// src/plugins/hooks.ts:987
function hasHooks(hookName: PluginHookName): boolean {
  return registry.typedHooks.some((h) => h.hookName === hookName);
}
```

`hasHooks` is a linear scan over `typedHooks`. It is intentionally not cached — the
plugin registry can change at runtime (hot-reload, plugin enable/disable). The typical
usage pattern is:

```typescript
if (hookRunner?.hasHooks("before_tool_call")) {
  const result = await hookRunner.runBeforeToolCall(event, ctx);
}
```

This two-step idiom avoids creating event objects and allocating context structs when no
handlers are registered — important for hooks on hot paths like `before_message_write`
and `tool_result_persist`.

**Set a breakpoint at line 988** (`return registry.typedHooks.some(...)`) to observe
the `typedHooks` array. When no plugins are loaded, this will be an empty array and
`some` returns `false` immediately.

---

## 5. `createHookRunner` — the factory

[src/plugins/hooks.ts:176](../../../../../src/plugins/hooks.ts) exports one function:

```typescript
// src/plugins/hooks.ts:176
export function createHookRunner(registry: PluginRegistry, options: HookRunnerOptions = {}) {
  const catchErrors = options.catchErrors ?? true;
  // ... defines runVoidHook, runModifyingHook, runClaimingHook, sync loops ...
  // ... defines all 26 public run* functions ...
  return {
    runBeforeModelResolve,
    runBeforePromptBuild,
    // ... all 26 ...
    hasHooks,
    getHookCount,
  };
}
```

All runner functions are closures over the same `registry` reference. If the registry is
mutated (plugin added/removed), the next call to any runner function picks up the change
without recreation.

### Priority sorting (line 156)

```typescript
// src/plugins/hooks.ts:156
function getHooksForName<K extends PluginHookName>(
  registry: PluginRegistry,
  hookName: K,
): PluginHookRegistration<K>[] {
  return (registry.typedHooks as PluginHookRegistration<K>[])
    .filter((h) => h.hookName === hookName)
    .toSorted((a, b) => (b.priority ?? 0) - (a.priority ?? 0));
}
```

`.toSorted` creates a new sorted array on every call. It does not mutate the stored
`typedHooks` array. The sort is descending by priority (`b - a`), so a handler with
`priority: 100` runs before `priority: 50`, which runs before `priority: 0` (the default).

When two handlers have the same priority, their relative order is determined by their
position in `typedHooks`, which reflects plugin registration order — not a stable
guarantee.

---

## 6. The four execution models

### 6a. Void — parallel, no return value (line 264)

```typescript
// src/plugins/hooks.ts:264
async function runVoidHook<K extends PluginHookName>(
  hookName: K,
  event: ...,
  ctx: ...,
): Promise<void> {
  const hooks = getHooksForName(registry, hookName);
  if (hooks.length === 0) return;

  const promises = hooks.map(async (hook) => {
    try {
      await (hook.handler as ...)(event, ctx);
    } catch (err) {
      handleHookError({ hookName, pluginId: hook.pluginId, error: err });
    }
  });

  await Promise.all(promises);
}
```

All handlers are started simultaneously (`Promise.all`). Return values are discarded.
Used for: `llm_input`, `llm_output`, `agent_end`, `message_received`, `message_sent`,
`session_start`, `session_end`, `gateway_start`, `gateway_stop`, and most lifecycle hooks.

**What this means in practice:** if Plugin A (`priority: 100`) and Plugin B (`priority: 50`)
both register `llm_input`, both fire at the same time. Priority is meaningless for Void
hooks because all handlers run regardless — the sort order only affects which handler's
error appears first in logs.

### 6b. Modifying — sequential, results merged (line 291)

```typescript
// src/plugins/hooks.ts:291
async function runModifyingHook<K extends PluginHookName, TResult>(
  hookName: K,
  event: ...,
  ctx: ...,
  policy: ModifyingHookPolicy<K, TResult> = {},
): Promise<TResult | undefined> {
  const hooks = getHooksForName(registry, hookName);
  let result: TResult | undefined;

  for (const hook of hooks) {
    try {
      const handlerResult = await (hook.handler as ...)(event, ctx);

      if (handlerResult !== undefined && handlerResult !== null) {
        if (policy.mergeResults) {
          result = policy.mergeResults(result, handlerResult, hook);
        } else {
          result = handlerResult;
        }
        if (result && policy.shouldStop?.(result)) {
          logger?.debug?.(`[hooks] ${hookName} decided by ${hook.pluginId}...`);
          policy.onTerminal?.({ hookName, pluginId: hook.pluginId, result });
          break;
        }
      }
    } catch (err) {
      handleHookError({ hookName, pluginId: hook.pluginId, error: err });
    }
  }

  return result;
}
```

Handlers run **one at a time, in priority order**. Each handler's result is passed to
`policy.mergeResults` to be accumulated. When `policy.shouldStop(result)` returns true,
the loop breaks immediately — lower-priority handlers are skipped.

The `ModifyingHookPolicy` controls three things:

- **`mergeResults`**: how to combine the accumulated result with the next handler's output
- **`shouldStop`**: when to short-circuit (e.g., `block: true` on `before_tool_call`)
- **`terminalLabel`**: the string logged at the break point (e.g., `"block=true"`)

Used for: `before_model_resolve`, `before_prompt_build`, `before_agent_start`,
`message_sending`, `before_tool_call`, `subagent_spawning`, `subagent_delivery_target`.

### 6c. Claiming — sequential, first handled wins (line 339)

```typescript
// src/plugins/hooks.ts:339
async function runClaimingHook<K extends PluginHookName, TResult extends { handled: boolean }>(
  hookName: K,
  event: ...,
  ctx: ...,
): Promise<TResult | undefined> {
  const hooks = getHooksForName(registry, hookName);
  // delegates to runClaimingHooksList...
}
```

Internally calls `runClaimingHooksList` which iterates handlers in priority order and
returns immediately when any handler returns `{ handled: true }`. The remaining handlers
are not invoked.

This is a special case of Modifying where `mergeResults` is replaced by "first claim
wins" semantics. Used for: `inbound_claim`, `before_dispatch`.

### 6d. Sync — inline loop, no async (hooks.ts:748 and 813)

Two hooks are executed via bespoke inline loops rather than `runModifyingHook`:
`tool_result_persist` and `before_message_write`. Both run on the session transcript
hot path where `async` is not acceptable.

```typescript
// src/plugins/hooks.ts:748 — tool_result_persist
function runToolResultPersist(event, ctx): PluginHookToolResultPersistResult | undefined {
  const hooks = getHooksForName(registry, "tool_result_persist");
  let current = event.message;

  for (const hook of hooks) {
    try {
      const out = (hook.handler as any)({ ...event, message: current }, ctx);

      // Guard against accidental async handlers
      if (out && typeof (out as any).then === "function") {
        logger?.warn?.(
          `[hooks] tool_result_persist handler from ${hook.pluginId} returned a Promise; ` +
            `this hook is synchronous and the result was ignored.`,
        );
        continue;
      }
      const next = (out as PluginHookToolResultPersistResult | undefined)?.message;
      if (next) current = next;
    } catch (err) {
      /* log + continue */
    }
  }
  return { message: current };
}
```

The async guard (`typeof out.then === "function"`) is critical: if a plugin registers an
async handler for a Sync hook, the runtime **silently drops the result and continues**.
No block, no error — just a warning log. This is a common mistake when porting a handler
from a Modifying hook.

**Set a breakpoint at line 769** (the `typeof (out as any).then === "function"` check) to
catch accidental async handlers during development.

---

## 7. Error handling

All four execution models share the same error sink:

```typescript
// src/plugins/hooks.ts:239
const handleHookError = (params: {
  hookName: PluginHookName;
  pluginId: string;
  error: unknown;
}): never | void => {
  const msg = `[hooks] ${params.hookName} handler from ${params.pluginId} failed: ${String(params.error)}`;
  if (catchErrors) {
    logger?.error(msg);
    return; // silently continue
  }
  throw new Error(msg, { cause: params.error });
};
```

When `catchErrors: true` (always the case in production, set at line 37 of
`hook-runner-global.ts`), any exception from any handler is:

1. Formatted into a single-line message with hook name and plugin ID
2. Logged at `error` level via the subsystem logger
3. Discarded — execution continues with the next handler

The first line of the error is extracted via `sanitizeHookError` (line 254) to avoid
multi-line stack traces polluting the single-line log format.

**Security implication:** A hook that throws never blocks the pipeline it is registered
on. A `before_tool_call` handler that throws does not block the tool call — the tool
proceeds as if no handler ran. This is intentional (resilience over security), but it
means hook-based security gates must never assume their handler ran successfully.

---

## 8. Merge strategies in Modifying hooks

Each Modifying hook defines its own `mergeResults` function. Understanding these strategies
is essential for predicting multi-plugin behavior.

### `firstDefined` vs `lastDefined`

Two primitive strategies are used throughout:

```typescript
// src/plugins/hooks.ts:180
const firstDefined = <T>(prev: T | undefined, next: T | undefined): T | undefined => prev ?? next;
const lastDefined = <T>(prev: T | undefined, next: T | undefined): T | undefined => next ?? prev;
```

- **`firstDefined`**: highest-priority plugin wins. Used for `modelOverride`,
  `providerOverride` in `before_model_resolve`. Once a high-priority plugin sets a
  value, lower-priority ones cannot override it.
- **`lastDefined`**: lowest-priority plugin wins. Used for `systemPrompt` in
  `before_prompt_build`. This is an additive override design where later (lower-priority)
  plugins can replace the system prompt set by earlier ones.

### Concatenation for context fields

```typescript
// src/plugins/hooks.ts:199
prependContext: concatOptionalTextSegments({
  left: acc?.prependContext,
  right: next.prependContext,
}),
```

`prependContext`, `prependSystemContext`, and `appendSystemContext` are concatenated across
all plugins. Every plugin that returns a non-empty string for these fields contributes to
the final value. The order is priority-descending (higher-priority plugin's text appears
first in the concatenation).

### Sticky boolean (`stickyTrue`) for `cancel` and `block`

```typescript
// src/plugins/hooks.ts:182
const stickyTrue = (prev?: boolean, next?: boolean): true | undefined =>
  prev === true || next === true ? true : undefined;
```

Once any handler sets `block: true` or `cancel: true`, that value cannot be unset by
lower-priority handlers. Used in `before_tool_call` (block) and `message_sending` (cancel).

---

## 9. Putting it together — a complete hook invocation trace

Here is a step-by-step trace for a `before_tool_call` invocation with two registered
plugins (Plugin A at priority 100, Plugin B at priority 50):

```
1. Agent tool dispatcher calls:
     if (hookRunner?.hasHooks("before_tool_call"))   // checks registry.typedHooks.some(...)
       → true (two handlers registered)

2. runBeforeToolCall(event, ctx) is called
     → calls runModifyingHook("before_tool_call", event, ctx, policy)

3. getHooksForName("before_tool_call") filters and sorts:
     [RegistrationA (priority=100), RegistrationB (priority=50)]

4. Loop iteration 1 — RegistrationA (pluginId="plugin-a"):
     handlerResult = await RegistrationA.handler(event, ctx)
     → { params: { command: "ls /tmp" } }   (param rewrite)
     mergeResults(undefined, handlerResult, regA)
     → accumulated = { params: { command: "ls /tmp" } }
     shouldStop({ params: ... })? → false (no block, no requireApproval)

5. Loop iteration 2 — RegistrationB (pluginId="plugin-b"):
     handlerResult = await RegistrationB.handler(event, ctx)
     → { requireApproval: { title: "Exec", ... } }
     mergeResults({ params: ... }, handlerResult, regB)
     → accumulated = {
         params: { command: "ls /tmp" },  // frozen — different plugin owns approval
         requireApproval: { ..., pluginId: "plugin-b" }  // pluginId stamped by runner
       }
     shouldStop(...)? → false (no block)

6. Loop ends (no more handlers).
   Return accumulated result to dispatcher.

7. Dispatcher checks hookResult.requireApproval → truthy → enters approval flow.
```

**Set a breakpoint at line 696** (`return runModifyingHook<"before_tool_call", ...>`) to
enter this trace in a live debug session.

---

## 10. Registration API

Plugins register hooks via `PluginRuntime.hooks.register`:

```typescript
// In your plugin's setup() function:
runtime.hooks.register("before_tool_call", {
  priority: 100,
  handler: async (event, ctx) => {
    // event: PluginHookBeforeToolCallEvent
    // ctx: PluginHookToolContext
    if (event.toolName === "exec") {
      return {
        requireApproval: { title: "Exec approval", description: "...", timeoutBehavior: "deny" },
      };
    }
  },
});
```

The `priority` field is optional (defaults to 0). The handler signature is typed via
`PluginHookHandlerMap[K]` — TypeScript enforces the correct event and context shapes for
each hook name.

---

## 11. Exercises

### Exercise 1 — Observe the `typedHooks` array at startup

1. Build with source maps: `OPENCLAW_BUILD_SOURCEMAP=1 pnpm build`
2. Set a breakpoint at `hook-runner-global.ts:44` (the `log.info` line after
   `createHookRunner`).
3. Start the gateway in debug mode. When the breakpoint hits:
   - Inspect `registry.typedHooks` — each entry is a `PluginHookRegistration`
   - Note the `pluginId`, `hookName`, `priority`, and `source` fields
   - Find entries for any bundled plugins that register hooks

### Exercise 2 — Trigger the async-handler warning for a Sync hook

1. Write a test plugin that registers `tool_result_persist` with an async handler:
   ```typescript
   runtime.hooks.register("tool_result_persist", {
     handler: async (event, ctx) => {
       await new Promise((r) => setTimeout(r, 10));
       return { message: event.message };
     },
   });
   ```
2. Install the plugin and run a tool call.
3. Observe the `[hooks] tool_result_persist handler from ... returned a Promise` warning
   in the gateway log.
4. Verify the tool result was NOT modified (the async return was dropped).

### Exercise 3 — Observe priority ordering under the debugger

1. Install two plugins that both register `before_prompt_build`:
   - Plugin A at `priority: 100` returning `{ prependContext: "FROM_A" }`
   - Plugin B at `priority: 50` returning `{ prependContext: "FROM_B" }`
2. Set a breakpoint at `hooks.ts:304` (the `result = policy.mergeResults(...)` line inside
   `runModifyingHook`).
3. Send a message to the agent. The breakpoint fires twice:
   - First iteration: `result = mergeResults(undefined, {prependContext:"FROM_A"}, regA)`
   - Second iteration: `result = mergeResults({...}, {prependContext:"FROM_B"}, regB)`
4. Inspect the final `result.prependContext` — it should be `"FROM_A\n\nFROM_B"` (concatenated,
   higher-priority first).

---

## 12. Key concepts checklist

After completing this session you should be able to answer:

- [ ] Where is `PluginHookRegistration` defined and what fields does it have?
- [ ] How does `getHooksForName` sort handlers?
- [ ] What happens when a Void hook handler throws?
- [ ] What is the difference between `firstDefined` and `lastDefined` merge strategies?
- [ ] Why does `tool_result_persist` use an inline sync loop instead of `runModifyingHook`?
- [ ] Why is `catchErrors: true` always set in production?
- [ ] What does `hasHooks` check and why is it called before every runner invocation?

---

## See Also

- [Session 2 — Agent Lifecycle Hooks](session2-agent-lifecycle-hooks.md)
- [plugin-security-hooks.md](../plugin-security-hooks.md) — security analysis
- [Gateway Session 1 — server.impl.ts](../../gateway/security/study/sessions/session1-server-impl.md) — where `initializeGlobalHookRunner` is called during startup
