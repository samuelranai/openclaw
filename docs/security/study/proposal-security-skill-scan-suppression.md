---
title: "Proposal: False-Positive Suppression for Security Skills in Static Scanner"
summary: "Root-cause analysis and solution design for legitimate security skills being flagged as suspicious/dangerous by the built-in static scanner — covering scan rule mechanics, source-tier trust, and three suppression mechanisms"
status: draft
---

# Proposal: False-Positive Suppression for Security Skills

## 1. Background

A developer building OpenClaw security skills finds that every upload gets labeled
suspicious or dangerous by the static code scanner, even though the flagged behaviors
(process spawning, file reading combined with network calls, environment variable access)
are the canonical behaviors of a security tool.

**Root problem in one sentence:** the static scanner applies heuristics designed to
detect malicious code, but those heuristics are structurally identical to legitimate
security tool behavior — a security scanner that reads files, spawns processes, and
reports findings to a SIEM is _supposed_ to do all those things.

---

## 2. How the Scanner Works Today

### 2.1 Rules that fire on security tools

[src/security/skill-scanner.ts:147-205](../../../src/security/skill-scanner.ts#L147-L205)
defines two rule sets. The ones that fire on legitimate security skills:

| Rule ID                  | Severity     | Trigger                                          | Why a security tool fires it                               |
| ------------------------ | ------------ | ------------------------------------------------ | ---------------------------------------------------------- |
| `dangerous-exec`         | **critical** | `exec\|spawn\|execSync` + `child_process` import | Runs nmap, yara, osquery, openssl, or any external scanner |
| `env-harvesting`         | **critical** | `process.env` + `fetch\|http.request`            | Reads `SIEM_API_KEY` / `SENTRY_DSN` to report findings     |
| `potential-exfiltration` | warn         | `readFile` + `fetch\|http.request`               | Reads log/config file and sends to audit dashboard         |
| `suspicious-network`     | warn         | WebSocket to non-standard port                   | Connects to local security agent (port 9229, 4444, etc.)   |

A security skill that spawns `nmap`, reads a config file, and sends a finding to a
security API will trigger **two critical** rules and two warn-level rules on its first
scan.

### 2.2 Where the "suspicious" label appears

**During install** — [src/agents/skills-install.ts:58-85](../../../src/agents/skills-install.ts#L58-L85):

```ts
// collectSkillInstallScanWarnings
if (summary.critical > 0) {
  warnings.push(`WARNING: Skill "${skillName}" contains dangerous code patterns: ...`);
} else if (summary.warn > 0) {
  warnings.push(`Skill "${skillName}" has ${summary.warn} suspicious code pattern(s).`);
}
```

These warnings appear in the install output and in `SkillInstallResult.warnings`.

**During security audit** — [src/security/audit-extra.async.ts:1297-1319](../../../src/security/audit-extra.async.ts#L1297-L1319):

```ts
findings.push({
  checkId: "skills.code_safety",
  severity: "critical",
  title: `Skill "${skillName}" contains dangerous code patterns`,
  ...
});
```

The skill appears as a `critical` finding in `openclaw security audit --deep` output,
indistinguishable from a genuinely malicious skill.

### 2.3 No suppression mechanism exists today

There is no config key, no skill manifest field, and no operator allowlist that allows
a finding to be acknowledged and suppressed. The only escape hatch is:

- **`openclaw-bundled` source** — the audit skips skills with `source === "openclaw-bundled"`
  ([audit-extra.async.ts:1264](../../../src/security/audit-extra.async.ts#L1264)). This
  requires the skill to be baked into the gateway binary — not available to third-party
  developers.
- **`openclaw-managed` source** — skills under `~/.openclaw/skills/` get the trusted-source
  status for install warnings, but are **still scanned** in the audit.

### 2.4 `before_skill_install` hook — current status

The study docs reference `before_skill_install` as a recently-shipped hook (PR #56050),
but it is **not present** in the current `PluginHookName` union
([src/plugins/types.ts:1736-1762](../../../src/plugins/types.ts#L1736-L1762)). It
either did not land or was reverted. The proposal below does not depend on it.

---

## 3. Solution Design

Three complementary mechanisms address different actors:

| Mechanism                           | Who controls it  | Scope                           | Code change |
| ----------------------------------- | ---------------- | ------------------------------- | ----------- |
| **M1 — Skill-manifest suppression** | Skill author     | Per-skill, per-rule             | Medium      |
| **M2 — Operator config allowlist**  | Gateway operator | Per-skill or global             | Small       |
| **M3 — Managed-tier placement**     | Skill deployer   | Whole skill (no audit findings) | None        |

A complete solution ships M1 + M2 together. M3 is immediately available as a workaround.

---

## 4. Mechanism 1 — In-Manifest Scan Rule Suppression

The skill author declares which rules fire as false positives and provides a written
justification. The scanner reads this at scan time and skips suppressed rules.

### 4.1 Manifest field

In the skill's `SKILL.md` or `skill.json` frontmatter:

```yaml
---
name: security-port-scanner
description: Runs nmap and reports findings to the security dashboard
security:
  suppressScanRules:
    - dangerous-exec # spawns nmap — intentional
    - env-harvesting # reads SIEM_API_KEY from env to authenticate reports
  suppressionJustification: >
    This skill runs external security tools (nmap, yara) and forwards structured
    findings to an internal SIEM. The env-variable access is read-only (SIEM_API_KEY)
    and is sent only to the configured internal endpoint, not to external services.
---
```

Rules:

- `suppressScanRules` is a list of `ruleId` strings from the known rule set.
- `suppressionJustification` is required when `suppressScanRules` is non-empty (enforced
  at scan time, logged as an audit note if absent).
- Suppressed findings are **downgraded to `info`** (not removed), so they still appear
  in `--deep` audit output — just not as `warn`/`critical`.
- Only the exact listed `ruleId` values are suppressed for that skill. Other rules still
  fire normally.

### 4.2 Scanner changes

The scanner needs to accept a suppression list parameter and apply it:

**[src/security/skill-scanner.ts](../../../src/security/skill-scanner.ts) — `scanSource`:**

```ts
export type SkillScanOptions = {
  includeFiles?: string[];
  maxFiles?: number;
  maxFileBytes?: number;
  suppressRuleIds?: ReadonlySet<string>; // ← new
};

// In scanSource():
findings.push({
  ...finding,
  severity: suppressRuleIds?.has(finding.ruleId) ? "info" : finding.severity,
  suppressed: suppressRuleIds?.has(finding.ruleId) ?? false, // ← mark suppressed
});
```

**[src/agents/skills-install.ts](../../../src/agents/skills-install.ts) — `collectSkillInstallScanWarnings`:**

```ts
// Read suppression list from skill frontmatter
const suppressRuleIds = new Set(entry.skill.metadata?.security?.suppressScanRules ?? []);
const summary = await scanDirectoryWithSummary(skillDir, { suppressRuleIds });
// Warnings now only fire on un-suppressed critical/warn findings
```

**[src/security/audit-extra.async.ts](../../../src/security/audit-extra.async.ts) — `collectInstalledSkillsCodeSafetyFindings`:**

```ts
const suppressRuleIds = new Set(entry.skill.metadata?.security?.suppressScanRules ?? []);
const summary = await getCodeSafetySummary({ dirPath: skillDir, suppressRuleIds, ... });
// If all remaining critical/warn findings are suppressed (severity downgraded to info),
// the skill no longer appears as critical/warn in audit output.
// A separate info-level note is emitted: "N scan rules suppressed by skill manifest; justification: ..."
```

### 4.3 Audit trail

When rules are suppressed, the audit emits an `info`-level finding that shows exactly
what was suppressed and why:

```
INFO  skills.code_safety.suppressed_rules
  Skill "security-port-scanner": 2 scan rule(s) suppressed by manifest
  Rules: dangerous-exec, env-harvesting
  Justification: "This skill runs external security tools..."
```

This preserves auditability: the suppression is recorded, not silently ignored.

---

## 5. Mechanism 2 — Operator Config Allowlist

For enterprise deployments where an operator pre-approves a set of security skills,
a config-level override avoids requiring each skill to be re-authored:

```yaml
# ~/.openclaw/config.yaml  (or operator config)
security:
  scan:
    skillRuleSuppressions:
      "security-port-scanner":
        suppressRuleIds:
          - dangerous-exec
          - env-harvesting
        justification: "Approved by security team 2026-04-05 (ticket SEC-1042)"
      "network-monitor":
        suppressRuleIds:
          - suspicious-network
        justification: "Monitors internal network on non-standard port per architecture doc"
```

This is read in the same scanner path as M1. When both manifest and config suppression
exist for the same rule, they merge (union of suppressed rule IDs).

**Config schema addition** ([src/config/zod-schema.core.ts](../../../src/config/zod-schema.core.ts)):

```ts
security: z.object({
  scan: z.object({
    skillRuleSuppressions: z.record(
      z.string(),  // skill name
      z.object({
        suppressRuleIds: z.array(z.string()),
        justification: z.string().optional(),
      })
    ).optional(),
  }).optional(),
}).optional(),
```

**Operator-only scope:** config-level suppression is controlled by the gateway operator,
not by the skill author. A skill author cannot suppress rules via config — only via
manifest (M1). This preserves the principle that an operator must explicitly approve
elevated-privilege behaviors.

---

## 6. Mechanism 3 — Managed-Tier Placement (Immediate Workaround)

**Available today, no code changes required.**

Skills placed in `~/.openclaw/skills/` get `source = "openclaw-managed"`, which:

1. Suppresses the "non-bundled source" install warning
   ([src/agents/skills-install.ts:447-452](../../../src/agents/skills-install.ts#L447-L452))
2. Still runs the scan in `audit --deep` — the suspicious label persists

This is a partial workaround for the install-time warning only. The audit finding
remains until M1/M2 are implemented.

**Managed skills directory:** `~/.openclaw/skills/<skill-name>/SKILL.md`

```
~/.openclaw/skills/
  security-port-scanner/
    SKILL.md
    scanner.sh        ← the actual security tool invocation
```

---

## 7. Comparison of Approaches

| Approach                          | Suppresses install warning | Suppresses audit finding | Auditable | Available now    |
| --------------------------------- | -------------------------- | ------------------------ | --------- | ---------------- |
| M3 — `openclaw-managed` placement | ✅ Yes                     | ❌ No                    | N/A       | ✅ Yes           |
| M1 — Manifest suppression         | ✅ Yes                     | ✅ Yes (→ info)          | ✅ Yes    | ❌ Needs dev     |
| M2 — Operator config              | ✅ Yes                     | ✅ Yes (→ info)          | ✅ Yes    | ❌ Needs dev     |
| Promote to `openclaw-bundled`     | ✅ Yes                     | ✅ Yes (skipped)         | ❌ No     | Maintainer-only  |
| Disable scanner globally          | ✅ Yes                     | ✅ Yes                   | ❌ No     | ❌ Not supported |

---

## 8. Design Boundaries

### 8.1 What suppression does NOT do

- Does not allow a skill to bypass the scanner entirely.
- Does not remove the finding from the audit output — it downgrades severity to `info`
  and adds an explicit suppressed-rules note.
- Does not allow suppression of rules not in the known rule set (unknown `ruleId` values
  are silently ignored with a `debug` log).
- Does not affect plugin scanning — `scanPackageInstallSourceRuntime` and
  `scanBundleInstallSourceRuntime` are separate paths for plugins (not skills).

### 8.2 Abuse resistance

A malicious skill that adds `suppressScanRules: [dangerous-exec, env-harvesting]` to
its manifest is not meaningfully more dangerous than without it: the rule was advisory
(not blocking) to begin with. The suppression mechanism does not open a new attack
surface — it only affects the display of audit findings, not whether the skill runs.

The real protection against malicious skills is the operator's control over which
skills are loaded (allowlists, managed dirs, exec approval at runtime). Suppression
addresses the _false-positive rate_ for trusted developers, not the trust boundary itself.

### 8.3 Relationship to `before_skill_install` hook

When `before_skill_install` is eventually implemented, it will receive `builtinFindings`
from the scanner. The suppression mechanism described here should be applied _before_
passing findings to the hook, so that the hook sees the post-suppression findings
(severity-downgraded) rather than the raw pre-suppression ones. This way, a security
plugin that escalates findings based on severity won't re-flag findings the skill author
has already acknowledged.

---

## 9. Implementation Checklist

### No-code (immediate)

- [ ] Place security skills under `~/.openclaw/skills/` to suppress install-time warning

### Mechanism 1 — Manifest suppression

- [ ] Define `security.suppressScanRules` and `security.suppressionJustification` fields
      in skill frontmatter schema
- [ ] Update `scanSource` / `scanDirectoryWithSummary` to accept `suppressRuleIds` and
      downgrade matched findings to `info` + set `suppressed: true`
- [ ] Update `collectSkillInstallScanWarnings` to read suppression from skill manifest
- [ ] Update `collectInstalledSkillsCodeSafetyFindings` to read suppression + emit
      `info`-level audit note for suppressed rules
- [ ] Write tests: suppressed rule → `info` severity; non-suppressed rule → unchanged
      severity; unknown ruleId in suppression list → ignored

### Mechanism 2 — Operator config

- [ ] Add `security.scan.skillRuleSuppressions` to config schema
      (`src/config/zod-schema.core.ts` + `src/config/validation.ts`)
- [ ] Update scanner call sites to merge manifest + config suppressions
- [ ] Update config schema docs and help text
- [ ] Write tests: config suppression merges with manifest suppression

### Documentation

- [ ] Document `security.suppressScanRules` in skill authoring guide
- [ ] Document `security.scan.skillRuleSuppressions` in operator config reference
- [ ] Add example: security skill manifest with suppression + justification

---

## 10. References

| File                                                                                                    | Relevance                                                                                     |
| ------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| [src/security/skill-scanner.ts:147-205](../../../src/security/skill-scanner.ts#L147-L205)               | Rule definitions (`LINE_RULES`, `SOURCE_RULES`) — the four rules that fire on security skills |
| [src/agents/skills-install.ts:58-85](../../../src/agents/skills-install.ts#L58-L85)                     | `collectSkillInstallScanWarnings` — install-time suspicious label                             |
| [src/security/audit-extra.async.ts:1250-1324](../../../src/security/audit-extra.async.ts#L1250-L1324)   | `collectInstalledSkillsCodeSafetyFindings` — audit `--deep` suspicious label                  |
| [src/agents/skills/workspace.ts:459-490](../../../src/agents/skills/workspace.ts#L459-L490)             | Source tier assignment: `openclaw-bundled`, `openclaw-managed`, `openclaw-extra`, etc.        |
| [src/agents/skills-install.ts:443-452](../../../src/agents/skills-install.ts#L443-L452)                 | Trusted install sources — `openclaw-managed` suppresses non-bundled install warning           |
| [src/plugins/install-security-scan.runtime.ts](../../../src/plugins/install-security-scan.runtime.ts)   | Plugin-side scan path (separate from skill scan — not the subject of this proposal)           |
| [docs/security/study/plugin-security-hooks.md:293-315](plugin-security-hooks.md#concerns-and-proposals) | C2 concern — `suppressBuiltinFinding` design (precursor to this proposal)                     |
