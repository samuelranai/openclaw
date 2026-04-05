# OpenClaw Security WG — Panel Agenda (1 page)

## Goal

Converge on a shippable recommendation for:

- Who can run security-sensitive plugins (trust model)
- Which hook primitives are needed, with enforcement semantics

## 0) Ground Truth (5 minutes)

- Plugins execute in-process with the same OS privileges as the Gateway (not a sandbox boundary).
- Current security is primarily guardrails: safe install paths/extraction, best-effort scanning, allow/deny controls, tool/exec approvals, and hardened webhook/hook ingress.

## 1) Align on Threat Model (10 minutes)

Agree on the primary attacker scenarios we must handle:

- A1: Malicious “security plugin” that gains high privilege and exfiltrates or subverts enforcement.
- A2: Bypass of enforcement by inducing failures/timeouts/crashes so policy checks fail open.

Decide which deployment target is in scope:

- Consumer safety guardrails
- Enterprise compliance enforcement

## 2) Decision 1 — Advisory vs Enforcement Hooks (15 minutes)

For each hook surface (start with `before_install`, `before_tool_call`):

- Advisory: can report findings; errors/timeouts can be non-fatal.
- Enforcement-capable: can block or pause; requires deterministic behavior and explicit failure defaults.

Output: an agreed classification table.

| Hook             | Advisory? | Enforcement-capable? | Can block? | Can pause for approval? | Failure default (advisory) | Failure default (enforcement) |
| ---------------- | --------- | -------------------- | ---------- | ----------------------- | -------------------------- | ----------------------------- |
| before_install   |           |                      |            |                         |                            |                               |
| before_tool_call |           |                      |            |                         |                            |                               |

## 3) Decision 2 — Who Can Register Privileged Hooks? (20 minutes)

Pick one of these models (or a combination), and define the minimum bar:

- Vendor attestation: signed plugins, verified publisher identity, revocation, and transparency.
- Enterprise policy gate: org-wide allowlist for plugin IDs/publishers allowed to register enforcement hooks.
- Curated registry: higher review bar, combined with signing for updates.

Output: a clear rule: “A plugin may register enforcement hooks only if …”

## 4) Decision 3 — Payload Minimization & Redaction Defaults (15 minutes)

Decide what hook handlers receive by default vs only with explicit privilege:

- Default: derived metadata (hashes, ids, categories) over absolute paths and raw specifiers.
- Redact by default: URL credentials/tokens, absolute filesystem paths, raw stack traces.
- Capability-gated: “read sensitive install payload” and any other high-risk fields.

Output: required fields + redacted-by-default fields + capability gates.

## 5) Reliability & Composition (10 minutes)

Agree on the operational rules needed for enforcement:

- Timeouts per hook and what happens on timeout.
- Deterministic ordering (priority + stable tie-breaker) and how multiple plugins compose.
- Precedence rules (for example: block is terminal; findings accumulate; approval pauses only when not blocked).

Output: a short “execution semantics” section to embed in docs.

## 6) Close — Next Steps (5 minutes)

- Restore and ship install-time hooks with agreed trust model + enforcement semantics.
- Document operator UX: how users and enterprises can verify a plugin is privileged/trusted.
- Identify follow-up hooks (if any), such as network egress policy, post-install verification, or permission escalation signals.
