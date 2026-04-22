---
title: "Security — Study and WG Materials"
summary: "Plugin architecture analysis, security hook design proposals, external research landscape, and Security WG working materials produced during the gateway security study"
read_when:
  - Reviewing plugin security architecture or hook design proposals
  - Preparing for the Security WG panel
  - Orienting to external research on OpenClaw security
---

# Security — Study and WG Materials

These documents were produced during the OpenClaw security study series.
They are not part of the official product documentation.

---

## Security Posture Analysis

| Document                                                     | Purpose                                                                                                                          |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| [security-posture-today.md](security-posture-today.md)       | What is and is not a security boundary today — guardrails vs hard boundaries, install-time controls, exec approval, tool policy. |
| [plugin-architecture-today.md](plugin-architecture-today.md) | Plugin packaging, discovery, loading, and SDK surface — structured as a "dig in by file" reading map.                            |

---

## Hook Design Analysis

| Document                                             | Purpose                                                                                                                                                           |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [plugin-security-hooks.md](plugin-security-hooks.md) | Deep analysis of `before_skill_install` and `before_tool_call` + `requireApproval` — architecture, specific concerns, and design proposals with TypeScript types. |

---

## Plugin Hooks — Source Walkthrough Series

Seven-session tutorial series covering all 26 plugin hooks: execution models, event/result shapes, merge strategies, call sites, and security implications — all grounded in source code.

| Session | Document                                                                                              | Hooks Covered                                                                                                                                          |
| ------- | ----------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1       | [session1-hook-system-architecture.md](plugin-hooks/session1-hook-system-architecture.md)             | Foundation: PluginHookRegistration, global singleton, priority, four execution models, error handling                                                  |
| 2       | [session2-agent-lifecycle-hooks.md](plugin-hooks/session2-agent-lifecycle-hooks.md)                   | `before_model_resolve`, `before_prompt_build`, `before_agent_start`, `llm_input`, `llm_output`, `agent_end`                                            |
| 3       | [session3-compaction-reset-hooks.md](plugin-hooks/session3-compaction-reset-hooks.md)                 | `before_compaction`, `after_compaction`, `before_reset`                                                                                                |
| 4       | [session4-message-flow-hooks.md](plugin-hooks/session4-message-flow-hooks.md)                         | `inbound_claim`, `message_received`, `before_dispatch`, `message_sending`, `message_sent`                                                              |
| 5       | [session5-tool-execution-hooks.md](plugin-hooks/session5-tool-execution-hooks.md)                     | `before_tool_call`, `after_tool_call`, `tool_result_persist`, `before_message_write`                                                                   |
| 6       | [session6-session-subagent-gateway-hooks.md](plugin-hooks/session6-session-subagent-gateway-hooks.md) | `session_start`, `session_end`, `subagent_spawning`, `subagent_spawned`, `subagent_delivery_target`, `subagent_ended`, `gateway_start`, `gateway_stop` |
| 7       | [session7-security-analysis.md](plugin-hooks/session7-security-analysis.md)                           | Cross-cutting security analysis: trust model, attack surfaces, missing gates, defense-in-depth                                                         |

Full index and 26-hook quick reference: [plugin-hooks/README.md](plugin-hooks/README.md)

---

## External Research (2026 Papers)

| Document                                                         | Purpose                                                                                                                                                                                                                                                         |
| ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [external-research-landscape.md](external-research-landscape.md) | Synthesis of four independent 2026 papers: ClawKeeper (arXiv:2603.24414), 190-advisory taxonomy (arXiv:2603.27517), HITL defense analysis (arXiv:2603.10387), FASA architecture (arXiv:2603.12644). Mapped to source code surfaces and existing study findings. |

---

## Security WG

| Document                                                   | Purpose                                                                                                                                                                  |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [security-wg-panel-agenda.md](security-wg-panel-agenda.md) | Panel agenda for converging on trust model, hook enforcement semantics, provenance model, and structural unified-policy decision. Updated with external research inputs. |

---

## Security Working Plan

| Document                                                         | Purpose                                                                                                                                                       |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [security-wg-executive-brief.md](security-wg-executive-brief.md) | One-page executive summary: mission, problem statement, objectives, roadmap tables, and success metrics — for management review and approval.                 |
| [security-working-plan.md](security-working-plan.md)             | Full working plan: three-stage roadmap (ongoing / planning / backlog) across four workstreams — Plugin Trust, Credential Provider RFC, IAM, and Supply Chain. |

---

## Case Studies

| Document                                                   | Purpose                                                                                                                                                                                                                                                                                                |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [case-skill-scan-flagging.md](case-skill-scan-flagging.md) | End-to-end analysis of the "Skill flagged" label: which of the two scan systems fires, why results are non-deterministic, what information is actually available, and what the path forward looks like. Includes factual assessment of a Gemini-generated developer guide against the actual codebase. |

---

## Implementation Proposals

| Document                                                                                   | Purpose                                                                                                                                                  |
| ------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [proposal-content-inspection-interception.md](proposal-content-inspection-interception.md) | Hook architecture analysis and `before_dispatch`-based solution for inbound content inspection with LLM-assisted blocking and group-chat discrimination. |
| [proposal-security-skill-scan-suppression.md](proposal-security-skill-scan-suppression.md) | Root-cause analysis and three-mechanism solution for legitimate security skills being flagged as suspicious/dangerous by the built-in static scanner.    |

---

## See Also

- [Official threat model](../THREAT-MODEL-ATLAS.md)
- [Gateway security study sessions](../../gateway/security/study/README.md)
- [Architecture Synthesis](../../gateway/security/study/architecture-synthesis.md)
- [STRIDE Threat Model](../../gateway/security/study/stride-threat-model.md) — threats 1–21
