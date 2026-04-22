---
title: "OpenClaw Security WG — Executive Brief"
summary: "One-page executive summary of the Security WG mission, problem statement, objectives, roadmap, and success metrics — for management review and approval"
status: draft
---

# OpenClaw Security WG — Executive Brief

## Mission

Establish a security-first plugin and identity architecture for OpenClaw that enables
enterprise adoption, protects against supply chain threats, and supports partner
security teams operating at scale — without breaking the existing operator-trust model.

---

## Problem Statement

OpenClaw is widely adopted as a personal and team productivity platform, and is now
entering enterprise environments where security guarantees, auditability, and access
control are baseline requirements. Four structural gaps must be closed for
enterprise-grade deployment, with LLM security protection emerging as an additional
priority:

| Area             | Current Gap                                                                                                                 |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------- |
| **Plugin Trust** | All plugins run with equal in-process privilege; no enforcement boundary separates a security plugin from a compromised one |
| **Credentials**  | `SecretRef` cannot handle token refresh, rotation events, or federated identity (SPIFFE/OIDC)                               |
| **IAM**          | No built-in sender identity model; tool policy cannot be scoped by role, team, or group membership                          |
| **Supply Chain** | Static scanner produces false positives on legitimate security skills; no signing or provenance                             |
| **LLM Security** | No prompt-level guardrails; LLM inputs and outputs are not inspected for injection, exfiltration, or policy violations      |

---

## Objectives

1. Define and ship a **plugin trust tier** — designated security plugins can enforce
   policy with block semantics, protected from subversion by lower-trust plugins.
2. Enable **enterprise credential management** via a plugin-extensible credential
   provider supporting token lifecycle, rotation, and federated identity (RFC #59165).
3. Introduce a **built-in identity model** (sender roles, group context) enabling
   team-scoped access policies without replacing the single-operator trust model.
4. Harden **supply chain integrity** with scan suppression, provenance receipts, and
   eventual signing — unblocking security skill developers immediately.
5. Add **LLM security protection** — prompt guards, input/output inspection hooks,
   and injection detection to protect against prompt injection and data leakage at
   the model boundary.

---

## Roadmap

### Ongoing Short-Term Tasks

_Priority focus: complete these first. Target: Q2 2026._

**1A. OpenClaw Security WG — Plugin Trust Model**

Get deeply involved in security topics proposed by the OpenClaw Security WG. The
latest discussion covers the following areas and partner feedback is expected before
4/11:

- _Topic 0: Foundational Trust Model and Security Policy_ — close operational gaps:
  runtime plugin disablement, built-in plugin override, enforcement plugin self-protection.
- _Topic 1: Trust Model — Who Gets to Run Security Plugins?_ — converge on enforcement
  model (enterprise policy gate vs. vendor attestation vs. curated registry); produce
  trust model decision memo.
- _Topic 2: Hook Design — Do the Primitives Cover Security Needs?_ — ship the missing
  `before_skill_install` hook (proposed in PR #56050, currently absent); produce hook
  execution semantics spec (timeouts, priority ordering, fail-closed behavior).

For more details, refer to the
[OpenClaw Security WG Plugin Trust Model Discussion Draft](https://bytedance.larkoffice.com/docx/UkoJdaDBko8yVBxHgYlcdnCzn7c).

**1B. RFC: Credential Provider Plugin**

Credential leakage is one of the major security challenges in enterprise environments.
The current `SecretRef` system (`env` / `file` / `exec`) has significant limitations
in scenarios involving short-lived tokens, rotation events, and federated identity.
We are actively working on a proposal — the **Credential Provider Plugin** — to
provide a first-class, plugin-extensible option for managing credentials through a
dedicated provider with full lifecycle support (resolve, refresh, expiry, audit).

Partner review of RFC #59165 is underway; feedback is being collected from enterprise
security teams to validate the API design and trust tier requirements.

For details, refer to
[RFC: Credential Provider Plugin #59165](https://github.com/openclaw/openclaw/issues/59165).

**1C. Supply Chain — Scan Suppression (Immediate Unblock)**

Security skill developers are blocked today: every skill upload is flagged
suspicious or critical by the static scanner, even when the flagged behaviors
(`exec`, `readFile` + network, `process.env`) are canonical for security tools.
We are implementing scan suppression mechanisms M1 (skill-author manifest declaration)
and M2 (operator config allowlist) to downgrade acknowledged findings from critical to
informational — unblocking skill development without removing auditability.

**1D. LLM Security Protection — Prompt Guard (New)**

As OpenClaw processes increasing volumes of enterprise messages and tool calls, the
LLM boundary itself becomes an attack surface. Prompt injection attacks, sensitive data
leakage in model inputs, and policy-violating model outputs are emerging threat vectors.

We will propose and prototype a **prompt guard** capability, adding an inspection layer
at the LLM input and output boundary:

- _Inbound prompt inspection_ — detect and block prompt injection patterns before the
  model receives them (using the existing `before_dispatch` claiming hook as the
  interception point).
- _Outbound response inspection_ — scan model outputs for sensitive data patterns
  (PII, credential strings, confidential keywords) before delivery to end users.
- _Policy-driven allow/block rules_ — operator-configurable rule sets, with support
  for partner security plugin extensions once the enforcement tier (Topic 1) lands.

---

### Planning Near-Term Tasks

_In progress alongside community; continue after short-term tasks deliver._

**2A. IAM — Identity and Access Control**

The IAM model is one of the most immediate security challenges for enterprise customers
adopting OpenClaw. Enterprise users are struggling to manage who can access what resources
in what contexts. We plan to propose a built-in identity model in OpenClaw — introducing
structured `SenderIdentity` (stable canonical ID, operator-assigned roles, group
memberships) — and then further build access control capabilities:

- Extend `tools.policy` to support identity-based conditions (`sender.roles`, `sender.channel`).
- Expose group context (`groupId`, `resolvedSenderPolicy`) to hook plugins, enabling
  per-sender security policy in group-chat deployments.
- Work with the OpenClaw community to land the full IAM specification and external
  IdP integration (LDAP / OIDC) as mid-term follow-on work.

**2B. Supply Chain — Scanner Hardening and Provenance**

Supply chain integrity is a significant problem for enterprise plugin ecosystems.
Beyond the immediate scan suppression fix, we will work with the OpenClaw community on
a broader supply chain plan:

- _AST-level scanner rules_ — replace regex heuristics with AST analysis to reduce
  false positives and distinguish safe from dangerous `exec` patterns.
- _Pre-download scan_ — tarball-level scan before `npm install` runs, closing the
  `postinstall` script execution gap.
- Longer term: skill and plugin signing, provenance receipts (`.scan-receipt.json`),
  and ClawHub registry integration — all coordinated with the community roadmap.

**2C. Plugin Enforcement Tier**

Following the trust model decision (Topic 1), implement the enforcement hook tier:
add `plugins.securityHooks.allowedPluginIds` to the config schema, demoting
non-allowlisted plugins to advisory (void) semantics at registration time. This is the
prerequisite for the credential provider, supply chain scanner plugin, and LLM prompt
guard to operate with block semantics rather than advisory-only observation.

---

### Goal Mid/Long Term — Community Collaboration Backlog

_These items will be contributed and tracked through the OpenClaw community roadmap._

Watcher process API · `before_plugin_install` hook · Network egress policy hook ·
Vault reference credential provider plugin · Credential handling security audit ·
IAM full specification · LDAP/OIDC external IdP integration · RBAC-lite resource-scoped
access control · Session-scoped identity · Per-sender `groupScope` session partitioning ·
Group role model (replace `senderIsOwner`) · Skill/plugin signing scheme ·
Provenance receipt (`.scan-receipt.json`) · ClawHub registry scan UI ·
Supply chain scanner plugin capability · LLM output redaction and audit trail

---

## Expected Outcomes

Successful delivery of this work plan will:

- Enable partner security teams to **deploy enforcement plugins** with reliable block
  semantics and a trust model the OpenClaw community can verify.
- Give enterprise customers **native credential lifecycle management**, removing fragile
  `exec`-based credential workarounds.
- Make **group-chat and multi-team deployments safe** — sender identity and per-sender
  policy closes the gap that today forces teams to restrict OpenClaw to single-user use.
- **Unblock security skill developers immediately** via scan suppression — removing the
  false-positive barrier to publishing skills on ClawHub.
- Add a **prompt guard layer** that protects enterprises from prompt injection and data
  leakage at the LLM boundary, a capability differentiator for security-conscious
  enterprise customers.
- Position OpenClaw as **enterprise-ready** with the enforcement, credential, identity,
  and supply chain controls that enterprise security teams require for production approval.

### KPIs for Successful Delivery

| KPI                                                   | Target                                                     |
| ----------------------------------------------------- | ---------------------------------------------------------- |
| Partner feedback collected (Q1–Q3)                    | Before 4/11                                                |
| Trust model decision memo approved by WG              | End of Ongoing Short-Term phase                            |
| `before_skill_install` hook shipped and merged        | End of Ongoing Short-Term phase                            |
| Scan suppression M1 + M2 shipped                      | End of Ongoing Short-Term phase                            |
| False-positive rate on legitimate security skills     | Zero unacknowledged critical/warn findings after T12 lands |
| Prompt guard prototype available for partner review   | End of Ongoing Short-Term phase                            |
| Enforcement hook tier available and documented        | End of Planning Near-Term phase                            |
| Credential Provider RFC merged and SDK API published  | End of Planning Near-Term phase                            |
| Security plugin deployable with block semantics       | At least one partner plugin using enforcement tier         |
| Enterprise deployment with identity-based tool policy | First customer pilot during Planning Near-Term phase       |

---

## Risks and Mitigation

| Risk                                                   | Likelihood | Impact | Mitigation                                                                       |
| ------------------------------------------------------ | ---------- | ------ | -------------------------------------------------------------------------------- |
| Trust model decision stalls in WG debate               | Medium     | High   | Time-box discussion; default to enterprise policy gate if no convergence by 4/11 |
| `before_skill_install` deferred again                  | Medium     | High   | Assign dedicated owner; track as WG milestone; link to supply chain urgency      |
| RFC #59165 feedback requires API redesign              | Low        | Medium | Publish early with clear deadline; design with extension points                  |
| LLM prompt guard performance overhead unacceptable     | Medium     | Medium | Benchmark early; make inspection async and configurable; allow operator opt-out  |
| IAM model creates migration friction                   | Medium     | Medium | Design as purely additive — existing deployments unchanged without config        |
| Community prioritization misalignment on backlog items | High       | Low    | Maintain active WG participation; contribute PRs directly for critical items     |

---

## External References

| Document                                                                                                                          | Description                                                                                   |
| --------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| [OpenClaw Security WG — Plugin Trust Model (Discussion Draft)](https://bytedance.larkoffice.com/docx/UkoJdaDBko8yVBxHgYlcdnCzn7c) | WG discussion draft on plugin trust model — foundational input for T01                        |
| [RFC: Credential Provider Plugin #59165](https://github.com/openclaw/openclaw/issues/59165)                                       | GitHub RFC for the plugin-extensible credential provider — basis for T06/T07                  |
| [OpenClaw Security and Sandboxing](https://docs.openclaw.ai/gateway/security)                                                     | Official docs: current security model, operator trust boundary, out-of-scope definition       |
| [ClawSentry LT Planning One Pager](https://bytedance.larkoffice.com/wiki/Ggifw3Ziti2RG2k03nVclDTMnKd)                             | Leadership planning one-pager for the ClawSentry security initiative                          |
| [OpenClaw 需求讨论草案](https://bytedance.larkoffice.com/docx/Psv5dwDrOoSIcyx5LZpczAKknoh)                                        | Requirements discussion draft (Chinese) — internal product requirements for security features |
| [Full Security Working Plan](security-working-plan.md)                                                                            | Complete task breakdown, workstream details, and dependency map                               |
