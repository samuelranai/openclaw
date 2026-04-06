---
title: "OpenClaw Security Working Plan — Ongoing and Planning Roadmap"
summary: "Security WG working plan: goals, task roadmap, and workstream details across Plugin Trust Model, Credential Provider RFC, IAM identity model, and Supply Chain security"
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

## Roadmap at a Glance

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

### Goal Mid/Long Term

Larger architectural work and strategic goals. Require earlier stages to land and
involve more complex design or external coordination.

| ID  | Workstream   | Task                                              | Type   | Depends on |
| --- | ------------ | ------------------------------------------------- | ------ | ---------- |
| T16 | Plugin Trust | Watcher process API                               | Design | T15        |
| T17 | Plugin Trust | `before_plugin_install` hook                      | Eng    | T15        |
| T18 | Plugin Trust | Network egress policy hook                        | Eng    | T15        |
| T21 | Credentials  | Vault reference credential provider plugin        | Eng    | T20        |
| T22 | Credentials  | Credential handling security audit                | Audit  | T20        |
| T23 | IAM          | IAM full specification document                   | Design | T08        |
| T24 | IAM          | LDAP / OIDC external IdP integration              | Eng    | T23        |
| T25 | IAM          | RBAC-lite: resource-scoped access control         | Eng    | T23        |
| T26 | IAM          | Session-scoped identity (prevent escalation)      | Eng    | T23        |
| T27 | IAM          | `groupScope`: per-sender session partitioning     | Eng    | T11        |
| T28 | IAM          | Group role model (replace binary `senderIsOwner`) | Eng    | T11        |
| T29 | Supply Chain | Skill and plugin signing scheme                   | Design | T02        |
| T30 | Supply Chain | Provenance receipt (`.scan-receipt.json`)         | Eng    | T02        |
| T31 | Supply Chain | ClawHub registry: scan summaries + signing UI     | Design | T29        |
| T32 | Supply Chain | Supply chain scanner plugin capability            | Eng    | T15, T17   |

---

## Workstream 1 — Plugin Trust Model

### Problem

OpenClaw's plugin system has no privilege tiers. All loaded plugins run in-process
with operator-level trust. This is correct for the current personal-use trust model,
but creates a structural problem for enterprise security deployments:

- A "security enforcer" plugin and a potentially compromised plugin have identical trust.
- There is no enforcement boundary; a lower-trust plugin can disable a security plugin.
- Hook primitives exist (`before_tool_call`, `before_dispatch`) but any plugin can
  register them — there is no mechanism to reserve enforcement semantics for trusted
  plugins only.

Additionally, once a malicious plugin is discovered, the operator has no way to
revoke it without restarting the gateway.

### WG Open Topics

**Topic 0 — Foundational Trust and Security Policy (T01, T05)**

Three specific operational gaps to close regardless of which trust model is chosen:

1. **Runtime disablement** — `openclaw plugins disable <id> --immediate` must terminate
   a plugin's hook registrations in the running process without a gateway restart.
2. **Built-in plugin override** — a config key to quarantine a bundled plugin without
   rebuilding the binary.
3. **Enforcement plugin self-protection** — enforcement-tier plugins must be shielded
   from being disabled by lower-trust plugins or by LLM-triggered tool calls.

These are operational requirements, not vulnerability reports. They belong in the
active roadmap regardless of reporting scope policy.

**Topic 1 — Who Gets to Run Security Plugins? (T01, T03)**

Three candidate trust models are under WG review:

| Model                  | Description                                     | Strength                    | Weakness                                         |
| ---------------------- | ----------------------------------------------- | --------------------------- | ------------------------------------------------ |
| Vendor attestation     | Signed plugin, verified publisher, revocation   | Strong chain of custody     | PKI infrastructure; blocks community tools       |
| Enterprise policy gate | Org-wide allowlist for plugin IDs/publishers    | Operator-controlled; no PKI | Trust is config, not code — can be misconfigured |
| Curated registry       | Higher review bar + signing for ClawHub updates | Balanced                    | Centralized; latency for new security tools      |

WG recommendation under discussion:

- **Initial path:** Enterprise policy gate (`plugins.securityHooks.allowedPluginIds`)
  for operator-controlled deployments; no PKI required.
- **Upgrade path:** OpenClaw-operated signing service (analogous to
  [Windows driver signing](https://learn.microsoft.com/en-us/windows-hardware/drivers/install/driver-signing))
  — CA issues attestations; vendors submit for scanning; revocation is centralized.
- **UX:** Dedicated "Security Vendors" tab on ClawHub rather than mixing security
  plugins with general plugins.
- **Enterprise policy override** applies broadly to all plugins (not just security
  plugins) and works as a complement to, not a replacement for, the attestation model.

Tiered privilege (necessary regardless of signing model):

| Tier        | Qualifies when                          | Hook access                      | Priority |
| ----------- | --------------------------------------- | -------------------------------- | -------- |
| Enforcement | OpenClaw-signed OR operator-allowlisted | All hooks; `block` semantics     | Highest  |
| Verified    | Vendor-signed (not yet OpenClaw CA)     | Modifying hooks; no exec `block` | High     |
| Community   | Unsigned; operator opt-in               | Read-only (void hooks)           | Low      |

**Topic 2 — Do Hook Primitives Cover Security Needs? (T02, T04)**

Current hook coverage analysis (from [plugin-security-hooks.md](plugin-security-hooks.md)):

| Hook                    | Security capability                             | Status                                         |
| ----------------------- | ----------------------------------------------- | ---------------------------------------------- |
| `before_dispatch`       | Block/intercept inbound messages; group context | ✅ Claiming hook; awaited; `isGroup` present   |
| `before_tool_call`      | Block tool execution; require approval          | ✅ Shipped; modifying hook with `block`        |
| `before_skill_install`  | Block skill install before file copy            | ⚠️ Not in `PluginHookName` union — not shipped |
| `before_compaction`     | Observe context before compaction               | ⚠️ Void only — cannot block                    |
| `before_plugin_install` | Block plugin installation                       | ❌ Does not exist                              |
| `llm_input`             | Inspect prompt before LLM call                  | ⚠️ Void — cannot block                         |
| Network egress          | Restrict plugin outbound calls                  | ❌ Does not exist                              |

### Ongoing Short-Term Tasks

**T01 — Trust model decision memo (WG)**
Produce a decision memo selecting the enforcement model, defining the minimum bar for
a plugin to register enforcement hooks, and documenting the failure posture (default
fail-closed once enforcement tier is configured). Input needed from partner security
teams before 4/11.

**T02 — Ship `before_skill_install` hook**
The hook was proposed in PR #56050 but is absent from the current `PluginHookName`
union. Must be implemented as a modifying hook (not claiming), with `block: true`
terminal semantics and a TOCTOU file-hash check after the hook returns.
See [plugin-security-hooks.md](plugin-security-hooks.md) for the full API design.

**T04 — Hook execution semantics spec**
Produce a short specification covering: per-hook timeout behavior, deterministic
priority ordering, precedence rules (`block` is terminal, `requireApproval` pauses
only when not blocked), and cross-call state scope for STAC detection.

**T05 — Runtime plugin disablement + self-protection**
Implement: `openclaw plugins disable <id> --immediate` (terminates registrations in
running process); `plugins.builtinOverrides.<id>: disabled` config key (quarantine
bundled plugins without binary rebuild); protection of enforcement-tier plugins from
being disabled via in-process API or LLM tool calls.

### Planning Near-Term Tasks

**T03 — Enforcement hook tier config key**
Add `plugins.securityHooks.allowedPluginIds: string[]` to the config schema. Plugins
not on this list have their hooks demoted to advisory (void) semantics at registration
time, not at call time. This is the minimum viable enforcement boundary before a full
signing model is available. Depends on T01 trust model decision.

**T15 — Enforcement hook tier shipped and documented**
All decisions from T01 and T03 land in code. Operator UX: `openclaw plugins status
--trust-level` shows each plugin's current trust tier and hook access.

### Goal Mid/Long Term

**T16 — Watcher process API**
A "Watcher" external-process model for security monitoring decoupled from in-process
hooks. Design decisions: observation interface (read-only event stream vs.
bi-directional control), watcher authentication, event scope, and deployment model.

**T17 — `before_plugin_install` hook**
Equivalent gate for plugin (extension) installation. Fires before any files are
copied from a plugin package. A security plugin can block installation of
untrusted/unsigned plugins.
See [plugin-security-hooks.md](plugin-security-hooks.md) §10 for the design
boundaries and abuse resistance analysis applicable to this hook.

**T18 — Network egress policy hook**
A `before_network_egress` hook (or OPA/WASM policy check) that fires before any
outbound HTTP/WebSocket call made by a plugin or tool, enabling data exfiltration
interception at the transport layer.

---

## Workstream 2 — Credential Provider Plugin (RFC #59165)

### Problem

The current `SecretRef` system supports three sources (`env`, `file`, `exec`).
While `exec` can wrap vault CLIs, it cannot handle credential lifecycle scenarios
that enterprise deployments require:

| Scenario                         | Gap                                                         |
| -------------------------------- | ----------------------------------------------------------- |
| Short-lived tokens (OAuth2/OIDC) | No token lifecycle management; no refresh                   |
| Credential rotation events       | Cannot receive push rotation — only poll via `exec`         |
| Federated identity (SPIFFE)      | Not supported; requires custom `exec` wrapper script        |
| Secret access audit trail        | `exec` output is opaque to OpenClaw's audit path            |
| Enterprise credential bus        | Per-gateway config; no org-wide routing to enterprise vault |

See [src/config/types.secrets.ts](../../../src/config/types.secrets.ts) for current
`SecretRefSource` implementation.

### Design Direction

A new `"plugin"` source extends `SecretRefSource`. A plugin registered as a
credential provider exposes a `resolve(event)` callback; the gateway calls it at
credential-fetch time and manages refresh before token expiry.
Full API design in the RFC; detailed TypeScript types to be produced in T07.

Security constraints: only plugins in `plugins.securityHooks.allowedPluginIds` may
register credential providers. Credential values are never written to config files,
logs, or telemetry — in-memory only.

### Ongoing Short-Term Tasks

**T06 — RFC #59165 partner review**
Publish the RFC for partner security team input on three questions: (1) does the
`"plugin"` source model cover your token-refresh and federated-identity scenarios?
(2) what trust tier do you require for a credential-handling plugin? (3) what audit
and observability requirements does your compliance team have for credential access?

### Planning Near-Term Tasks

**T07 — Credential Provider API design**
Document the full API contract: `CredentialResolveEvent` fields, caching model
(per-session / global with TTL), error handling (fail-closed on provider unavailable),
and refresh semantics (`expiresAt` triggers proactive re-resolve). Depends on T06
partner feedback.

**T19 — `"plugin"` SecretRefSource implementation**
Land the new source in `types.secrets.ts`, `zod-schema.core.ts`, the resolver, and
audit checks. Depends on T06 and T07.

**T20 — Plugin SDK `registerCredentialProvider` API**
Add to the public Plugin SDK surface with TypeScript types, error handling contract,
and refresh semantics documentation. Depends on T19.

### Goal Mid/Long Term

**T21 — Vault reference credential provider plugin**
A bundled or closely maintained plugin integrating HashiCorp Vault (and AWS Secrets
Manager as a second backend). Demonstrates the full lifecycle: resolve → refresh →
expiry → audit token.

**T22 — Credential handling security audit**
Audit the full credential resolution path for: in-memory-only residency, timing of
resolution, and plugin isolation (can a credential provider observe secrets belonging
to other plugins?).

---

## Workstream 3 — IAM: Identity and Access Control

### Problem

There is no built-in concept of _who_ is sending a message beyond `senderId` (a
channel-derived string) and a boolean `CommandAuthorized` flag. Enterprise deployments
cannot express policies like "Team A can use tools X and Y; Team B can only use Z" or
"messages from verified employees get exec approval; guests do not."

The IAM model extends — rather than replaces — SECURITY.md's operator-trust model.
It adds policy richness within a single-operator boundary; it does not create OS-level
per-user isolation (still out of scope for shared-gateway multi-tenancy).

### Group-Chat Security Gaps

The current "one-user trusted-operator" model has five structural gaps in group-chat
deployments where multiple senders share a session:

| Gap | Description                                                                                  |
| --- | -------------------------------------------------------------------------------------------- |
| G1  | Group participants share one session key — tool state and history visible across all senders |
| G2  | `senderIsOwner` is binary — no role gradation within groups                                  |
| G3  | `toolsBySender` is static operator config — plugins cannot read or modify it at hook time    |
| G4  | No `groupScope` equivalent — `dmScope` supports per-peer isolation; groups have none         |
| G5  | `before_dispatch` carries `isGroup: boolean` but no stable `groupId` for multi-group policy  |

Even with `groupPolicy: "allowlist"` configured, a security plugin cannot observe
group membership, per-sender session boundaries, or group identity at runtime.
Tasks T11, T27, and T28 address these gaps.

### Ongoing Short-Term Tasks

**T11 — Group context exposed to hook plugins**
Add `groupId` (stable group identifier) and `resolvedSenderPolicy` (the operator's
`toolsBySender` resolved policy for this sender) to `PluginHookBeforeDispatchEvent`.
The `groupId` patch is a 2-line additive change; can ship independently ahead of the
full IAM model. Prerequisite for T27 and T28.

### Planning Near-Term Tasks

**T08 — Sender identity primitives**
Extend inbound message context to carry structured `SenderIdentity` (stable canonical
ID, operator-assigned roles, group memberships, source: `channel | external_idp |
config_allowlist`). Resolution chain: channel-native ID → operator config mapping →
external IdP → fallback to `senderId`.

**T09 — Per-channel identity-based access policy**
Extend `tools.policy` to support identity-based conditions (`sender.roles`,
`sender.channel`). Evaluated in the existing `applyToolPolicyPipeline` via a new
`identity` step. No breaking change to deployments that don't configure identity.
Depends on T08.

**T10 — `before_dispatch` identity enrichment**
Pass resolved `SenderIdentity` into the `before_dispatch` event, enabling
content-inspection plugins to make identity-aware blocking decisions. Depends on T08.

### Goal Mid/Long Term

**T23 — IAM full specification**
Produce a specification covering: identity resolution pipeline, attributes schema,
operator-vs-channel identity assertion, and identity lifecycle (session start,
mid-session role change, session end). Depends on T08 primitives landing.

**T24 — LDAP / OIDC external IdP integration**
A plugin-based or config-based connector mapping channel sender IDs to enterprise
identity attributes. Candidates: OpenLDAP / Active Directory, OIDC userinfo endpoint,
custom `exec`-based resolver. Depends on T23.

**T25 — RBAC-lite: resource-scoped access control**
Apply identity-based access control to session operations (spawn sub-agents, read
history), memory operations (write to `MEMORY.md`, trigger compaction), and plugin
management (install/enable at runtime). Depends on T23.

**T26 — Session-scoped identity**
Bind identity attributes to session creation and enforce that tool policy matches
the identity established at session start. Prevent mid-session identity escalation.
Depends on T23.

**T27 — `groupScope`: per-sender session partitioning**
Add a `groupScope` config option (`shared` / `per-sender` / `per-role`) parallel to
`dmScope`. Per-sender partitioning prevents group-chat session poisoning (one sender's
tool state leaking to another's context). Prerequisite for per-sender RBAC in groups.
Depends on T11.

**T28 — Group role model enrichment**
Replace the binary `senderIsOwner` flag with named operator-assigned roles
(`senderRoles` in group config). Resolved role flows into `SenderIdentity.groups`
for hook plugins and RBAC policy. Default behavior unchanged for existing deployments.
Depends on T11.

---

## Workstream 4 — Supply Chain Security

### Problem

The built-in static scanner applies pattern rules at install time and in `--deep`
audit runs. Key gaps:

| Gap                                 | Description                                                                            |
| ----------------------------------- | -------------------------------------------------------------------------------------- |
| False positives for security skills | `dangerous-exec`, `env-harvesting`, `potential-exfiltration` fire on any security tool |
| No suppression mechanism            | No way to acknowledge a finding as intentional                                         |
| No pre-download hook                | Malicious `postinstall` scripts run before `before_skill_install` can block            |
| No cryptographic provenance         | No signing, no publisher identity, no tamper detection on installed files              |
| `before_skill_install` not shipped  | Proposed in PR #56050; absent from current `PluginHookName` union                      |
| No customizable rule set            | Security teams cannot add YARA / AST rules or suppress by policy                       |

Full analysis and M1/M2/M3 suppression mechanism design in
[proposal-security-skill-scan-suppression.md](proposal-security-skill-scan-suppression.md).

### Ongoing Short-Term Tasks

**T12 — Ship scan suppression M1 + M2**
Implement two suppression mechanisms that unblock security skill developers immediately:

- **M1** (skill author): `security.suppressScanRules` in `SKILL.md` frontmatter —
  scanner downgrades matched findings to `info` severity, records justification.
- **M2** (operator): `security.scan.skillRuleSuppressions` in gateway config — operator
  pre-approves specific skills for specific rules.

Immediate workaround (no code needed): place skills under `~/.openclaw/skills/` (M3).

### Planning Near-Term Tasks

**T13 — AST-level and custom scanner rules**
The current regex scanner cannot distinguish `exec(userInput)` from
`exec(['nmap', '--version'])`. Add optional AST analysis (using `oxc-parser`) and a
rule plugin interface so trusted scanner plugins can add custom rules (YARA, AST checks,
private enterprise rules). Depends on T02.

**T14 — Pre-download scan for npm-sourced skills**
Verify `--ignore-scripts` is consistently applied across all install code paths; add
a tarball-level scan before `npm install` (fetch `.tgz`, scan contents, then run
install only if scan passes). Depends on T02.

### Goal Mid/Long Term

**T29 — Skill and plugin signing scheme**
Design and implement signing for skills and plugins: minisign or SSH-based signing,
content hash of each file + manifest, verification against operator-trusted public
keys at install time. UX: `openclaw skill verify <name>`. Depends on T02.

**T30 — Provenance receipt (`.scan-receipt.json`)**
After `before_skill_install` passes, write a content-addressed receipt (file hashes,
scan summary, hook results, signature). Subsequent `audit --deep` runs verify against
the receipt, detecting post-install tampering. Depends on T02.

**T31 — ClawHub registry: scan summaries + signing UI**
Surface per-skill scan summaries in ClawHub before install; block skills with
unacknowledged `critical` findings; allow authors to upload signed suppression
attestations tied to specific versions. Depends on T29.

**T32 — Supply chain scanner plugin capability**
A first-class "supply chain scanner" plugin role: registered in the enforcement tier,
receives `before_skill_install` and `before_plugin_install` events, consults external
vulnerability databases (OSV, Snyk, npm audit), and enforces org-specific dependency
allowlists. Depends on T15 and T17.

---

## Cross-Cutting Concerns

### Workstream Dependencies

```
T01 (Trust model decision)
  └── T03 (Enforcement tier config)
        └── T15 (Enforcement tier shipped)
              ├── T16 (Watcher API)
              ├── T17 (before_plugin_install)
              ├── T18 (Network egress)
              └── T32 (Supply chain scanner plugin)

T06/T07 (Credentials RFC + design)
  └── T19–T22 (Credential provider implementation)

T02 (before_skill_install)
  ├── T12 (Scan suppression, shared prerequisite)
  └── T13, T14, T29, T30 (Scanner + provenance work)

T08 (Sender identity primitives)
  └── T09, T10 (Policy, dispatch enrichment)
        └── T23–T28 (IAM spec + full implementation)

T11 (Group context for hooks)
  └── T27, T28 (groupScope + group role model)
```

**Critical path:** T01 → T03 → T15 is the bottleneck for both the credential
provider (needs a trusted plugin tier) and the supply chain scanner plugin.
The trust model decision (T01) must come first and is the top WG priority.

### Failure Posture Policy

All enforcement hooks in this roadmap default to **fail-closed** when an enforcement
tier is configured, and **fail-open** (with logged warning) when no enforcement is
configured. This is consistent with SECURITY.md's opt-in framing: enforcement is
off by default, but once opted in, failures must not silently permit blocked operations.

---

## Open Questions for the Security WG

These require input from partner security teams; responses targeted before 4/11.

**Q1 — Trust model selection**
Which enforcement model best fits your enterprise deployment?

- A. Enterprise policy gate: `plugins.securityHooks.allowedPluginIds` in operator
  config (operator-controlled, no PKI, fast deployment)
- B. Vendor attestation: signed plugin with verified publisher key (stronger, requires
  PKI setup)
- C. Hybrid: policy gate for internal/private plugins; attestation required for
  third-party/community plugins

**Q2 — Hook coverage gaps**
Are there security enforcement points NOT covered by the current or proposed hook set?
Specific scenarios where you need to intercept but cannot with `before_dispatch`,
`before_tool_call`, `before_skill_install`, `before_compaction`, or `credential_resolve`?

**Q3 — Credential lifecycle**
Does the proposed `"plugin"` SecretRefSource cover your token-refresh, rotation
notification, and federated identity scenarios? What is the latency budget for a
`credential_resolve` call in a tool-call hot path?

---

## Reference Documents

| Document                                                                                   | Relevance                                                                          |
| ------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------- |
| [plugin-security-hooks.md](plugin-security-hooks.md)                                       | `before_skill_install` and `before_tool_call` full design — concerns C1–C6 and §10 |
| [proposal-content-inspection-interception.md](proposal-content-inspection-interception.md) | `before_dispatch` hook solution (T10, T11)                                         |
| [proposal-security-skill-scan-suppression.md](proposal-security-skill-scan-suppression.md) | Scan suppression mechanisms M1/M2/M3 (T12)                                         |
| [external-research-landscape.md](external-research-landscape.md)                           | Four 2026 papers: ClawKeeper, taxonomy, HITL, FASA — mapped to tasks below         |
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
