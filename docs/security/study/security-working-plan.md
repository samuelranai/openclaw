---
title: "OpenClaw Security Working Plan — Ongoing and Planning Roadmap"
summary: "Security WG working plan: problem statement, objectives, task roadmap, expected outcomes, and risks across Plugin Trust Model, Credential Provider RFC, IAM identity model, and Supply Chain security"
status: draft
---

# OpenClaw Security Working Plan

## Mission

The OpenClaw Security Working Group (Security WG) is responsible for defining the
security architecture, trust model, and enforcement mechanisms for the OpenClaw
platform. This document sets the WG's working goals and execution roadmap,
structured for weekly review meetings. The four active workstreams address: the
plugin trust and enforcement model, enterprise credential management, built-in
identity and access control, and supply chain integrity.

---

## Problem Statement

OpenClaw is widely adopted as a personal and team productivity platform, and is
now entering enterprise environments where security guarantees, auditability, and
access control become baseline requirements. Four structural gaps must be closed
to make enterprise-grade deployment viable:

| Workstream       | Problem Summary                                                                                                                                                                                                                                             |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Plugin Trust** | All plugins run with identical in-process privilege. No enforcement boundary separates a security enforcer plugin from a compromised one. Once a malicious plugin is identified, there is no runtime mechanism to revoke it without a full gateway restart. |
| **Credentials**  | The current `SecretRef` system (`env`/`file`/`exec`) cannot manage token lifecycle: no refresh, no rotation event handling, no federated identity (SPIFFE/OIDC). Enterprise credential buses require a plugin-extensible provider model.                    |
| **IAM**          | There is no built-in sender identity model. Tool and resource access cannot be scoped by role, team, or group membership. In group-chat deployments, all participants share a single session key — tool state and history are not isolated per sender.      |
| **Supply Chain** | The static scanner produces false positives on legitimate security skills (e.g., any tool spawning `nmap` is flagged critical). There is no signing, provenance, or tamper detection on installed skills and plugins.                                       |

For detailed technical analysis of each workstream, see the [Plan Details](#plan-details) section.

---

## Objectives

1. **Establish a plugin enforcement tier** — define and ship a trust model that allows
   designated security plugins to enforce policy (block tool calls, block installs) while
   being protected from subversion by lower-trust plugins or LLM-triggered actions.

2. **Enable enterprise credential management** — deliver a plugin-extensible credential
   provider API (RFC #59165) that supports token refresh, rotation events, federated
   identity, and structured audit trails — replacing `exec`-based credential workarounds.

3. **Introduce sender identity and access control** — add a built-in `SenderIdentity`
   model enabling team-scoped tool policies, per-sender group context in hook plugins,
   and a foundation for RBAC in enterprise deployments.

4. **Harden supply chain integrity** — unblock security skill developers by shipping
   scan suppression today, and lay the groundwork for skill/plugin signing, provenance
   receipts, and registry-level security checks.

---

## Roadmap

### Ongoing Short-Term Tasks

Active work with highest priority. Tasks are self-contained or have few external
dependencies, and directly unblock developers or WG decision-making.

| ID  | Workstream   | Task                                           | Type     |
| --- | ------------ | ---------------------------------------------- | -------- |
| T01 | Plugin Trust | Trust model decision memo (WG converge)        | WG       |
| T02 | Plugin Trust | Ship `before_skill_install` hook               | Eng      |
| T04 | Plugin Trust | Hook execution semantics spec                  | WG + Doc |
| T05 | Plugin Trust | Runtime plugin disablement + self-protection   | Eng      |
| T06 | Credentials  | RFC #59165 partner review (collect feedback)   | WG       |
| T11 | IAM          | Group context exposed to hook plugins          | Eng      |
| T12 | Supply Chain | Scan suppression M1 + M2 (unblocks skill devs) | Eng      |

> **Note:** T02 and T12 share the same engineering prerequisite (`before_skill_install`)
> and are coordinated across WS1 and WS4.

### Planning Near-Term Tasks

Tasks that depend on ongoing short-term decisions landing, or require cross-team
design alignment before implementation begins.

| ID  | Workstream   | Task                                        | Type   | Depends on |
| --- | ------------ | ------------------------------------------- | ------ | ---------- |
| T03 | Plugin Trust | Enforcement hook tier config key            | Eng    | T01        |
| T15 | Plugin Trust | Enforcement hook tier shipped + documented  | Eng    | T01, T03   |
| T07 | Credentials  | Credential Provider API design              | Design | T06        |
| T19 | Credentials  | `"plugin"` SecretRefSource implementation   | Eng    | T06, T07   |
| T20 | Credentials  | Plugin SDK `registerCredentialProvider` API | Eng    | T19        |
| T08 | IAM          | Sender identity primitives                  | Eng    | —          |
| T09 | IAM          | Per-channel identity-based access policy    | Eng    | T08        |
| T10 | IAM          | `before_dispatch` identity enrichment       | Eng    | T08        |
| T13 | Supply Chain | AST-level and custom scanner rules          | Eng    | T02        |
| T14 | Supply Chain | Pre-download scan for npm-sourced skills    | Eng    | T02        |

### Goal Mid/Long Term — Backlog

Larger architectural work requiring earlier stages to land. Listed for awareness;
not actively scheduled until near-term tasks complete.

| ID  | Workstream   | Task                                                           | Depends on |
| --- | ------------ | -------------------------------------------------------------- | ---------- |
| T16 | Plugin Trust | Watcher process API (external security monitoring)             | T15        |
| T17 | Plugin Trust | `before_plugin_install` hook                                   | T15        |
| T18 | Plugin Trust | Network egress policy hook                                     | T15        |
| T21 | Credentials  | Vault reference credential provider plugin                     | T20        |
| T22 | Credentials  | Credential handling security audit                             | T20        |
| T23 | IAM          | IAM full specification document                                | T08        |
| T24 | IAM          | LDAP / OIDC external IdP integration                           | T23        |
| T25 | IAM          | RBAC-lite: resource-scoped access control                      | T23        |
| T26 | IAM          | Session-scoped identity (prevent mid-session escalation)       | T23        |
| T27 | IAM          | `groupScope`: per-sender session partitioning                  | T11        |
| T28 | IAM          | Group role model (replace binary `senderIsOwner`)              | T11        |
| T29 | Supply Chain | Skill and plugin signing scheme                                | T02        |
| T30 | Supply Chain | Provenance receipt (`.scan-receipt.json`) for tamper detection | T02        |
| T31 | Supply Chain | ClawHub registry: scan summaries + signing UI                  | T29        |
| T32 | Supply Chain | Supply chain scanner plugin capability (OSV, Snyk, npm audit)  | T15, T17   |

---

## Expected Outcomes

### Value of Collaboration with OpenClaw

This work plan represents a joint effort between our security team and the OpenClaw
open-source community. The expected outcomes from this collaboration are:

- **Security plugins become first-class citizens** — partner security teams can ship
  enforcement plugins that reliably block threats, with a trust model that the OpenClaw
  community and enterprise customers can verify and depend on.
- **Enterprise credential workflows are supported natively** — customers currently
  working around `SecretRef` limitations with fragile `exec` scripts gain a structured,
  auditable, lifecycle-aware credential provider API contributed upstream.
- **Group-chat and multi-team deployments become safe** — adding sender identity and
  per-sender policy closes the gap that today forces enterprise teams to restrict OpenClaw
  to single-user deployments or accept unscoped tool access in group channels.
- **Security skill development is unblocked** — scan suppression (T12) lands immediately,
  removing the false-positive barrier that currently prevents security teams from
  publishing and sharing skills in ClawHub.
- **OpenClaw positions itself as enterprise-ready** — the combination of enforcement
  hooks, credential provider, identity model, and supply chain hardening gives enterprise
  security teams the controls they need to approve OpenClaw for production deployment.

### KPIs for Successful Delivery

| KPI                                                   | Target                                                      |
| ----------------------------------------------------- | ----------------------------------------------------------- |
| Partner security team feedback collected (Q1–Q3)      | Completed before 4/11                                       |
| Trust model decision memo approved by WG              | By end of Ongoing Short-Term phase                          |
| `before_skill_install` hook shipped and merged        | By end of Ongoing Short-Term phase                          |
| Scan suppression M1 + M2 shipped                      | By end of Ongoing Short-Term phase                          |
| False-positive rate on legitimate security skills     | Zero unacknowledged critical/warn findings after T12 lands  |
| Enforcement hook tier available and documented        | By end of Planning Near-Term phase                          |
| Credential Provider RFC merged and SDK API published  | By end of Planning Near-Term phase                          |
| Security plugin deployable with block semantics       | At least one partner security plugin using enforcement tier |
| Enterprise deployment with identity-based tool policy | First customer pilot during Planning Near-Term phase        |
| WG weekly meeting cadence maintained                  | Zero consecutive missed syncs; action items tracked         |

---

## Risks and Mitigation

| Risk                                                                       | Likelihood | Impact                                              | Mitigation                                                                                                                   |
| -------------------------------------------------------------------------- | ---------- | --------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| Trust model decision stalls in WG debate (T01)                             | Medium     | High — blocks T03, T15, and all enforcement work    | Time-box WG discussion; escalate to maintainers if no convergence by 4/11; default to enterprise policy gate as fallback     |
| `before_skill_install` hook is re-scoped or deferred again (T02)           | Medium     | High — blocks T12–T14, T29–T32                      | Assign dedicated owner; track as a milestone in WG weekly; link to supply chain gaps to signal urgency                       |
| RFC #59165 partner feedback reveals API redesign needed (T06)              | Low        | Medium — delays credential workstream by one cycle  | Publish RFC early with clear feedback deadline; design API with extension points to absorb scope changes without full rework |
| Enforcement tier config demoted to advisory-only due to community pushback | Low        | High — negates security value of plugin trust model | Frame as opt-in operator config (off by default); document clearly that enforcement requires explicit operator enablement    |
| IAM identity model creates migration friction for existing deployments     | Medium     | Medium — slows adoption                             | Design as additive and progressive — deployments without identity config continue working unchanged                          |
| Security skill false positives not fixed quickly enough (T12)              | Low        | Medium — partner security teams lose confidence     | T12 has no blocking dependencies; can ship independently of all other workstreams                                            |
| External dependency: ClawHub registry integration timeline unknown (T31)   | High       | Low — affects mid/long-term only                    | T31 is backlog; no short-term dependency; flag early to ClawHub team as a planning input                                     |

---

## Plan Details

Detailed technical analysis, design decisions, and task breakdowns for each workstream.

### Workstream 1 — Plugin Trust Model

OpenClaw's plugin system has no privilege tiers. All loaded plugins run in-process
with operator-level trust. This is correct for the current personal-use trust model,
but creates a structural problem for enterprise security deployments:

- A "security enforcer" plugin and a potentially compromised plugin have identical trust.
- There is no enforcement boundary; a lower-trust plugin can disable a security plugin.
- Hook primitives exist (`before_tool_call`, `before_dispatch`) but any plugin can
  register them — there is no mechanism to reserve enforcement semantics for trusted
  plugins only.
- Once a malicious plugin is discovered, the operator has no way to revoke it without
  restarting the gateway.

**WG Open Topics**

_Topic 0 — Foundational Trust and Security Policy (T01, T05)_

Three operational gaps to close regardless of which trust model is chosen: (1) runtime
disablement — terminate a plugin's registrations without gateway restart; (2) built-in
plugin override — quarantine a bundled plugin via config without binary rebuild;
(3) enforcement plugin self-protection — shield enforcement-tier plugins from being
disabled by lower-trust plugins or LLM-triggered tool calls.

_Topic 1 — Who Gets to Run Security Plugins? (T01, T03)_

Three candidate models under WG review:

| Model                  | Strength                    | Weakness                                         |
| ---------------------- | --------------------------- | ------------------------------------------------ |
| Vendor attestation     | Strong chain of custody     | PKI infrastructure; blocks community tools       |
| Enterprise policy gate | Operator-controlled; no PKI | Trust is config, not code — can be misconfigured |
| Curated registry       | Balanced                    | Centralized; latency for new security tools      |

WG recommendation: Enterprise policy gate (`plugins.securityHooks.allowedPluginIds`)
as the initial path; OpenClaw-operated signing service as the upgrade path (analogous
to [Windows driver signing](https://learn.microsoft.com/en-us/windows-hardware/drivers/install/driver-signing)).
Tiered privilege model (Enforcement / Verified / Community) applies regardless of
which signing model is chosen.

_Topic 2 — Do Hook Primitives Cover Security Needs? (T02, T04)_

| Hook                    | Status                                         |
| ----------------------- | ---------------------------------------------- |
| `before_dispatch`       | ✅ Claiming hook; awaited; `isGroup` present   |
| `before_tool_call`      | ✅ Shipped; modifying hook with `block`        |
| `before_skill_install`  | ⚠️ Not in `PluginHookName` union — not shipped |
| `before_compaction`     | ⚠️ Void only — cannot block                    |
| `before_plugin_install` | ❌ Does not exist                              |
| `llm_input`             | ⚠️ Void — cannot block                         |
| Network egress          | ❌ Does not exist                              |

See [plugin-security-hooks.md](plugin-security-hooks.md) for full design analysis.

---

### Workstream 2 — Credential Provider Plugin (RFC #59165)

The current `SecretRef` system (`env` / `file` / `exec`) cannot handle credential
lifecycle scenarios enterprise deployments require: token refresh, rotation push events,
federated identity (SPIFFE/OIDC), structured audit trails, or org-wide credential bus
routing. A new `"plugin"` source extends `SecretRefSource` with a plugin-registered
`resolve(event)` callback and gateway-managed token refresh lifecycle.

Security constraint: only plugins in `plugins.securityHooks.allowedPluginIds` may
register credential providers. Values are in-memory only — never written to config,
logs, or telemetry.

See [src/config/types.secrets.ts](../../../src/config/types.secrets.ts) for the
current `SecretRefSource` implementation and RFC #59165 for the full proposal.

---

### Workstream 3 — IAM: Identity and Access Control

There is no built-in concept of who is sending a message beyond a channel-derived
`senderId` and a boolean authorized flag. The IAM model adds policy richness within
the existing single-operator trust boundary — it does not replace the operator model
or create OS-level per-user isolation.

**Group-chat security gaps** (five structural gaps in multi-sender group deployments):

| Gap | Description                                                                                  |
| --- | -------------------------------------------------------------------------------------------- |
| G1  | Group participants share one session key — tool state and history visible across all senders |
| G2  | `senderIsOwner` is binary — no role gradation within groups                                  |
| G3  | `toolsBySender` is static config — plugins cannot read or modify it at hook time             |
| G4  | No `groupScope` — `dmScope` supports per-peer isolation; groups have none                    |
| G5  | `before_dispatch` carries `isGroup` but no stable `groupId` for multi-group policy           |

Tasks T11, T27, and T28 address these gaps progressively.

---

### Workstream 4 — Supply Chain Security

The built-in static scanner applies regex pattern rules at install time. Four rules
(`dangerous-exec`, `env-harvesting`, `potential-exfiltration`, `suspicious-network`)
fire as critical/warn on any legitimate security tool that spawns processes, reads
files, or calls a SIEM API. There is no suppression mechanism, no signing, and no
tamper detection on installed files.

Full analysis and M1/M2/M3 suppression design in
[proposal-security-skill-scan-suppression.md](proposal-security-skill-scan-suppression.md).

---

## Cross-Cutting Concerns

### Workstream Dependencies

```
T01 (Trust model decision)
  └── T03 (Enforcement tier config)
        └── T15 (Enforcement tier shipped)
              ├── T16, T17, T18 (Watcher / plugin install / egress hooks)
              └── T32 (Supply chain scanner plugin)

T06/T07 (Credentials RFC + design)
  └── T19 → T20 → T21/T22 (Credential provider implementation)

T02 (before_skill_install)
  ├── T12 (Scan suppression — shared prerequisite)
  └── T13, T14, T29, T30 (Scanner + provenance work)

T08 (Sender identity primitives)
  └── T09, T10 (Policy, dispatch enrichment)
        └── T23 → T24–T28 (Full IAM implementation)

T11 (Group context for hooks)
  └── T27, T28 (groupScope + group role model)
```

**Critical path:** T01 → T03 → T15 is the bottleneck for enforcement, credential
provider, and supply chain scanner plugin. The trust model decision (T01) is the
top WG priority.

### Failure Posture Policy

All enforcement hooks default to **fail-closed** when an enforcement tier is
configured, and **fail-open** (with logged warning) when not configured. Enforcement
is opt-in; once opted in, failures must not silently permit blocked operations.

---

## Open Questions for the Security WG

Input needed from partner security teams before 4/11.

**Q1 — Trust model:** Enterprise policy gate, vendor attestation, or hybrid?

**Q2 — Hook coverage:** Are there enforcement points not covered by `before_dispatch`,
`before_tool_call`, `before_skill_install`, `before_compaction`, or `credential_resolve`?

**Q3 — Credential lifecycle:** Does the `"plugin"` SecretRefSource cover token-refresh,
rotation, and federated identity scenarios? What is the latency budget for
`credential_resolve` in a tool-call hot path?

---

## External References

| Document                                                                                                                          | Description                                                                                   |
| --------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| [OpenClaw Security WG — Plugin Trust Model (Discussion Draft)](https://bytedance.larkoffice.com/docx/UkoJdaDBko8yVBxHgYlcdnCzn7c) | WG discussion draft on plugin trust model — foundational input for T01                        |
| [RFC: Credential Provider Plugin #59165](https://github.com/openclaw/openclaw/issues/59165)                                       | GitHub RFC for the plugin-extensible credential provider — basis for T06/T07                  |
| [OpenClaw Security and Sandboxing](https://docs.openclaw.ai/gateway/security)                                                     | Official docs: current security model, operator trust boundary, out-of-scope definition       |
| [ClawSentry LT Planning One Pager](https://bytedance.larkoffice.com/wiki/Ggifw3Ziti2RG2k03nVclDTMnKd)                             | Leadership planning one-pager for the ClawSentry security initiative                          |
| [OpenClaw 需求讨论草案](https://bytedance.larkoffice.com/docx/Psv5dwDrOoSIcyx5LZpczAKknoh)                                        | Requirements discussion draft (Chinese) — internal product requirements for security features |

---

## Reference Documents

| Document                                                                                   | Relevance                                                                          |
| ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| [plugin-security-hooks.md](plugin-security-hooks.md)                                       | `before_skill_install` and `before_tool_call` full design — concerns C1–C6 and §10 |
| [proposal-content-inspection-interception.md](proposal-content-inspection-interception.md) | `before_dispatch` hook solution (T10, T11)                                         |
| [proposal-security-skill-scan-suppression.md](proposal-security-skill-scan-suppression.md) | Scan suppression mechanisms M1/M2/M3 (T12)                                         |
| [external-research-landscape.md](external-research-landscape.md)                           | Four 2026 papers: ClawKeeper, taxonomy, HITL, FASA — mapped to tasks               |
| [agentic-enterprise-security-landscape.md](agentic-enterprise-security-landscape.md)       | Okta, Palo Alto AIRS, NVIDIA OpenShell vendor analysis                             |
| [security-wg-panel-agenda.md](security-wg-panel-agenda.md)                                 | WG panel agenda with 9 attacker scenarios and 5 decision areas                     |
| [SECURITY.md](../../../../SECURITY.md)                                                     | Current trust model, operator model, out-of-scope definition                       |
| [src/config/types.secrets.ts](../../../src/config/types.secrets.ts)                        | SecretRef current implementation: `env \| file \| exec` sources                    |
| [src/security/skill-scanner.ts](../../../src/security/skill-scanner.ts)                    | Static scanner: rule definitions, scanning pipeline                                |
| [src/plugins/types.ts:1736-1762](../../../src/plugins/types.ts#L1736-L1762)                | Current `PluginHookName` union — confirms `before_skill_install` absent            |

### External Research Alignment

| Paper finding                            | Task(s)  | Notes                                                    |
| ---------------------------------------- | -------- | -------------------------------------------------------- |
| STAC (Sequential Tool Attack Chains)     | T04      | Cross-call state spec for `before_tool_call`             |
| Memory pollution / soft backdoor         | T25      | `before_tool_call`: `memory_write` provenance gate       |
| Mutable platform ID allowlists (13 CVEs) | Code     | `resolveAllowlistIdentity()` refactor (channel adapters) |
| Exec lexical parsing bypasses (3 CVEs)   | T13      | AST-level scanner rule                                   |
| Docker bind-mount escape                 | T13      | `validate-sandbox-security.ts` code addition             |
| Decentralized trust enforcement          | T01, T16 | Unified policy engine — goal mid/long term               |
| Context provenance (`InputProvenance`)   | T09      | `InputProvenance` on `sessions_send`/context entries     |
