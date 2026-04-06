---
title: "OpenClaw Security Working Plan — Near-Term and Mid-Term Roadmap"
summary: "Structured security roadmap covering Plugin Trust Model, Credential Provider RFC, IAM identity model, and supply chain scanner hardening — with near-term (1–2 month) and mid-term (3–6 month) planning tracks"
status: draft
---

# OpenClaw Security Working Plan

## Overview

This document captures the near-term and mid-term security roadmap for OpenClaw,
organized around four converging workstreams:

1. **Plugin Trust Model** — defining who can register enforcement hooks and what
   policies govern privileged plugin behavior
2. **Credential Provider Plugin (RFC #59165)** — a plugin-extensible credential
   resolution layer to address SecretRef limitations in enterprise environments
3. **IAM + Access Control** — a built-in identity model and per-resource access
   control capability for enterprise adoption
4. **Supply Chain Security** — scanner hardening, trust provenance, and skill/plugin
   attestation

The table below summarizes the full timeline; detailed workstream sections follow.

---

## Timeline Summary

| Track            | Near-Term (Months 1–2)                                | Mid-Term (Months 3–6)                               |
| ---------------- | ----------------------------------------------------- | --------------------------------------------------- |
| **Plugin Trust** | Converge on trust model; define enforcement semantics | Ship enforcement hook tier; watcher process API     |
| **Credentials**  | RFC #59165 community input; design API                | Plugin-based credential provider; vault integration |
| **IAM**          | Sender identity primitives; per-channel policy        | Built-in identity model; RBAC-lite; session scoping |
| **Supply Chain** | Scan suppression (proposal); before_skill_install     | Signing, provenance, registry verification          |

---

## Current Trust Model Baseline (from SECURITY.md)

Before planning changes, the current documented trust model defines the baseline:

- OpenClaw is a **personal assistant / single-operator model** — one trusted operator
  per gateway instance.
- Plugins execute **in-process** with gateway-level OS privileges (no sandbox boundary).
- `SECURITY.md` explicitly states: "Installing or enabling a plugin grants it the same
  trust level as local code running on that gateway host."
- Session identifiers (`sessionKey`) are routing controls, not per-user authorization
  boundaries.
- Multi-tenant adversarial isolation (e.g., multiple untrusted users sharing one gateway)
  is **explicitly out of scope** for the current trust model.

**Implication for planning:** IAM and access control work extends, rather than replaces,
this model. The enterprise ask is for _operator-controlled_ identity and policy — not
per-user multi-tenant isolation. The design must be consistent with SECURITY.md's
"one trusted operator" framing while adding richness for enterprise contexts.

---

## Workstream 1 — Plugin Trust Model

### 1.1 Current State

The Security WG has three open topics:

**Topic 0 — Foundational Trust and Security Policy**

The current plugin system has no privilege tiers. All loaded plugins receive the same
in-process operator-level trust. This is documented and correct for the current trust
model, but creates a structural problem when the enterprise requirement is to:

- Allow a plugin to _enforce_ security policy (block tool calls, block installs)
- While preventing a _different_ plugin from subverting that enforcement
- Without requiring all plugins to be reviewed and signed by OpenClaw maintainers

The foundational question: what is the minimum set of constraints that creates a
meaningful enforcement boundary between a "security enforcer" plugin and other plugins?

**Vulnerability report scope and the "Out of Scope" boundary — a gap to close**

`SECURITY.md` correctly states that reports showing malicious behavior from a
trusted-installed/enabled plugin are out of scope, because in-process operator-level
execution is the documented trust model. However, this creates an operational gap
that the core system must address separately from the vulnerability reporting policy:

> _The "Out of Scope" ruling is fair — but the system still needs mechanisms to
> contain the blast radius once a malicious plugin is identified, and to revoke
> its trust promptly without requiring a full gateway restart._

Specific containment requirements not currently addressed:

- **Runtime plugin disablement:** An operator who discovers a plugin is malicious
  today must restart the gateway to unload it. There is no `openclaw plugins disable
<id> --immediate` command that terminates the plugin's hook registrations and tool
  factories in the running process.
- **Built-in plugin exposure:** The gateway ships with bundled plugins
  (`openclaw-bundled` source tier). If a vulnerability is found in a bundled plugin,
  there is no mechanism for an operator to override or quarantine it without rebuilding
  the binary. Built-in plugins need a lighter-weight override path.
- **Enforcement plugin self-protection:** A security enforcement plugin (once the trust
  tier exists) must be protected against being disabled by a lower-trust plugin or an
  in-process LLM-triggered action.

These are operational tasks, not vulnerability reports. They belong in the near-term
roadmap regardless of the reporting scope policy. See Task 1.2.5 below.

**Topic 1 — Who Gets to Run Security Plugins?**

Three candidate trust models identified in prior WG discussions:

| Model                  | Description                                   | Strength                           | Weakness                                            |
| ---------------------- | --------------------------------------------- | ---------------------------------- | --------------------------------------------------- |
| Vendor attestation     | Signed plugin, verified publisher, revocation | Strong chain of custody            | PKI infrastructure; blocks community security tools |
| Enterprise policy gate | Org-wide allowlist for plugin IDs/publishers  | Operator-controlled; no PKI needed | Trust is config, not code — can be misconfigured    |
| Curated registry       | Higher review bar + signing for updates       | Balanced                           | Centralized; latency for new security tools         |

**Possible approaches — notes from review:**

_Vendor attestation:_ The practical path is an **OpenClaw-operated signing service**:
the core runtime trusts only certificates issued (or counter-signed) by OpenClaw's
CA. Individual vendors submit their plugins for scanning and receive a signed
attestation in return. Scanning itself can be a **joint effort** across participating
security vendors — no single vendor has to cover everything. This mirrors the
[Microsoft Windows driver signing model](https://learn.microsoft.com/en-us/windows-hardware/drivers/install/driver-signing),
where Microsoft operates the signing authority but OEMs and IHVs supply the drivers.
Key properties of that model worth adopting:

- The runtime only loads code carrying a valid CA-issued cert (trust is at the CA, not
  the vendor)
- Revocation is centralized — OpenClaw can invalidate a cert without a gateway update
- Vendors can ship iterations without full re-certification if the delta is below a
  policy threshold

_Tiered privilege:_ This is necessary regardless of which signing model is chosen.
Signed security plugins should receive higher-priority hook registration and access
to enforcement-capable hook points. Unsigned or low-trust plugins get advisory-only
(void) hooks. Read-only static analysis could be sandboxed more tightly than runtime
interception hooks. Concretely:

| Tier        | Who qualifies                           | Hook access                               | Priority |
| ----------- | --------------------------------------- | ----------------------------------------- | -------- |
| Enforcement | OpenClaw-signed OR operator-allowlisted | All hook points; `block` semantics active | Highest  |
| Verified    | Vendor-signed (not yet OpenClaw CA)     | Modifying hooks; no `block` on exec       | High     |
| Community   | Unsigned; operator opt-in               | Read-only (void hooks)                    | Low      |

_Curated registry (ClawHub):_ Rather than mixing security plugins with general plugins
in the main listing, **a dedicated "Security Vendors" tab or section on ClawHub** is
the right UX. This creates a visible, lightweight separation without requiring a
separate registry infrastructure — lower cost, clearer signal to operators.

_Enterprise policy override:_ This mechanism (org-wide approved vendor list) applies
broadly to **all plugins**, not just security plugins. It is unlikely to be sufficient
as a standalone trust model for security plugins specifically. It works better as a
complement: the enterprise allowlist governs which plugins can be installed at all;
the signing/attestation model governs which of those installed plugins may register
enforcement hooks.

**Topic 2 — Do Hook Primitives Cover Security Needs?**

From prior analysis (see [plugin-security-hooks.md](plugin-security-hooks.md)):

| Hook                    | Security-relevant capability                      | Gap                                                    |
| ----------------------- | ------------------------------------------------- | ------------------------------------------------------ |
| `before_dispatch`       | Block/intercept inbound messages                  | ✅ Already a claiming hook; awaited; `isGroup` present |
| `before_tool_call`      | Block tool execution; require approval            | ✅ Shipped; modifying hook with `block` support        |
| `before_skill_install`  | Block skill installation before file copy         | ⚠️ Not in current `PluginHookName` union — not shipped |
| `before_compaction`     | Observe/tag safety-turn context before compaction | ⚠️ Shipped as void hook only — cannot block            |
| `before_plugin_install` | Block plugin installation                         | ❌ Does not exist                                      |
| `llm_input`             | Inspect prompt before LLM call                    | ⚠️ Void hook — cannot block                            |
| Network egress          | Restrict where plugins can send data              | ❌ Does not exist                                      |

**Feedback requested from partner security teams:**

- The foundational trust model (operator-controlled vs. signed vs. curated)
- How to establish trust for security plugins without requiring full PKI
- Whether the hook primitives listed above cover what security teams actually need

### 1.2 Near-Term Tasks (Months 1–2)

**Task 1.2.1 — Converge on trust model (WG decision)**

Deliverable: a trust model decision memo covering:

- Selected enforcement model (recommend: Enterprise policy gate + optional vendor
  attestation as an upgrade path)
- Minimum bar for a plugin to register enforcement hooks:
  ```
  A plugin may register enforcement hooks only if:
  (a) its plugin ID appears in the operator's `plugins.securityHooks.allowedPluginIds` list, OR
  (b) it is signed by a publisher whose key is in `plugins.securityHooks.trustedPublishers`
  ```
- Failure posture: enforcement hooks that fail should default to `deny` (fail-closed),
  not `allow` (fail-open) — operator-configurable per hook

**Task 1.2.2 — Ship `before_skill_install` hook**

The hook was proposed in PR #56050 but is absent from the current `PluginHookName` union.
Prior design analysis established the correct API:

```ts
// Handler input
type BeforeSkillInstallEvent = {
  skillName: string;
  sourceDir: string; // absolute path; plugin reads files directly
  source?: string; // "openclaw-bundled" | "workspace" | ...
  builtinFindings: SkillScanFinding[]; // what built-in scanner already found
};

// Handler result
type BeforeSkillInstallResult = {
  block?: boolean;
  blockReason?: string;
  findings?: SkillScanFinding[]; // augment built-in findings
  suppressBuiltinFinding?: string[]; // ruleId list to downgrade to info
};
```

Key implementation notes:

- Must be a **modifying hook** (sequential, accumulated result), not claiming
- `block: true` from any handler is terminal; subsequent handlers do not run
- TOCTOU gap: hash all files in `sourceDir` after hook returns; verify hashes match
  before file copy (C1 from prior analysis)
- Error handling: hook errors non-fatal → installation continues with warning
  (deliberate resilience choice)

**Task 1.2.3 — Enforcement hook registration tier**

Add `plugins.securityHooks.allowedPluginIds: string[]` to config schema. Only plugins
on this list (or signed by trusted publishers, when attestation is enabled) may register
hooks with block/enforcement semantics. Hooks registered by non-allowlisted plugins
are demoted to advisory (void) semantics at registration time, not at call time.

**Task 1.2.4 — WG output: hook execution semantics spec**

Produce a short "execution semantics" document covering:

- Per-hook timeout values and what happens on timeout (block? allow? escalate?)
- Deterministic priority ordering with stable tie-breaker
- Precedence rules: block is terminal; findings accumulate; `requireApproval` pauses
  only when not blocked
- Cross-call state for STAC detection (stored, scoped, reset on session end)

**Task 1.2.5 — Runtime plugin disablement and self-protection**

Implement the three operational containment mechanisms identified in Topic 0:

- **Immediate disablement:** `openclaw plugins disable <id> --immediate` terminates
  the plugin's hook registrations, tool factory, and any active timers in the running
  process — no gateway restart required. The plugin's config entry is marked `disabled`
  to survive restarts.
- **Built-in plugin override path:** A config key (`plugins.builtinOverrides.<id>:
disabled`) allows an operator to quarantine a bundled (`openclaw-bundled`) plugin
  without rebuilding the gateway binary. The gateway checks this map at plugin load
  time and skips the matching bundled plugin.
- **Enforcement plugin self-protection:** Plugins registered in
  `plugins.securityHooks.allowedPluginIds` cannot be disabled by (a) a lower-trust
  plugin calling a disable API or (b) an LLM-generated tool call targeting the plugin
  management endpoint. Disable requests for enforcement plugins must be explicitly
  issued via the operator CLI or config, not via in-process API calls.

### 1.3 Mid-Term Tasks (Months 3–6)

**Task 1.3.1 — Enforcement hook tier, shipped and documented**

All decisions from near-term land in code. Operator UX: how to verify a plugin is
privileged/trusted (CLI `openclaw plugins status --trust-level`).

**Task 1.3.2 — Watcher process API**

A "Watcher" (external process, decoupled from gateway in-process hooks) is a
first-class construct for security monitoring. Design decisions:

- What observation interface does the gateway expose for watchers? (Read-only event
  stream vs. bi-directional control channel)
- Authentication: how does a watcher prove to the gateway it is authorized?
- What events flow to watchers? (All hook events, or a curated audit stream?)
- Who can deploy a watcher — any operator config, or only allowlisted publisher?

**Task 1.3.3 — `before_plugin_install` hook**

Equivalent gate for plugin (extension) installation. Fires before any files are
copied from a plugin package. Receives plugin manifest and extension metadata.
A security plugin can block installation of untrusted/unsigned plugins.
See [plugin-security-hooks.md](plugin-security-hooks.md) §10 for the design
boundaries and abuse resistance analysis applicable to this hook.

**Task 1.3.4 — Network egress policy hook**

A `before_network_egress` hook (or an OPA/WASM policy check) that fires before any
outbound HTTP/WebSocket call made by a plugin or tool. Intercepts data exfiltration
at the transport layer, not just at the tool call level.

---

## Workstream 2 — Credential Provider Plugin (RFC #59165)

### 2.1 Current SecretRef Limitations

The current `SecretRef` system supports three sources:
[src/config/types.secrets.ts](../../../src/config/types.secrets.ts):

```
source: "env"   → environment variable lookup
source: "file"  → JSON or single-value file read
source: "exec"  → subprocess invocation (e.g., `vault kv get ...`)
```

The `exec` source can already integrate with HashiCorp Vault, AWS Secrets Manager,
and Azure Key Vault via CLI wrappers. However, the RFC identifies scenarios where
this is insufficient:

| Scenario                                       | Current behavior                                     | Gap                                          |
| ---------------------------------------------- | ---------------------------------------------------- | -------------------------------------------- |
| Short-lived tokens (OAuth2 / OIDC)             | `exec` calls CLI each time                           | No token lifecycle management; no refresh    |
| Credential rotation notifications              | Cannot receive push rotation events                  | Plugin must poll via `exec`                  |
| Federated identity (workload identity, SPIFFE) | Not supported by any source                          | `exec` workaround requires custom script     |
| Secret access audit trail                      | No structured audit from OpenClaw side               | `exec` output is opaque                      |
| Plugin-provided credential UI                  | No onboarding wizard integration                     | Plugin cannot register credential setup flow |
| Centralized enterprise credential bus          | Cannot route `env`/`file`/`exec` to enterprise vault | Config per-gateway, not enterprise-wide      |

### 2.2 Proposed Design: Credential Provider Plugin API

A new `"plugin"` source extends `SecretRefSource`:

```ts
// src/config/types.secrets.ts — addition
type PluginSecretProviderConfig = {
  source: "plugin";
  pluginId: string; // which plugin provides this credential
  config?: Record<string, unknown>; // provider-specific config (e.g., vault address)
};
```

Plugin SDK surface — new hook:

```ts
// New: credential_resolve hook
type CredentialResolveEvent = {
  providerId: string;
  secretId: string; // what the config's SecretRef.id value is
  context?: {
    // calling context for audit / policy
    sessionKey?: string;
    toolName?: string;
    agentId?: string;
  };
};

type CredentialResolveResult = {
  value: string; // resolved secret value
  expiresAt?: number; // epoch ms; gateway refreshes before expiry
  auditToken?: string; // opaque token for audit trail correlation
};

// Plugin registers as a credential provider
plugin.registerCredentialProvider({
  providerId: "vault",
  resolve: async (event: CredentialResolveEvent): Promise<CredentialResolveResult> => {
    const token = await vaultClient.read(event.secretId);
    return { value: token.data.apiKey, expiresAt: token.lease_duration_ms + Date.now() };
  },
});
```

**Refresh semantics:** if `expiresAt` is set, the gateway proactively calls `resolve`
before expiry. The plugin manages the token refresh lifecycle (OIDC token exchange,
Vault lease renewal, etc.).

**Security constraints:**

- Only plugins with `source: "plugin"` provider registration AND listed in
  `plugins.securityHooks.allowedPluginIds` may register credential providers.
- Credential values are **never** written to gateway config files, logs, or telemetry
  — they remain in-memory only.
- The plugin may return an `auditToken` for external audit trail correlation without
  exposing the secret value.

### 2.3 Near-Term Tasks (Months 1–2)

**Task 2.3.1 — Publish RFC #59165 for partner security team review**

Collect feedback on three questions:

- Does the `"plugin"` source model cover the credential lifecycle scenarios your
  team faces? (Token refresh, rotation, federated identity)
- What trust model do you require for a plugin that handles credentials?
  (Stronger than tool-call plugins — likely requires signing or explicit trust tier)
- What audit/observability requirements does your enterprise compliance team have for
  credential access events?

**Task 2.3.2 — Design: CredentialResolveEvent + caching + expiry semantics**

Document the API contract, caching model (per-session? per-run? global with TTL?),
and error handling (what happens if the provider is unavailable — fail-closed on
credential fetch).

### 2.4 Mid-Term Tasks (Months 3–6)

**Task 2.4.1 — Implement `"plugin"` SecretRefSource**

Land the new source in `types.secrets.ts`, `zod-schema.core.ts`, resolver, and audit
checks.

**Task 2.4.2 — Plugin SDK: `registerCredentialProvider` API**

Add to the public Plugin SDK surface with TypeScript types, error handling contract,
and refresh semantics.

**Task 2.4.3 — Reference implementation: Vault credential provider plugin**

A bundled or closely maintained plugin that integrates HashiCorp Vault (and AWS
Secrets Manager as a second backend). Demonstrates the full lifecycle: resolve →
refresh → expiry → audit.

**Task 2.4.4 — Security audit: credential handling path**

Audit the full credential resolution path for:

- In-memory residency only (no logging, no config persistence)
- Timing of resolution (lazy at first use? eager at gateway start?)
- Plugin isolation: can a plugin's credential provider observe secrets for other plugins?

---

## Workstream 3 — IAM: Built-In Identity Model

### 3.1 Problem Statement

Enterprise customers adopting OpenClaw face a structural gap: there is no built-in
concept of _who_ is sending a message beyond `senderId` (a channel-derived string) and
`CommandAuthorized` (a boolean from sender allowlist matching). This makes it impossible
to express policies like:

- "Team A can use tools X, Y; Team B can only use tool Z"
- "Users with role `admin` can spawn sub-agents; others cannot"
- "Messages from verified employees get `exec` approval; guest users do not"
- "This channel's messages are trusted; this other channel's are not"

The current model documents this explicitly: "session identifiers are routing controls,
not per-user authorization boundaries." For personal use this is correct. For enterprise
deployment it is a barrier to adoption.

**Group-chat security gaps** — an additional structural gap exists in multi-user
group-chat deployments where multiple senders share a single channel session.
Existing infrastructure (`groupPolicy`, `toolsBySender`) provides static
operator-configured controls, but no runtime mechanism exposes group trust context
to hook plugins:

| Gap                                      | Description                                                                                                               |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| G1: No per-sender session isolation      | Group participants share one session key — tool state and session history are visible across all senders                  |
| G2: Binary `senderIsOwner` flag          | No role gradation within groups; only owner vs. non-owner                                                                 |
| G3: `toolsBySender` invisible to plugins | Operator pre-configures per-sender tool policy; plugins cannot read or modify it at hook time                             |
| G4: No `groupScope` equivalent           | `dmScope` supports `per-peer`/`per-channel-peer` isolation; groups have no equivalent sender partitioning                 |
| G5: `groupId` absent from hook context   | `before_dispatch` carries `isGroup: boolean` but no stable `groupId` — multi-group policy is a 2-line additive patch away |

Even when `groupPolicy: "allowlist"` is configured, a security plugin enforcing
per-sender controls cannot observe group membership, per-sender session boundaries,
or group identity attributes at runtime. Tasks 3.3.4, 3.4.5, and 3.4.6 below
address these gaps.

### 3.2 Design Principles

The IAM model must be _consistent_ with SECURITY.md's operator-trust model, not replace
it:

1. **Operator-controlled, not user-controlled.** Identity attributes are asserted by the
   operator's config or a trusted identity provider — not self-asserted by message senders.
2. **Additive policy, not multi-tenancy.** Enterprise IAM adds policy richness within a
   single-operator trust boundary. It does not create per-user isolation of OS-level
   resources (still out of scope for shared-gateway multi-tenancy).
3. **Progressive adoption.** Existing deployments without IAM config continue to work
   unchanged. IAM features activate only when the operator configures them.

### 3.3 Near-Term Tasks (Months 1–2)

**Task 3.3.1 — Sender identity primitives**

Extend the inbound message context to carry structured identity attributes, resolved
by the operator's identity config:

```ts
// Proposed addition to inbound context
type SenderIdentity = {
  id: string; // stable canonical ID (e.g., email, employee ID)
  roles?: string[]; // operator-assigned roles
  groups?: string[]; // group memberships
  attributes?: Record<string, string>; // arbitrary key-value (department, clearance, etc.)
  source: "channel" | "external_idp" | "config_allowlist";
};
```

Resolution chain: channel-native ID → operator config mapping → external IdP lookup
(LDAP/OIDC) → fallback to channel `senderId`.

**Task 3.3.2 — Per-channel access policy**

Extend `tools.policy` to support identity-based conditions:

```yaml
# openclaw.json example
tools:
  policy:
    - condition:
        sender.roles: [admin]
      profile: dangerous
    - condition:
        sender.roles: [employee]
        channel: telegram
      profile: messaging
    - condition: {} # default
      profile: minimal
```

This is evaluated in the existing `applyToolPolicyPipeline` by adding a new
`identity` step before the `global` step.

**Task 3.3.3 — `before_dispatch` identity enrichment**

Pass resolved `SenderIdentity` into `before_dispatch` event and context, enabling
content-inspection plugins to make identity-aware decisions:

```ts
type PluginHookBeforeDispatchEvent = {
  // ... existing fields ...
  senderIdentity?: SenderIdentity; // ← new
};
```

**Task 3.3.4 — Expose group membership context to hook plugins**

Add `groupId` to `PluginHookBeforeDispatchEvent` (2-line additive patch at
`src/hooks/message-hook-mappers.ts` and `src/auto-reply/reply/dispatch-from-config.ts`)
and surface the resolved `toolsBySender` policy for the current sender so enforcement
hooks can make per-sender decisions within a group:

```ts
type PluginHookBeforeDispatchEvent = {
  // ... existing fields ...
  groupId?: string; // stable group identifier; absent for DMs (G5 fix)
  resolvedSenderPolicy?: {
    // operator-configured policy for this sender in this group
    profile: string;
    allowedTools: string[];
  };
};
```

This is a prerequisite for identity-aware group enforcement plugins. The `groupId`
patch is low-risk and can ship independently ahead of the full IAM model.

### 3.4 Mid-Term Tasks (Months 3–6)

**Task 3.4.1 — Built-in identity model: full specification**

Produce a specification document covering:

- Identity resolution pipeline (channel → config → IdP → fallback)
- Identity attributes schema and required vs. optional fields
- Operator-controlled identity assertion vs. channel-provided identity
- Identity lifecycle (session start, mid-session role change, session end)

**Task 3.4.2 — External IdP integration: LDAP / OIDC**

A plugin-based or config-based IdP connector that maps channel sender IDs to
enterprise identity attributes. Candidates:

- OpenLDAP / Active Directory via LDAP bind
- OIDC userinfo endpoint (for platforms that carry OIDC claims in webhooks)
- Custom `exec`-based resolver (using existing `exec` source pattern)

**Task 3.4.3 — RBAC-lite: resource-scoped access control**

Beyond tool policy, apply identity-based access control to:

- Session operations: who can spawn sub-agents, who can read session history
- Memory operations: who can write to `MEMORY.md`, who can trigger compaction
- Plugin management: who can install/enable plugins at runtime

**Task 3.4.4 — Session-scoped identity**

Bind identity attributes to session start (`session_start` hook receives
`SenderIdentity`) and enforce that the session's tool policy matches the identity
established at session creation. Prevent mid-session identity escalation.

**Task 3.4.5 — Per-sender session partitioning within groups (`groupScope`)**

Add a `groupScope` config option parallel to `dmScope`. Options:

- `shared` (current behavior — all group senders share one session and tool state)
- `per-sender` — each group sender gets an isolated session context (G1 fix)
- `per-role` — senders with the same operator-assigned role share a partition

Per-sender partitioning prevents group-chat session poisoning (one sender's tool
state leaking to another sender's context) and is required for true per-sender
RBAC within groups.

**Task 3.4.6 — Group role model enrichment**

Replace the binary `senderIsOwner` flag with a named role system that integrates
with `SenderIdentity` (G2 fix). Config example:

```yaml
groups:
  telegram:
    config:
      senderRoles:
        admin:
          senderIds: ["@alice", "@bob"]
          tools: dangerous
        member:
          senderIds: [] # everyone else
          tools: minimal
```

Resolved role is passed into `SenderIdentity.groups` for hook plugins and
RBAC policy conditions. Replaces `senderIsOwner` without breaking existing
deployments (operator-only config change; default behavior unchanged).

---

## Workstream 4 — Supply Chain Security

### 4.1 Current State

The built-in static scanner ([src/security/skill-scanner.ts](../../../src/security/skill-scanner.ts))
applies pattern rules to skill/plugin code at install time and during `--deep` audit.
Key gaps (from full analysis in [proposal-security-skill-scan-suppression.md](proposal-security-skill-scan-suppression.md)):

| Gap                                      | Description                                                                                    |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------- |
| False positives for security skills      | `dangerous-exec`, `env-harvesting`, `potential-exfiltration` fire on legitimate security tools |
| No suppression mechanism                 | No way to acknowledge a finding as intentional                                                 |
| No pre-download hook                     | Malicious `postinstall` scripts run before `before_skill_install` can fire                     |
| No cryptographic provenance              | No signing, no publisher identity, no tamper detection on installed files                      |
| `before_skill_install` not shipped       | Proposed in PR #56050 but absent from current `PluginHookName` union                           |
| Audit scan covers `warn`/`critical` only | Security teams cannot customize rule set or add YARA/AST rules                                 |

### 4.2 Near-Term Tasks (Months 1–2)

**Task 4.2.1 — Ship scan suppression (M1 + M2)**

Implement the two suppression mechanisms from
[proposal-security-skill-scan-suppression.md](proposal-security-skill-scan-suppression.md):

- M1: `security.suppressScanRules` in SKILL.md frontmatter (skill author declares
  per-rule justification; scanner downgrades to `info`)
- M2: `security.scan.skillRuleSuppressions` in operator config (operator pre-approves
  specific skills for specific rules)

Immediate unblock for security skill developers.

**Task 4.2.2 — Land `before_skill_install` hook**

See Task 1.2.2 above. Priority: this is the architectural prerequisite for all
plugin-based supply chain checks.

**Task 4.2.3 — Extend scanner: AST-level and custom rules**

The current regex scanner cannot distinguish `exec(userInput)` (dangerous) from
`exec(['nmap', '--version'])` (safe). Add:

- Optional AST analysis for JS/TS files (using `@babel/parser` or `oxc-parser`)
- A rule plugin interface: plugins registered as trusted scanners can add custom
  rules (YARA patterns, AST checks, custom regex)
- Rule categories: built-in (unchanged), community (operator opt-in), private
  (enterprise-only, not published)

**Task 4.2.4 — Pre-download scan for npm-sourced skills**

Current gap: `npm install` runs `postinstall` scripts before `before_skill_install`
can block. Mitigation:

- Pass `--ignore-scripts` flag to all npm installs (already done for node skill
  installer in [skills-install.ts:102-111](../../../src/agents/skills-install.ts#L102-L111))
- Verify `--ignore-scripts` is consistently applied across all install code paths
- Add a tarball-level scan before `npm install`: fetch the `.tgz` from npm registry,
  scan the tarball contents, then run install only if scan passes

### 4.3 Mid-Term Tasks (Months 3–6)

**Task 4.3.1 — Skill and plugin signing**

Design and implement a signing scheme for skills and plugins:

- Signing format: minisign or SSH-based signing (simple, widely available, no PKI)
- What is signed: content hash of each file in the skill directory + skill manifest
- Verification: gateway checks signature against operator-trusted public keys at
  install time
- UX: `openclaw skill verify <name>` reports signer, signature status, and hash

**Task 4.3.2 — Provenance chain: `.scan-receipt.json`**

After `before_skill_install` passes, write a content-addressed receipt:

```json
{
  "skillName": "security-scanner",
  "installTimestamp": 1743801600000,
  "fileHashes": { "scanner.ts": "sha256:abc...", "SKILL.md": "sha256:def..." },
  "scanSummary": { "critical": 0, "warn": 1, "suppressed": ["dangerous-exec"] },
  "hookResults": [{ "pluginId": "corp-security-scanner", "block": false }],
  "signature": "..."
}
```

Subsequent `audit --deep` runs verify current files match the receipt — detecting
post-install tampering (addresses concern C5 from prior analysis).

**Task 4.3.3 — ClawHub / registry integration**

Collaborate with the ClawHub team to:

- Surface per-skill scan summaries in the ClawHub UI before install
- Block skills with unacknowledged `critical` findings from registry listing
- Allow skill authors to upload signed suppression attestations tied to specific
  versions (so the scanner knows "this finding is acknowledged for version 1.2.3")

**Task 4.3.4 — Supply chain scanner plugin capability**

A first-class "supply chain scanner" plugin role:

- Registered in `plugins.securityHooks.allowedPluginIds` as a scanner plugin
- Receives `before_skill_install` and `before_plugin_install` events
- Can consult external vulnerability databases (OSV, Snyk, npm audit) via HTTP
- Can enforce organization-specific dependency allowlists
- Can integrate with internal artifact registries

---

## Cross-Cutting Concerns

### Dependency between workstreams

```
Workstream 1 (Trust Model)
    └── Plugin enforcement tier (Task 1.2.3)
            └── Workstream 2 (Credential Provider)   — needs trusted plugin tier
            └── Workstream 4 supply chain scanner    — needs trusted scanner tier

Workstream 3 (IAM)
    └── SenderIdentity primitives (Task 3.3.1)
            └── before_dispatch enrichment (Task 3.3.3)
            └── RBAC-lite for session/memory ops (Task 3.4.3)
```

**Critical path for near-term:** the plugin enforcement tier (Task 1.2.3) is a
prerequisite for both the Credential Provider plugin and the supply chain scanner
plugin, because both require a trust boundary higher than the default plugin trust.
The trust model decision (Task 1.2.1) must come first.

### Failure posture policy

All enforcement hooks added in this roadmap should default to **fail-closed** for
operators who configure an enforcement trust tier, and **fail-open** (with warning)
for operators who have not configured enforcement. This is consistent with SECURITY.md's
"these capabilities are intentional when enabled" framing: enforcement is opt-in, but
once opted in, failures must not silently allow blocked operations.

### External Research Alignment

The near-term and mid-term work directly addresses findings from the four 2026 papers
synthesized in [external-research-landscape.md](external-research-landscape.md):

| Paper finding                                | Workstream | Task                                                     |
| -------------------------------------------- | ---------- | -------------------------------------------------------- |
| STAC (Sequential Tool Attack Chains)         | WS1        | `before_tool_call` cross-call history API                |
| Memory pollution / soft backdoor             | WS1        | `before_tool_call`: `memory_write` provenance gate       |
| Mutable platform ID allowlists (13 CVEs)     | Code       | `resolveAllowlistIdentity()` refactor (channel adapters) |
| Exec lexical parsing bypasses (3 CVEs)       | WS4        | AST-level scanner rule                                   |
| Docker bind-mount escape                     | WS4        | `validate-sandbox-security.ts` code addition             |
| Decentralized trust enforcement (structural) | WS1        | Unified policy engine consideration (defer to mid-term)  |
| Context provenance (`InputProvenance`)       | WS1        | `InputProvenance` on `sessions_send`/context entries     |

---

## Partner Security Team: Feedback Requested

For the Security WG partner review session, three specific inputs are needed:

**Q1 — Trust model:** Which enforcement model best fits your enterprise deployment?

- A. Enterprise policy gate: `plugins.securityHooks.allowedPluginIds` in operator
  config (operator-controlled, no PKI, fast deployment)
- B. Vendor attestation: signed plugin with verified publisher key (stronger, requires
  PKI setup)
- C. Hybrid: policy gate for internal/private plugins, attestation required for
  third-party/community plugins

**Q2 — Hook coverage:** Are there security enforcement points NOT covered by the
current or proposed hook set? Specific scenarios where you would need to intercept
but cannot with `before_dispatch`, `before_tool_call`, `before_skill_install`,
`before_compaction`, and `credential_resolve`?

**Q3 — Credential lifecycle:** Does the proposed `"plugin"` SecretRefSource cover
the token refresh, rotation notification, and federated identity scenarios you face?
What is the latency budget for a `credential_resolve` call in a tool-call hot path?

---

## Reference Documents

| Document                                                                                   | Relevance                                                                      |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------ |
| [plugin-security-hooks.md](plugin-security-hooks.md)                                       | `before_skill_install` and `before_tool_call` design analysis — concerns C1–C6 |
| [proposal-content-inspection-interception.md](proposal-content-inspection-interception.md) | `before_dispatch` hook solution (Req 1)                                        |
| [proposal-security-skill-scan-suppression.md](proposal-security-skill-scan-suppression.md) | Scan suppression mechanisms M1/M2/M3 (Req 2)                                   |
| [external-research-landscape.md](external-research-landscape.md)                           | Four 2026 papers: ClawKeeper, taxonomy, HITL, FASA                             |
| [agentic-enterprise-security-landscape.md](agentic-enterprise-security-landscape.md)       | Okta, Palo Alto AIRS, NVIDIA OpenShell vendor analysis                         |
| [security-wg-panel-agenda.md](security-wg-panel-agenda.md)                                 | WG panel agenda with 9 attacker scenarios and 5 decision areas                 |
| [SECURITY.md](../../../../SECURITY.md)                                                     | Current trust model, operator model, out-of-scope definition                   |
| [src/config/types.secrets.ts](../../../src/config/types.secrets.ts)                        | SecretRef current implementation: `env \| file \| exec` sources                |
| [src/security/skill-scanner.ts](../../../src/security/skill-scanner.ts)                    | Static scanner: rule definitions, scanning pipeline                            |
| [src/plugins/types.ts:1736-1762](../../../src/plugins/types.ts#L1736-L1762)                | Current `PluginHookName` union — confirms `before_skill_install` absent        |
