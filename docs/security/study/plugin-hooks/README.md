---
title: "Plugin Hook System — Study Series"
summary: "Seven structured study sessions covering the OpenClaw plugin hook system: runner architecture, all 26 named hooks, their pipelines, event and context shapes, merge semantics, and a security analysis of attack surfaces exposed by each hook category"
read_when:
  - Orienting to the plugin hook study materials
  - Looking for a specific hook's technical reference
  - Evaluating the security posture of the plugin hook system
---

# Plugin Hook System — Study Series

These documents are a structured source walkthrough of the OpenClaw plugin hook system.
They complement the architectural overview in
[plugin-security-hooks.md](../plugin-security-hooks.md) with step-by-step source code
walkthroughs, precise line numbers, and exercises that build intuition for how each hook
integrates into the runtime pipeline.

**Prerequisites:** Familiarity with the plugin manifest format and the `PluginRuntime` SDK
surface described in [plugin-architecture-today.md](../plugin-architecture-today.md).
Sessions 5 and 7 assume prior reading of the gateway tool dispatch session
([sessions/session6-tool-dispatch.md](../../gateway/security/study/sessions/session6-tool-dispatch.md)).

---

## Session Index

| Session | Topic                                | File                                                                                     |
| ------- | ------------------------------------ | ---------------------------------------------------------------------------------------- |
| 1       | Hook System Architecture             | [session1-hook-system-architecture.md](session1-hook-system-architecture.md)             |
| 2       | Agent Lifecycle Hooks                | [session2-agent-lifecycle-hooks.md](session2-agent-lifecycle-hooks.md)                   |
| 3       | Compaction and Reset Hooks           | [session3-compaction-reset-hooks.md](session3-compaction-reset-hooks.md)                 |
| 4       | Message Flow Hooks                   | [session4-message-flow-hooks.md](session4-message-flow-hooks.md)                         |
| 5       | Tool Execution Hooks                 | [session5-tool-execution-hooks.md](session5-tool-execution-hooks.md)                     |
| 6       | Session, Subagent, and Gateway Hooks | [session6-session-subagent-gateway-hooks.md](session6-session-subagent-gateway-hooks.md) |
| 7       | Security Analysis                    | [session7-security-analysis.md](session7-security-analysis.md)                           |

Read in order. Session 1 is the foundation; every subsequent session assumes you
understand the four execution models and the global singleton established there.

**Sequence diagrams:** [hook-sequence-diagrams.md](hook-sequence-diagrams.md) — 9 Mermaid diagrams covering all 26 hooks across every pipeline, plus a full execution model reference table.

---

## Security Hardening Implementation Guides

Practical multi-hook combination patterns for security plugin authors. Read Session 5
and Session 7 first.

| Document                                                             | Topic                                                                                   |
| -------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| [hardening-hook-combinations.md](hardening-hook-combinations.md)     | Analysis of why single hooks fail and the general shared-state bridge pattern           |
| [hardening-combo-intercept.md](hardening-combo-intercept.md)         | Post-tool detection with interception: block result + prevent further tool calls        |
| [hardening-combo-prompt-modify.md](hardening-combo-prompt-modify.md) | Post-tool detection with prompt modification: rewrite LLM instructions without blocking |

---

## Hook Quick Reference

All 26 named hooks, their execution model, and the session where they are covered in depth.

| Hook                       | Model              | Session |
| -------------------------- | ------------------ | ------- |
| `before_model_resolve`     | Modifying          | 2       |
| `before_prompt_build`      | Modifying          | 2       |
| `before_agent_start`       | Modifying (legacy) | 2       |
| `llm_input`                | Void               | 2       |
| `llm_output`               | Void               | 2       |
| `agent_end`                | Void               | 2       |
| `before_compaction`        | Void               | 3       |
| `after_compaction`         | Void               | 3       |
| `before_reset`             | Void               | 3       |
| `inbound_claim`            | Claiming           | 4       |
| `message_received`         | Void               | 4       |
| `before_dispatch`          | Claiming           | 4       |
| `message_sending`          | Modifying          | 4       |
| `message_sent`             | Void               | 4       |
| `before_tool_call`         | Modifying          | 5       |
| `after_tool_call`          | Void               | 5       |
| `tool_result_persist`      | Sync               | 5       |
| `before_message_write`     | Sync               | 5       |
| `session_start`            | Void               | 6       |
| `session_end`              | Void               | 6       |
| `subagent_spawning`        | Modifying          | 6       |
| `subagent_delivery_target` | Modifying          | 6       |
| `subagent_spawned`         | Void               | 6       |
| `subagent_ended`           | Void               | 6       |
| `gateway_start`            | Void               | 6       |
| `gateway_stop`             | Void               | 6       |

---

## Key Source Files

| File                                                          | Role                                                                  |
| ------------------------------------------------------------- | --------------------------------------------------------------------- |
| `src/plugins/types.ts`                                        | All hook name, event, context, and result types                       |
| `src/plugins/hooks.ts`                                        | `createHookRunner` — four execution models, all 26 runner functions   |
| `src/plugins/hook-runner-global.ts`                           | Global singleton: `initializeGlobalHookRunner`, `getGlobalHookRunner` |
| `src/agents/pi-embedded-runner/run/setup.ts`                  | `before_model_resolve` + `before_agent_start` call site               |
| `src/agents/pi-embedded-runner/run/attempt.prompt-helpers.ts` | `before_prompt_build` call site                                       |
| `src/agents/pi-embedded-runner/run/attempt.ts`                | `llm_input`, `llm_output`, `agent_end` call sites                     |
| `src/agents/pi-embedded-runner/compaction-hooks.ts`           | `before_compaction`, `after_compaction` call sites                    |
| `src/auto-reply/reply/commands-core.ts`                       | `before_reset` call site                                              |
| `src/auto-reply/reply/dispatch-from-config.ts`                | `inbound_claim`, `message_received`, `before_dispatch` call sites     |
| `src/infra/outbound/deliver.ts`                               | `message_sending`, `message_sent` call sites                          |
| `src/agents/pi-tools.before-tool-call.ts`                     | `before_tool_call` call site + approval flow                          |
| `src/agents/session-tool-result-guard-wrapper.ts`             | `tool_result_persist`, `before_message_write` call sites              |
| `src/auto-reply/reply/session.ts`                             | `session_start`, `session_end` call sites                             |
| `src/agents/subagent-spawn.ts`                                | `subagent_spawning`, `subagent_spawned` call sites                    |
| `src/agents/subagent-announce-delivery.ts`                    | `subagent_delivery_target` call site                                  |
| `src/agents/subagent-registry-completion.ts`                  | `subagent_ended` call site                                            |
| `src/gateway/server.impl.ts`                                  | `gateway_start` call site                                             |
| `src/plugins/hook-runner-global.ts`                           | `gateway_stop` call site                                              |

---

## See Also

- [plugin-security-hooks.md](../plugin-security-hooks.md) — architectural overview and security proposals
- [plugin-architecture-today.md](../plugin-architecture-today.md) — plugin manifest, SDK surface, trust model
- [Gateway Source Walkthrough Sessions](../../gateway/security/study/sessions/) — prerequisite for Sessions 5 and 7
- [hardening-hook-combinations.md](hardening-hook-combinations.md) — multi-hook combination analysis (start here for security plugin design)
- [hardening-combo-intercept.md](hardening-combo-intercept.md) — implementation: post-tool interception
- [hardening-combo-prompt-modify.md](hardening-combo-prompt-modify.md) — implementation: post-tool prompt modification
