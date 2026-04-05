# OpenClaw Security WG — Panel Agenda

## Goal

Converge on a shippable recommendation for:

- Who can run security-sensitive plugins (trust model)
- Which hook primitives are needed, with enforcement semantics
- How to address the structural decentralized-trust problem exposed by external research

---

## Pre-Reading (required before session)

| Document                                                                      | What to take in                                                                      |
| ----------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| [plugin-security-hooks.md](plugin-security-hooks.md)                          | `before_tool_call` and `before_skill_install` design — concerns C1–C6                |
| [external-research-landscape.md](external-research-landscape.md)              | Four 2026 papers; especially Part 2 (190 advisories taxonomy) and Part 5 (synthesis) |
| [stride-threat-model.md](../../gateway/security/study/stride-threat-model.md) | Threats 14–21 (added from external research)                                         |

---

## 0) Ground Truth (5 minutes)

Updated from research findings:

- Plugins execute in-process with the same OS privileges as the Gateway (not a sandbox boundary).
- Current security is primarily guardrails: safe install paths/extraction, best-effort scanning, allow/deny controls, tool/exec approvals, and hardened webhook/hook ingress.
- **New (from arXiv:2603.27517):** The dominant structural pattern is per-layer, per-call-site trust enforcement rather than unified policy boundaries. This makes cross-layer composition attacks (STAC, multi-stage RCE, ClawJacked) systematically resistant to layer-local fixes.
- **New (empirical, from arXiv:2603.10387):** Backend LLM selection alone produces a 66-point variance in defense rate (17% to 83%). Security cannot be LLM-dependent.

---

## 1) Align on Threat Model (15 minutes)

### Original attacker scenarios

- A1: Malicious "security plugin" that gains high privilege and exfiltrates or subverts enforcement.
- A2: Bypass of enforcement by inducing failures/timeouts/crashes so policy checks fail open.

### New scenarios from external research (must decide: in scope or out?)

| Scenario                                                                                     | Source           | Priority                   |
| -------------------------------------------------------------------------------------------- | ---------------- | -------------------------- |
| A3: Sequential Tool Attack Chains (STAC) — benign tools chained to achieve malicious outcome | arXiv:2603.12644 | High                       |
| A4: Context compression evicts safety constraints (token exhaustion attack)                  | arXiv:2603.12644 | High                       |
| A5: Memory pollution — adversarial writes to MEMORY.md or vector DB as soft backdoor         | arXiv:2603.12644 | High                       |
| A6: Exec allowlist bypass via lexical parsing (line-continuation, busybox, GNU long-opt)     | arXiv:2603.27517 | High (3 confirmed CVEs)    |
| A7: Mutable platform IDs in channel allowlists (13 confirmed CVEs)                           | arXiv:2603.27517 | High                       |
| A8: ClawJacked — browser CSRF to steal loopback gateway token                                | arXiv:2603.12644 | Medium                     |
| A9: Docker bind-mount escape from sandbox configuration                                      | arXiv:2603.27517 | Critical (1 confirmed CVE) |

Decide which deployment target is in scope:

- Consumer safety guardrails
- Enterprise compliance enforcement

---

## 2) Decision 1 — Advisory vs Enforcement Hooks (15 minutes)

For each hook surface:

| Hook                    | Advisory? | Enforcement-capable? | Can block? | Can pause for approval? | Failure default (advisory) | Failure default (enforcement) |
| ----------------------- | --------- | -------------------- | ---------- | ----------------------- | -------------------------- | ----------------------------- |
| `before_skill_install`  |           |                      |            |                         |                            |                               |
| `before_plugin_install` |           |                      |            |                         |                            |                               |
| `before_tool_call`      |           |                      |            |                         |                            |                               |
| `before_message_write`  |           |                      |            |                         |                            |                               |
| `before_compaction`     |           |                      |            |                         |                            |                               |
| `after_tool_call`       |           |                      |            |                         |                            |                               |

**New question from research:** Should `before_tool_call` have access to the session's
recent tool call history (for STAC detection)? If yes, this is a new API surface —
decide whether the history is read-only or whether the hook can annotate it.

---

## 3) Decision 2 — Who Can Register Privileged Hooks? (20 minutes)

Pick one of these models (or a combination), and define the minimum bar:

- Vendor attestation: signed plugins, verified publisher identity, revocation, and transparency.
- Enterprise policy gate: org-wide allowlist for plugin IDs/publishers allowed to register enforcement hooks.
- Curated registry: higher review bar, combined with signing for updates.

**New question from research (arXiv:2603.24414 Watcher pattern):**
Should a "Watcher" process (separate from the gateway, decoupled from in-process hooks)
be a first-class construct in OpenClaw's security model? If yes, what API does the gateway
expose for watcher observation, and who can deploy a watcher?

Output: a clear rule: "A plugin may register enforcement hooks only if …"

---

## 4) Decision 3 — Payload Minimization & Redaction Defaults (15 minutes)

Decide what hook handlers receive by default vs only with explicit privilege:

- Default: derived metadata (hashes, ids, categories) over absolute paths and raw specifiers.
- Redact by default: URL credentials/tokens, absolute filesystem paths, raw stack traces.
- Capability-gated: "read sensitive install payload" and any other high-risk fields.

**New question from research:** Should `before_tool_call` receive the session's recent
tool call sequence (for STAC detection), and if so, how many prior calls and with what
redaction?

Output: required fields + redacted-by-default fields + capability gates.

---

## 5) Decision 4 — Context Provenance (new, 15 minutes)

External research independently identifies context provenance as the key missing primitive:

> "Data paths that terminate in the context window were treated as information channels
> rather than as potential instruction channels." — arXiv:2603.27517

Proposed: `InputProvenance` enum on all context-window entries:

```
external_user | inter_session | internal_system | tool_result | skill_content
```

Decide:

- Is provenance a gateway-level primitive or a plugin concern?
- Which operations are gated on provenance (memory writes, compaction, sessions_send)?
- What does the LLM see — raw provenance metadata, annotations, or nothing?

Output: a provenance model spec.

---

## 6) Reliability & Composition (10 minutes)

Agree on operational rules needed for enforcement:

- Timeouts per hook and what happens on timeout.
- Deterministic ordering (priority + stable tie-breaker) and how multiple plugins compose.
- Precedence rules (for example: block is terminal; findings accumulate; approval pauses only when not blocked).
- **New:** STAC detection requires cross-call state — how is this state stored, scoped, and reset?

Output: a short "execution semantics" section to embed in docs.

---

## 7) Structural Fix Decision (new, 10 minutes)

The taxonomy paper's key structural finding:

> "Per-layer, per-call-site trust enforcement rather than unified policy boundaries
> makes cross-layer composition attacks systematically resistant to layer-local remediation."

The four papers collectively point toward one structural recommendation:
a **unified policy engine** above per-component enforcement.

Decide: is this in scope for the current WG, or a separate architectural workstream?

Options:

- A. Defer — fix individual CVE classes (A6, A7, A9) now, unified engine later.
- B. Pursue now — the OPA/WASM proposal as an explicit WG output.
- C. Hybrid — `InputProvenance` + cross-call history as minimal unified primitive; OPA later.

---

## 8) Close — Next Steps (5 minutes)

- Restore and ship install-time hooks with agreed trust model + enforcement semantics.
- Document operator UX: how users and enterprises can verify a plugin is privileged/trusted.
- Identify follow-up hooks: `before_compaction` (safety-turn tagging), `before_plugin_install`, network egress policy, post-install verification.
- Assign owners for: mutable-ID allowlist fix (A7), exec semantic interpretation (A6), Docker bind-mount validation (A9).
- Decide watcher process API design workstream ownership.

---

## Reference: Current Hook Coverage vs Threat Matrix

| Threat                             | Hook that addresses it                   | Status                              |
| ---------------------------------- | ---------------------------------------- | ----------------------------------- |
| Prompt injection (T1)              | `before_tool_call` with semantic judge   | Hook exists; no default handler     |
| SOUL.md tampering (T2)             | None (filesystem integrity)              | Gap                                 |
| Exec lexical bypass (T19)          | `exec-approval-manager` refactor         | Code change, not hook               |
| Memory pollution (T15)             | `before_tool_call` for `memory_write`    | Hook exists; no handler             |
| Context compression eviction (T14) | `before_compaction`                      | Hook exists; no safety-turn tagging |
| STAC (T16)                         | `before_tool_call` with cross-call state | Needs API extension                 |
| Mutable ID allowlist (T18)         | `resolveAllowlistIdentity()` abstraction | Architectural refactor              |
| Docker escape (T20)                | `validate-sandbox-security.ts`           | Code addition                       |
| Supply chain (T5)                  | `before_skill_install`                   | Shipped; in WG discussion           |
| Inter-session provenance (T21)     | `InputProvenance` on `sessions_send`     | Partially shipped                   |
