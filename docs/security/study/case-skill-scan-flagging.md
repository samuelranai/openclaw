---
title: "Case Study: Skill Scan Flagging — Root Cause and Developer Guidance"
summary: "End-to-end analysis of the 'Skill flagged — suspicious patterns detected' label on ClawHub: which of the two independent scan systems fires, why results are non-deterministic, what information is actually available today, and what the path forward looks like"
status: draft
---

# Case Study: Skill Scan Flagging

## Problem Statement

A skill developer uploads a security-oriented skill to ClawHub and receives:

> "Skill flagged — suspicious patterns detected"

Running the same upload multiple times produces inconsistent results — the flag appears
in some scan rounds but not others.

The ClawHub backend reply for this specific case:

> "The skill performs several high-risk operations including device fingerprinting via
> `node-machine-id`, spawning a detached background process for remote polling, and
> programmatically modifying the OpenClaw system configuration using shell commands. While
> these actions are documented as part of an installation flow for a security plugin, the use
> of the domain `omini-shield.com` (a potential typo of `omni-shield`) and the automated
> injection of API keys into the local environment are characteristic of supply-chain risks.
> These behaviors are primarily located in `bundle.cjs` and are triggered by instructions
> in `SKILL.md`."

This document explains exactly what fired, why, and what can actually be done about it.

---

## 1. Two Independent Scan Systems

There are **two completely separate scan systems**, and both must pass for the skill to
appear unflagged. Understanding which one is producing a given result is the critical
first step — they have different behaviors, different rules, and a very different
information surface.

|                       | Scan 1: Local Static Scanner                                            | Scan 2: ClawHub Server AI Scan                                                                                                    |
| --------------------- | ----------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Runs where            | Client-side (your machine / CI)                                         | ClawHub backend                                                                                                                   |
| Triggered by          | `openclaw skills install`, `openclaw security audit --deep`             | Skill upload / submission to ClawHub                                                                                              |
| Rule engine           | Static regex rules (deterministic)                                      | AI heuristic / LLM-assisted analysis (non-deterministic)                                                                          |
| Reads `SKILL.md`      | **No** — `.md` not scanned                                              | **Yes** — reads it for behavioral intent                                                                                          |
| Result varies per run | Never                                                                   | Can vary — confidence near threshold                                                                                              |
| Source in repo        | [src/security/skill-scanner.ts](../../../src/security/skill-scanner.ts) | Server-side only; result flows back via `verification.scanStatus` in [src/infra/clawhub.ts:60](../../../src/infra/clawhub.ts#L60) |

**The non-determinism is entirely from Scan 2.** Scan 1 is fully deterministic — the
same `bundle.cjs` will always produce the same local scan result.

---

## 2. Scan 1: Local Static Scanner

### 2.1 Rules and how they fire

The scanner lives in [src/security/skill-scanner.ts](../../../src/security/skill-scanner.ts).
It has two rule sets: `LINE_RULES` (line 147) and `SOURCE_RULES` (line 177).

The rules that fired on this skill's `bundle.cjs`:

| Rule ID                  | Severity     | How it fired                                                                                                                                     |
| ------------------------ | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `dangerous-exec`         | **critical** | `spawn(` on a line + `child_process` appears anywhere in the source ([skill-scanner.ts:148-173](../../../src/security/skill-scanner.ts#L148))    |
| `env-harvesting`         | **critical** | `process.env` in source + `fetch`/`post`/`http.request` also in source ([skill-scanner.ts:198-205](../../../src/security/skill-scanner.ts#L198)) |
| `potential-exfiltration` | warn         | `readFileSync`/`readFile` in source + `fetch`/`post` also present ([skill-scanner.ts:178-184](../../../src/security/skill-scanner.ts#L178))      |

**Important rule mechanics:**

- `dangerous-exec` is a **line rule** — it requires `exec/spawn/execSync` to appear on a
  single line, AND `child_process` to appear anywhere in the full source
  (`requiresContext`). In a bundled `.cjs`, the `child_process` import is typically in
  the bundle preamble, so the context check always passes.

- `env-harvesting` is a **source rule** — it matches if `process.env` exists in the
  file AND any network call pattern exists. In a skill that reads an API key from the
  environment and makes any network request, this fires unconditionally.

- There is **no rule for `node-machine-id`** by package name.
- There is **no rule for domain names** (`omini-shield.com` is invisible to Scan 1).
- `SKILL.md` is **never read** — only `.js/.ts/.mjs/.cjs/.mts/.cts/.jsx/.tsx` extensions
  are in `SCANNABLE_EXTENSIONS` ([skill-scanner.ts:39-48](../../../src/security/skill-scanner.ts#L39)).

### 2.2 How to reproduce locally

```bash
# See exactly what the local scanner finds
openclaw security audit --deep
```

The `collectInstalledSkillsCodeSafetyFindings` function in
[src/security/audit-extra.async.ts:1250](../../../src/security/audit-extra.async.ts#L1250)
runs `scanDirectoryWithSummary` on every installed skill directory and reports findings
under `checkId: "skills.code_safety"`. This gives the exact list of which rules fired
and on which lines.

The result is deterministic: run it ten times, get the same findings every time.

### 2.3 What the local scanner cannot tell you

- It cannot explain why the **server** flagged the skill.
- It cannot detect `node-machine-id` by package identity.
- It cannot read `SKILL.md`.
- It produces no Correlation ID, no `taint_analysis` field, no `entropy_score` —
  those concepts do not exist in the local scanner implementation.

---

## 3. Scan 2: ClawHub Server AI Scan

### 3.1 Where the result lives

The server scan verdict flows back as `verification.scanStatus` inside
`ClawHubPackageDetail` ([src/infra/clawhub.ts:53-62](../../../src/infra/clawhub.ts#L53)):

```typescript
verification?: {
  tier?: string;
  scope?: string;
  summary?: string;      // human-readable explanation — this is what ClawHub replied with
  sourceRepo?: string;
  sourceCommit?: string;
  hasProvenance?: boolean;
  scanStatus?: string;   // the verdict field that drives the UI label
} | null;
```

The `summary` field is the natural-language explanation (the ClawHub reply text in the
problem statement is the `summary` value). The `scanStatus` string drives the
"Skill flagged" badge.

### 3.2 Why results are non-deterministic

The server scan includes an LLM-assisted behavioral analysis pass. The behaviors it
detected in this case:

| Behavior                                       | Detection method                                                     |
| ---------------------------------------------- | -------------------------------------------------------------------- |
| `node-machine-id` = device fingerprinting      | AI recognized the package name semantically — not by regex           |
| Detached background process for remote polling | AI inferred from `spawn` + `detached: true` + polling loop pattern   |
| `omini-shield.com` domain                      | AI flagged as potential typosquatting / supply-chain risk signal     |
| API key injection via shell commands           | AI read both `bundle.cjs` and `SKILL.md` together, understood intent |
| Automated `openclaw config set` modification   | AI inferred from SKILL.md install instructions + code behavior       |

All of these are **semantic** detections — they cannot be expressed as simple regexes.
An LLM-based classifier will produce slightly different confidence scores across runs due
to temperature and context sampling. When a confidence score hovers near the flagging
threshold, the verdict alternates. This is the root cause of the "random" behavior.

### 3.3 What information is actually available from Scan 2

The `summary` field in the ClawHub API response is the only structured explanation
exposed today. There is no:

- Correlation ID retrieval CLI (`clawhub-cli scan-results` does not exist)
- `taint_analysis` or `entropy_score` fields in the API response
- `claw-verify` local command that replicates the server scan
- `// @security-intent:` comment suppression (proposed but not implemented — see
  [proposal-security-skill-scan-suppression.md](proposal-security-skill-scan-suppression.md))

The ClawHub reply text itself is the most detailed information available. It correctly
identified all five behaviors above.

---

## 4. Evaluating the Gemini Guides (Two Rounds)

Two rounds of Gemini-generated developer guidance were produced for this problem.
Both were verified against the actual codebase. The summary: the conceptual framing
improves in round 2, but both rounds contain fabricated tools and fields that do not
exist.

### 4.1 Round 1 — Gemini "Lead Mode" guide

| Gemini claim                                                 | Reality                                                                                                                                      |
| ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Non-determinism from AI heuristic engine                     | **Correct** — Scan 2 is LLM-assisted                                                                                                         |
| LLM classifier compares description vs code behavior         | **Correct** — server scan reads both `SKILL.md` and code                                                                                     |
| Entropy/obfuscation detection exists                         | **Correct** — `obfuscated-code` rule in Scan 1 ([skill-scanner.ts:188-196](../../../src/security/skill-scanner.ts#L188))                     |
| Narrow your permission scopes to reduce flags                | **Partially correct** — Scan 2 considers declared capabilities; no local scope system exists                                                 |
| `clawhub-cli scan-results --id <CORRELATION_ID> --verbose`   | **Hallucinated** — no Correlation ID is exposed, no such CLI command                                                                         |
| `taint_analysis` and `entropy_score` JSON fields             | **Hallucinated** — absent from `ClawHubPackageDetail` and every API response type                                                            |
| `claw-sandbox --trace-net`                                   | **Hallucinated**                                                                                                                             |
| `claw-verify --strict` / `claw-verify --detect-side-effects` | **Hallucinated**                                                                                                                             |
| `// @security-intent: [Reason]` comment suppression          | **Not implemented** — proposed in [proposal-security-skill-scan-suppression.md](proposal-security-skill-scan-suppression.md) but not shipped |
| `manifest.json` with `filesystem.read` capability scopes     | **Wrong format** — OpenClaw skills use `SKILL.md` and `openclaw.plugin.json`, not `manifest.json`                                            |
| `on_load` / `on_init` hooks                                  | **Wrong names** — see [plugin-hooks/README.md](plugin-hooks/README.md) for the actual 26 hook names                                          |

### 4.2 Round 2 — Gemini after being confronted about `claw-verify`

After admitting hallucination, Gemini provided a follow-up guide. Assessment:

| Gemini claim                                                                             | Reality                                                                                                                                                                                                                                          |
| ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| "Hitting the Heuristic & Behavioral Analysis layer" causes non-determinism               | **Correct** mental model — matches Scan 2 behavior                                                                                                                                                                                               |
| SKILL.md semantic gap: vague descriptions cause LLM to "hallucinate risks"               | **Correct and actionable** — the server AI reads `SKILL.md` and compares to code; unclear intent raises confidence scores                                                                                                                        |
| Explicit capability documentation in code comments helps                                 | **Conceptually correct** — not implemented today, but aligns with the suppression proposal                                                                                                                                                       |
| `python3 scripts/package_skill.py --source ./your_skill_dir`                             | **Hallucinated** — only Python script in `scripts/` is `check-composite-action-input-interpolation.py`; no packaging scripts exist for skills                                                                                                    |
| `openclaw skill install security-check` (community skill by "wolffan")                   | **Hallucinated** — no such skill, no such author in the repository or any tracked registry                                                                                                                                                       |
| `openclaw run security-check "audit my local skill at ./path/to/skill"`                  | **Hallucinated** — this command format does not match OpenClaw's actual CLI                                                                                                                                                                      |
| "Correlation ID in the fine print" → open GitHub issue with `security-flag-review` label | **Hallucinated** — no Correlation ID is exposed in the UI, no such GitHub label or issue template exists                                                                                                                                         |
| `taint_analysis` JSON report maintainers can pull                                        | **Hallucinated** — no such internal field exists                                                                                                                                                                                                 |
| `manifest.json version matches SKILL.md exactly`                                         | **Wrong format** — skills do not have a `manifest.json`; version is in `openclaw.plugin.json` when applicable                                                                                                                                    |
| No base64 strings in `SKILL.md`                                                          | **Real rule, wrong file** — `obfuscated-code` rule fires on large `atob()/Buffer.from()` base64 payloads in **code files** ([skill-scanner.ts:192-196](../../../src/security/skill-scanner.ts#L192)), not in `SKILL.md` (which is never scanned) |
| No hardcoded URLs                                                                        | **Partially useful** — the local scanner has no URL rule, but the server AI scanner does flag suspicious domains                                                                                                                                 |
| All outbound network calls declared in metadata                                          | **Correct direction** — not implemented yet; aligns with declared-capabilities proposal                                                                                                                                                          |

### 4.3 Net value of the second round

Round 2 adds one genuinely useful insight not in round 1: **the `SKILL.md` semantic gap
is a primary lever for reducing server AI confidence scores**. Making `SKILL.md` more
explicit about why each high-risk behavior exists (rationale, fixed endpoint, scope of
config changes) reduces the ambiguity the LLM classifier acts on. This is actionable
today without any platform changes.

Everything else in round 2 is either a repeat of round 1's hallucinations or introduces
new ones. The community skill angle is particularly risky — a developer who searches for
`security-check` on ClawHub and installs a skill with that name from an unknown author
is exposed to the very supply-chain risk the scanner was designed to catch.

### 4.4 Round 3 — ChatGPT guide

ChatGPT went further than Gemini by "checking docs and code paths first" and surfacing
a specific architectural claim: that ClawHub uses **VirusTotal Code Insight** and shows
a direct VT report link on the skill detail page.

**Verifying the VirusTotal claim against the source:**

The threat model at [docs/security/THREAT-MODEL-ATLAS.md:127](../THREAT-MODEL-ATLAS.md)
is explicit:

```
│  • VirusTotal scanning (coming soon)
```

The planned-improvements table at line 486 shows:

| Improvement            | Status          |
| ---------------------- | --------------- |
| VirusTotal Integration | **In Progress** |

And recommendation R-001 (line 544) lists "Complete VirusTotal integration" as a P0
immediate action — meaning it is not complete yet.

**ChatGPT's claim that VT is live, links exist on the skill page, and daily rescans run is
false.** VirusTotal integration is a planned feature, not a shipped one. The "direct link
to the full VirusTotal report" the guide instructs developers to open does not exist.

Full assessment of round 3:

| ChatGPT claim                                                                                 | Reality                                                                                                                                                                |
| --------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "ClawHub definitely uses VirusTotal Code Insight"                                             | **False** — VirusTotal integration is "In Progress" / "coming soon" per [THREAT-MODEL-ATLAS.md:127](../THREAT-MODEL-ATLAS.md) and [line 486](../THREAT-MODEL-ATLAS.md) |
| "Skill detail page includes a direct link to the full VirusTotal report"                      | **False** — no such link exists; the feature is not shipped                                                                                                            |
| "Active skills are re-scanned daily"                                                          | **Unverifiable / likely false** — no daily rescan logic exists in the local codebase; may be a future ClawHub backend feature but not documented as live               |
| "Stale label even when VT shows 0 detections" pattern                                         | **Concept is valid** but the specific VT framing is wrong; stale flags are a real concern in the server AI scan, not VT                                                |
| Behavioral categories that trigger scrutiny (credentials, shell exec, Keychain, remote fetch) | **Substantively correct** — these match real Scan 1 rules and Scan 2 AI behavioral patterns                                                                            |
| "openclaw/clawhub" GitHub repo for appeal issues                                              | **Unverified** — repo name not confirmed in codebase; the main repo is `openclaw/openclaw`                                                                             |
| Structured appeal issue template approach                                                     | **Reasonable advice** for any flagged skill, regardless of which scan system                                                                                           |
| "Security purpose summary" paragraph in SKILL.md                                              | **Correct and actionable** — aligns with the semantic gap insight from Gemini round 2                                                                                  |
| List every filesystem write, external domain, secret touched                                  | **Correct and actionable** — directly reduces Scan 2 AI confidence scores                                                                                              |
| Avoid remote bootstrap/install patterns                                                       | **Correct** — `downloadRemoteCodeOrBinaries` is a genuine high-risk signal                                                                                             |

### 4.5 Net value across all three rounds

| Source          | Genuine insights                                                                                                 | Hallucinated / wrong                                                                     |
| --------------- | ---------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| Gemini round 1  | AI heuristics exist; semantic gap matters; entropy detection real                                                | All specific commands and API fields fabricated                                          |
| Gemini round 2  | `SKILL.md` semantic gap is the primary actionable lever                                                          | All tools still fabricated; community skill suggestion is dangerous                      |
| ChatGPT round 3 | Behavioral category taxonomy is accurate; SKILL.md documentation checklist is the best practical advice produced | VirusTotal claim is false and central to the entire guide; daily rescan claim unverified |

**The most actionable content from AI assistants across all three rounds: the SKILL.md
documentation checklist from ChatGPT round 3.** It is correct, grounded in how the
server AI scan actually works (reads both code and `SKILL.md`), and requires no platform
changes. It is incorporated into Section 5.3 of this document.

**The most dangerous content: ChatGPT's VirusTotal framing.** A developer who spends
time searching for the "VT report link on the skill detail page" will find nothing and
lose confidence in the rest of the (largely correct) checklist advice.

---

## 5. What the Developer Can Actually Do Today

### 5.1 Understand which scan is blocking you

Run locally first:

```bash
openclaw security audit --deep
```

This runs Scan 1 deterministically. Read the `checkId: "skills.code_safety"` findings.
Each finding includes the rule ID, the file, and the line number. These are exactly the
findings that will not go away across re-submissions.

### 5.2 Classify the behaviors as intentional vs. incidental

For this specific skill, the Scan 1 findings are:

- `dangerous-exec` (critical) — the detached spawn is the core functionality; this
  cannot be removed
- `env-harvesting` (critical) — API key read from `process.env` + network call; this
  is also core functionality
- `potential-exfiltration` (warn) — file read + network post pattern

These are **true positives from the scanner's perspective** — the behaviors are real, and
the scanner cannot distinguish intended from malicious. The correct long-term fix is a
declared-capability mechanism (see Section 6).

### 5.3 For Scan 2 variance: close the SKILL.md semantic gap

The server AI reads both `bundle.cjs` and `SKILL.md` together. When it sees high-risk
code patterns that `SKILL.md` does not explain, the confidence score rises. When
`SKILL.md` clearly explains the rationale for each flagged behavior, the score drops.
This is the most actionable lever available today — no platform changes required.

Specifically for this skill:

1. **`omini-shield.com`** — if this is a typo of `omni-shield.com`, correct it immediately.
   Typosquatting signals are treated as hard supply-chain risk indicators by any behavioral
   scanner and are unlikely to pass even with perfect documentation.

2. **Detached process + remote polling** — this pattern is structurally identical to a
   C2 beacon. `SKILL.md` should explicitly name: the exact remote endpoint, the polling
   interval, what data is sent, and why the process must be detached. Example:

   ```markdown
   ## Background Process

   The skill spawns a detached Node process (`bundle.cjs daemon`) that polls
   `https://omni-shield.com/api/v1/config` every 5 minutes to check for updated
   detection signatures. No user data is transmitted — only the local agent version
   and a timestamp. The process is detached so it survives shell exit.
   ```

3. **Config modification** — document every `openclaw config set` call in `SKILL.md`
   with the key name, the value format, and why it is needed. Undocumented config
   mutation is a strong supply-chain signal.

4. **API key injection** — if the skill writes `OMNI_SHIELD_API_KEY` to the environment
   or to a config file, say so explicitly: where the key comes from, where it is stored,
   and what access it grants.

5. **`node-machine-id`** — name the package in `SKILL.md` and explain the purpose:
   "Device ID is used as an anonymous analytics dimension to count distinct
   installations — no PII is collected." Without this, the AI infers fingerprinting
   intent.

Reducing semantic ambiguity in `SKILL.md` is what moves the server AI's confidence
score below the flagging threshold. The non-determinism the developer observed is
directly caused by the AI hovering near that threshold — better documentation stabilizes
the result.

### 5.4 Use `verification.summary` as your feedback loop

The `summary` field from the ClawHub API response is the most specific feedback
available. When you receive a flag, the summary text (like the reply quoted in the
problem statement) lists the specific behaviors that triggered the decision. Use that
text to identify which behaviors to either remove or document more clearly.

---

## 6. Platform Gap: No Allowlist or Suppression Mechanism

### 6.1 Current state

There is no mechanism for a developer to say "this behavior is intentional" and have
the scanner accept it. The only escape hatch is `source === "openclaw-bundled"`
([audit-extra.async.ts:1264](../../../src/security/audit-extra.async.ts#L1264)), which
requires the skill to be baked into the gateway binary — not available to third-party
developers.

### 6.2 Proposed fix (not yet implemented)

[proposal-security-skill-scan-suppression.md](proposal-security-skill-scan-suppression.md)
covers three suppression mechanisms. The one most relevant to this case is **declared
capabilities in the skill manifest**:

```typescript
// Proposed addition to skill manifest schema
{
  "security": {
    "declaredCapabilities": [
      "spawnsProcesses",      // explains dangerous-exec
      "readsEnvironment",     // explains env-harvesting
      "modifiesConfig",       // explains openclaw config set calls
      "deviceIdentification"  // explains node-machine-id
    ],
    "rationale": "Security monitoring agent: spawns diagnostic processes and reports findings to the configured SIEM endpoint."
  }
}
```

With declared capabilities, the scanner can apply **capability-aware rules** — for
example, `dangerous-exec` would only escalate to critical if the skill does **not**
declare `spawnsProcesses`. A skill that declares the capability and documents the
rationale would receive a lower-severity advisory finding instead.

### 6.3 What server-side declared capabilities would fix

The `node-machine-id` and `omini-shield.com` detections are purely server-side. The
server AI scanner would need to be trained or prompted to treat declared capabilities
as authoritative context. This is a separate ClawHub backend change from the local
scanner fix.

---

## 7. Summary

| Question                           | Answer                                                                                                                                                                              |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Why does the flag appear?          | Scan 1: `dangerous-exec` + `env-harvesting` critical rules fire on `bundle.cjs`. Scan 2: AI detected fingerprinting, detached polling, config mutation, and a domain typo.          |
| Why are results inconsistent?      | Scan 1 is deterministic — it never varies. The variance is 100% from Scan 2's AI confidence score hovering near the flagging threshold.                                             |
| Can I get a detailed scan log?     | No. `verification.summary` is the most detailed field exposed. There is no Correlation ID retrieval command.                                                                        |
| Can I suppress the local findings? | Not today. The suppression proposal is unimplemented.                                                                                                                               |
| What should I fix immediately?     | (1) Correct `omini-shield.com` typo. (2) Add explicit rationale sections in `SKILL.md` for each flagged behavior: daemon endpoint, config keys set, device ID use, API key storage. |
| What fixes the root cause?         | Declared capabilities in the skill manifest + server-side training to honor them.                                                                                                   |

---

## See Also

- [proposal-security-skill-scan-suppression.md](proposal-security-skill-scan-suppression.md) — three-mechanism suppression design
- [plugin-security-hooks.md](plugin-security-hooks.md) — `before_tool_call` + `requireApproval` for runtime enforcement
- [plugin-hooks/session7-security-analysis.md](plugin-hooks/session7-security-analysis.md) — full plugin attack surface analysis
- [src/security/skill-scanner.ts](../../../src/security/skill-scanner.ts) — all local scan rules
- [src/infra/clawhub.ts:53](../../../src/infra/clawhub.ts#L53) — `ClawHubPackageDetail.verification` shape
