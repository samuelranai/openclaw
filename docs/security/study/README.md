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
