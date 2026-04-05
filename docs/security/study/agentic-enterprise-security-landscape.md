---
title: "Securing the Agentic Enterprise — Vendor and Open-Source Landscape (2026)"
summary: "Analysis of leading industry vendors (Okta, Palo Alto Networks) and open-source projects (NVIDIA NemoClaw, OpenShell) addressing agentic AI security in 2026, mapped to OpenClaw's architecture and the OWASP Agentic Top 10"
read_when:
  - Evaluating external security tooling for an OpenClaw enterprise deployment
  - Designing a defense-in-depth architecture for agentic workloads
  - Preparing for Security WG discussion on vendor integration strategy
---

# Securing the Agentic Enterprise — Vendor and Open-Source Landscape (2026)

The shift from passive chatbots to agentic AI systems — systems that plan, use tools,
execute code, and act autonomously — has created a new security frontier that existing
controls were not designed to handle. An agentic system like OpenClaw can send messages,
write files, run shell commands, browse the web, and spawn sub-agents. A compromised or
misdirected agent is not a chatbot giving a wrong answer; it is a process with operator-level
OS access acting on bad instructions.

This document analyzes how the leading identity, network, and runtime security vendors have
responded to this threat class as of 2026, maps their capabilities to OpenClaw's internal
architecture, and concludes with a recommended defense-in-depth integration model.

---

## 1. The Threat Landscape — OWASP Agentic Top 10 (2026)

The OWASP Top 10 for Agentic Applications (2026) is the industry-standard reference for
what we are defending against. Developed with 100+ experts, it identifies the most critical
risks for autonomous AI systems:

| #   | Risk                                                                                       | OpenClaw surface                                             |
| --- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------ |
| A1  | **Prompt Injection** — adversarial instructions override agent intent                      | Channel ingress → LLM call (unfiltered body)                 |
| A2  | **Excessive Agency** — agent granted too many tools/permissions                            | Tool policy pipeline; `tools.profile` not restrictive enough |
| A3  | **Insecure Tool Invocation** — tools called without proper validation or approval          | `exec-approval-manager`; `before_tool_call` empty by default |
| A4  | **Memory Poisoning** — persistent context manipulated to embed future instructions         | MEMORY.md + vector DB (no integrity check)                   |
| A5  | **Privilege Escalation** — agent acquires capabilities beyond its intended scope           | `sessions_spawn`; sub-agent inherits parent policy           |
| A6  | **Supply Chain Compromise** — malicious skill/plugin installed in operator-trusted context | `before_skill_install` (shipped, in WG discussion)           |
| A7  | **Data Exfiltration** — sensitive data leaked via tool calls or outbound messages          | `fs_read` + `send_message`; SSRF guard only partial          |
| A8  | **Insecure Agent-to-Agent Communication** — cross-session injection via `sessions_send`    | `InputProvenance` partially shipped; WS path open            |
| A9  | **Denial of Agent Service** — resource exhaustion or context flooding attacks              | No per-sender rate limit; compaction eviction risk           |
| A10 | **Inadequate Audit Trail** — insufficient logging for forensic reconstruction              | Session JSONL is mutable; no tamper-evident log              |

The OWASP framework makes explicit what the external research papers also concluded:
agents mostly **amplify existing vulnerability classes** (injection, privilege escalation,
supply chain) rather than introducing entirely novel ones. The difference is severity —
the blast radius when an agent is exploited is the agent's full tool set.

---

## 2. Identity Plane — Okta for AI Agents

### What Okta addresses

Okta positions itself as the **identity control plane** for the agentic enterprise.
Their core insight is that AI agents must be treated as **first-class Non-Human Identities
(NHIs)** with full lifecycle management — the same governance applied to human accounts and
service accounts.

General availability: **April 30, 2026**.

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Okta Universal Directory                                       │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Human identities  │  Service accounts  │  AI Agents ←new │ │
│  └──────────────────────────────────────────────────────────┘ │
│                                                                │
│  Agent record:                                                 │
│    id, owner (human), purpose, created, last_active           │
│    permissions: [list of OAuth grants]                        │
│    lifecycle_state: active | suspended | decommissioned       │
└───────────────────────────┬─────────────────────────────────── ┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  Agent Gateway  (centralized control plane, MCP-aware)          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  Virtual MCP server                                      │  │
│  │  Aggregates tools from Okta's MCP registry               │  │
│  │  Policy enforcement: what tools can this agent call?     │  │
│  │  Credential vaulting: injects API keys; never exposes    │  │
│  │  Audit logging: every tool call, auth decision, attempt  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                         │                                       │
│                         │  logs → SIEM                         │
└─────────────────────────────────────────────────────────────────┘
```

### Key capabilities

**Shadow Agent Discovery**
Continuously scans OAuth grants across connected SaaS apps (Salesforce, Microsoft Copilot
Studio, Slack, etc.) to surface agents that employees have connected without IT approval.
Provides blast-radius analysis: "if this agent is compromised, what can it access?"

**Credential Vaulting and Rotation**
API keys and OAuth tokens are stored in Okta's vault, not exposed to the agent's logic
layer. The gateway injects credentials at call time and rotates them automatically.
Eliminates long-lived tokens that could be exfiltrated by a compromised agent.

**Universal Kill Switch**
A single action revokes all OAuth tokens and access for a specific agent instance.
Triggers when behavioral deviation is detected ("intent breaking" behavior) or on
security incident.

**Context-Aware Authorization Policies**
Policies evaluate runtime context before authorizing an agent action — not just "is this
agent allowed to call this tool" but "given what it has done in this session, should it
be allowed to call this tool now?"

### Mapping to OpenClaw

| Okta capability              | OpenClaw gap it addresses                                                                                | Integration point                                                   |
| ---------------------------- | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| NHI registration + lifecycle | No agent identity concept in gateway; auth is per-gateway not per-agent                                  | Would register the gateway instance as an NHI in Okta               |
| Shadow agent discovery       | No detection of unauthorized OpenClaw instances connected to SaaS                                        | OAuth grant scanning catches unauthorized connections               |
| Credential vaulting          | API keys in `~/.openclaw/credentials/` plaintext; `config.writeConfigFile` gives plugins full key access | Gateway config points to Okta vault; keys never in local filesystem |
| Universal kill switch        | No cross-instance revocation mechanism                                                                   | Okta revokes; gateway detects revoked token on next auth            |
| Agent Gateway (MCP)          | `before_tool_call` hook is empty by default; no centralized tool governance                              | Okta Gateway mediates all MCP tool calls from OpenClaw              |

**Key limitation:** Okta operates at the **identity and access layer**. It cannot inspect
the content of an agent's reasoning or the parameters of a tool call in real time. Once
access is granted, execution is outside Okta's scope. This is why identity governance alone
is insufficient — it must be combined with runtime inspection.

---

## 3. Network and Runtime Detection — Palo Alto Networks Prisma AIRS 3.0

### What Palo Alto addresses

Palo Alto Networks (PANW) launched Prisma AIRS 3.0 on **March 23, 2026**, extending their
network security platform to the **active execution phase** of the agentic loop. Their
insight: "Many enterprises monitor what AI says; they remain blind to what AI does."

### Architecture — Three lifecycle phases

```
[Phase 1: Visibility]          [Phase 2: Assessment]          [Phase 3: Runtime]
─────────────────────          ──────────────────────         ──────────────────────
Discover agents wherever       Pre-deployment scanning         Real-time enforcement
they operate                   of agent artifacts              at speed of execution

- OAuth grant analysis         - Code scan: excessive          - Tool call monitoring
- MCP server inventory           permissions                   - Memory poisoning detect
- Shadow agent detection       - Skill/hook scan               - Prompt injection block
- Blast-radius mapping         - AI red teaming: simulate      - Cross-agent swarm
                                 context-aware attacks           protection
```

### Agent Runtime Security (in-depth)

**Tool Call Monitoring**
Prisma AIRS continuously monitors which tools an agent uses, what data it accesses, and how
its permissions change over time. Unsanctioned tool calls are blocked before execution.

**Memory Poisoning Detection**
Specifically targets long-term memory manipulation. Inspects RAG data context and tool
schema for evidence of poisoning — attempts to hijack the agent's goal to perform malicious
behavior in a future session.

**Agentic Gateway**
A central control plane (limited preview as of March 2026) for enforcing agent runtime and
identity security, with governance and observability across the agent fleet. Positioned as
the network-layer counterpart to Okta's identity-layer gateway.

**Deterministic Detectors (vs. ML classifiers)**
Rather than running every tool call through broad ML classifiers at 100–500ms latency,
AIRS uses deterministic detectors optimized for structured agent communications — JSON
payloads, tool call schemas. Significantly lower latency, suitable for inline enforcement.

**AI Red Teaming**
Pre-deployment behavioral simulation: generates context-aware attack scenarios and tests
the agent's response before it enters production. Surfaces exploitable vulnerabilities in
the agent's reasoning and tool-access patterns before they are reachable by real attackers.

### Mapping to OpenClaw

| PANW capability               | OpenClaw gap it addresses                                      | Integration point                                      |
| ----------------------------- | -------------------------------------------------------------- | ------------------------------------------------------ |
| Tool call monitoring          | `after_tool_call` is read-only; no runtime behavior monitoring | AIRS observes tool calls via API or network tap        |
| Memory poisoning detection    | MEMORY.md has no integrity check; no write guard               | AIRS inspects RAG/memory writes for poisoning patterns |
| Prompt injection blocking     | No content filter before LLM call                              | AIRS inline filter on channel ingress                  |
| Pre-deployment red teaming    | No behavioral testing of agent configurations                  | AIRS scans `SOUL.md`, tool policies, skill definitions |
| Excessive permission scanning | `tools.profile` is manual config; no automated audit           | AIRS scans agent artifact for over-permission          |

**Key limitation:** Prisma AIRS is a **monitoring and enforcement platform**, not a
design framework. The RSAC 2026 analysis notes it "functions as a policy enforcement engine
for well-defined security rules" — enterprises must define their governance policies first.
It cannot compensate for an agent with no policy to enforce.

---

## 4. Runtime Isolation — NVIDIA NemoClaw and OpenShell

### What NVIDIA addresses

NVIDIA announced NemoClaw and OpenShell at GTC 2026 (**March 2026**). Where Okta and PANW
operate at identity and network layers, NVIDIA operates at the **execution layer** — kernel-
level isolation of what a running agent can actually do, regardless of whether it was
tricked, compromised, or jailbroken.

### NemoClaw

NemoClaw is OpenClaw with NVIDIA's guardrails stack pre-integrated. It provides:

- **NeMo Guardrails** integration: software-defined safety rules for PII blocking, prompt
  injection detection, and topic filtering at the model call level
- **OpenShell runtime** pre-configured for OpenClaw's execution patterns
- **Privacy router**: keeps sensitive context on-device via local models; routes to frontier
  models (Claude, GPT) only when policy explicitly permits
- Deployable on NVIDIA RTX PCs, RTX PRO workstations, DGX Station/Spark

NemoClaw is explicitly described as "OpenClaw with guardrails" — a distribution, not a fork.
Existing OpenClaw configurations can be dropped into NemoClaw with minimal changes.

### OpenShell — Architecture

OpenShell is the security runtime layer. It enforces constraints at the **OS boundary**,
not the application boundary. This is the critical architectural distinction: even if the
agent's prompts are manipulated to request `rm -rf /`, the kernel-level sandbox blocks it.

```
┌─────────────────────────────────────────────────────────────────┐
│  OpenShell Runtime                                              │
│                                                                 │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐   │
│  │  Filesystem    │  │  Network       │  │  Process       │   │
│  │  Isolation     │  │  Control       │  │  Isolation     │   │
│  │                │  │                │  │                │   │
│  │  Linux Landlock│  │  OPA policy    │  │  seccomp BPF   │   │
│  │  LSM           │  │  proxy         │  │  filter        │   │
│  │  Allowed paths │  │  Deny-by-      │  │  Dangerous     │   │
│  │  locked at     │  │  default; all  │  │  syscalls      │   │
│  │  sandbox create│  │  traffic routed│  │  blocked at    │   │
│  │  Kernel-level  │  │  through OPA   │  │  kernel boundary   │
│  │  enforcement   │  │  real-time     │  │                │   │
│  └────────────────┘  └────────────────┘  └────────────────┘   │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Privacy Router                                           │ │
│  │  Local inference for sensitive context (on-device)       │ │
│  │  Frontier model routing only when policy permits         │ │
│  │  Strips caller credentials; injects backend credentials  │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │  Policy Engine (YAML, hot-reloadable)                    │ │
│  │  agent-x:                                                │ │
│  │    allow_binaries: [python, git]                         │ │
│  │    deny_binaries:  [curl, wget, sh, bash]                │ │
│  │    allow_paths:    [/workspace, /tmp/agent-x]            │ │
│  │    deny_paths:     [/etc, ~/.ssh, /var/run]              │ │
│  │    network:        [allow: api.openai.com, deny: *]      │ │
│  │  Hot-reload: policy updates apply to running sandbox     │ │
│  │  without restart                                         │ │
│  └───────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### Four protection domains

| Domain     | Mechanism                                                    | What it prevents                                                     |
| ---------- | ------------------------------------------------------------ | -------------------------------------------------------------------- |
| Filesystem | Linux Landlock LSM — kernel-enforced path allowlist          | Agent reading `~/.ssh`, `/etc/passwd`, credentials outside workspace |
| Network    | OPA policy proxy — deny-by-default, real-time evaluation     | SSRF; exfiltration to unauthorized hosts; C2 callbacks               |
| Process    | seccomp BPF — dangerous syscall filtering at kernel boundary | Privilege escalation; creating arbitrary sockets; container escape   |
| Privacy    | Privacy router — local inference + credential injection      | LLM provider exfiltrating sensitive context; credential leakage      |

### Out-of-process policy enforcement

OpenShell's most important architectural property: **constraints exist in the runtime
environment, not in behavioral prompts**. A jailbroken or prompt-injected agent cannot
override kernel-level Landlock restrictions by asking nicely. This is the architectural
answer to the fundamental weakness of Layer 1 (skill-based) soft enforcement.

### Supported agent runtimes

OpenShell supports unmodified deployment of:

- **OpenClaw** (primary target)
- Anthropic Claude Code
- OpenAI Codex
- OpenCode

No code changes to the agent are required — OpenShell wraps the agent process.

### Mapping to OpenClaw

| OpenShell capability          | OpenClaw gap it addresses                                     | What it adds                                                              |
| ----------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------- |
| Landlock filesystem isolation | `sandbox.mode = off` by default; fallback is host execution   | Kernel-enforced workspace boundary regardless of config                   |
| seccomp BPF process isolation | No syscall filtering; exec approval is application-level only | Dangerous syscalls blocked at kernel; complements `exec-approval-manager` |
| OPA network proxy             | SSRF guard is partial; no deny-by-default for outbound        | All outbound traffic evaluated against policy before leaving              |
| YAML policy engine            | Tool policy pipeline is code; security teams cannot edit it   | Declarative, version-controlled, hot-reloadable by security team          |
| Privacy router                | API keys in `~/.openclaw/credentials/`; plugins can read them | Keys never exposed to agent process; injected at call time                |
| Audit trail                   | Session JSONL is mutable, unsigned                            | Every tool call, policy decision, and routing event in structured log     |

**Key limitation:** OpenShell is **alpha software** (as of March 2026) in "single-player
mode with rough edges." Not production-ready for enterprise deployments at scale. The
Landlock + seccomp approach also requires Linux — macOS deployments use a different (less
complete) isolation model.

---

## 5. Comparative Analysis

### Capability matrix

| Capability                         | Okta                    | Palo Alto AIRS 3.0             | NVIDIA OpenShell          | OpenClaw built-in           |
| ---------------------------------- | ----------------------- | ------------------------------ | ------------------------- | --------------------------- |
| Agent identity lifecycle           | ✅ Core feature         | Partial (CyberArk integration) | ✗                         | ✗                           |
| Shadow agent discovery             | ✅ OAuth grant scan     | ✅ Network discovery           | ✗                         | ✗                           |
| Credential vaulting / rotation     | ✅ Never exposes keys   | ✗                              | ✅ Privacy router         | ✗ (plaintext files)         |
| Universal kill switch / revocation | ✅ Token revocation     | ✗                              | ✗                         | ✗                           |
| Pre-deployment red teaming         | ✗                       | ✅ AI red teaming              | ✗                         | ✗                           |
| Runtime tool call monitoring       | ✗ (identity layer only) | ✅ Real-time                   | ✅ Audit trail            | Partial (`after_tool_call`) |
| Memory / RAG poisoning detection   | ✗                       | ✅ Explicit feature            | ✗                         | ✗                           |
| Prompt injection inline blocking   | ✗                       | ✅                             | Partial (NeMo Guardrails) | ✗                           |
| Kernel-level filesystem isolation  | ✗                       | ✗                              | ✅ Landlock LSM           | Partial (Docker opt-in)     |
| Network egress control             | ✗                       | ✅ Network traffic             | ✅ OPA proxy              | Partial (SSRF guard)        |
| Process isolation (syscall filter) | ✗                       | ✗                              | ✅ seccomp BPF            | ✗                           |
| Declarative policy (non-code)      | ✅ UI-based policies    | ✅ Platform rules              | ✅ YAML                   | ✗ (TypeScript code)         |
| SIEM integration / audit           | ✅ Logs to SIEM         | ✅ XDR integration             | ✅ Structured log         | ✗ (mutable JSONL)           |
| MCP-aware governance               | ✅ Virtual MCP server   | ✅ Agent Gateway               | ✗                         | ✗                           |
| Multi-framework support            | ✅ Vendor-neutral       | ✅                             | ✅ (4 runtimes)           | OpenClaw only               |

### Defense layer alignment

Each vendor addresses a distinct layer of the defense model:

```
┌────────────────────────────────────────────────────────────────────┐
│  GOVERNANCE PLANE                                                  │
│  Okta for AI Agents                                                │
│  "Who is this agent, what can it access, what has it done?"        │
│  Identity lifecycle · credential vaulting · kill switch · SIEM    │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
┌────────────────────────────────▼───────────────────────────────────┐
│  DETECTION AND ENFORCEMENT PLANE                                   │
│  Palo Alto Prisma AIRS 3.0                                         │
│  "Is this agent behaving as intended right now?"                   │
│  Tool call monitoring · memory poisoning · prompt injection block  │
│  Pre-deployment red teaming · supply chain scan                    │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
┌────────────────────────────────▼───────────────────────────────────┐
│  ISOLATION PLANE                                                   │
│  NVIDIA OpenShell + NemoClaw                                       │
│  "Even if the agent is compromised, what can it actually do?"      │
│  Landlock filesystem · OPA network · seccomp process              │
│  YAML policy · privacy router · tamper-evident audit               │
└────────────────────────────────────────────────────────────────────┘
```

### Attack coverage vs. OWASP Agentic Top 10

| OWASP Risk                   | Okta                | PANW AIRS           | OpenShell         | All three |
| ---------------------------- | ------------------- | ------------------- | ----------------- | --------- |
| A1 Prompt Injection          | ✗                   | ✅                  | Partial           | ✅        |
| A2 Excessive Agency          | ✅ Access scope     | ✅ Permission scan  | ✅ Tool allowlist | ✅        |
| A3 Insecure Tool Invocation  | ✅ Gateway gate     | ✅ Runtime block    | ✅ Policy engine  | ✅        |
| A4 Memory Poisoning          | ✗                   | ✅                  | ✗                 | Partial   |
| A5 Privilege Escalation      | ✅ Least-privilege  | ✅                  | ✅ seccomp        | ✅        |
| A6 Supply Chain              | Partial (discovery) | ✅ Pre-deploy scan  | ✗                 | Partial   |
| A7 Data Exfiltration         | Partial (revoke)    | ✅ Network block    | ✅ OPA egress     | ✅        |
| A8 Agent-to-Agent Injection  | ✗                   | ✅ Swarm protection | ✗                 | Partial   |
| A9 DoS / Resource Exhaustion | ✗                   | Partial             | ✗                 | Partial   |
| A10 Inadequate Audit         | ✅ SIEM logs        | ✅ XDR trail        | ✅ Structured log | ✅        |

---

## 6. Integration Architecture for OpenClaw Enterprise Deployments

Combining all three vendor layers with OpenClaw's own controls produces a defense-in-depth
architecture addressing all OWASP Agentic Top 10 risks.

### Reference deployment

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  ENTERPRISE IDENTITY PLANE (Okta)                                          │
│  OpenClaw instance registered as NHI                                       │
│  API keys in Okta vault (never in ~/.openclaw/credentials/)                │
│  Agent Gateway mediates all MCP tool calls                                 │
│  SIEM receives: every tool call, auth decision, kill-switch event          │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│  DETECTION PLANE (Palo Alto AIRS 3.0)                                      │
│  Pre-deployment: red-team agent config (SOUL.md, tool policy, skills)      │
│  Runtime: inline prompt injection filter on channel ingress                │
│  Runtime: tool call monitoring + memory poisoning detection                │
│  Runtime: MEMORY.md + RAG writes scanned for poisoning patterns            │
│  Feeds: XDR + SIEM                                                         │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│  OPENCLAW GATEWAY (internal controls)                                      │
│                                                                             │
│  allow-from.ts      → immutable platform IDs only (R2 from research)       │
│  before_tool_call   → pattern match + semantic judge (ClawKeeper L2)       │
│  exec-approval-mgr  → semantic cmd interpretation (not lexical)            │
│  before_skill_install → hash verification + before_npm_install             │
│  InputProvenance    → tag all context-window inputs                        │
│  before_compaction  → safety-turn tagging                                  │
└────────────────────────────────┬────────────────────────────────────────────┘
                                 │
┌────────────────────────────────▼────────────────────────────────────────────┐
│  ISOLATION PLANE (NVIDIA OpenShell)                                        │
│  OpenClaw process wrapped in OpenShell sandbox                             │
│                                                                             │
│  Filesystem: Landlock LSM                                                  │
│    allow: [~/.openclaw/agents/, /tmp/workspace/]                           │
│    deny:  [~/.ssh, /etc, /var/run, ~/.openclaw/credentials/]               │
│                                                                             │
│  Network: OPA proxy deny-by-default                                        │
│    allow: [configured LLM API endpoints, allowed channel hosts]            │
│    deny:  [*]                                                               │
│                                                                             │
│  Process: seccomp BPF                                                      │
│    block: ptrace, mount, unshare, keyctl, dangerous socket syscalls        │
│                                                                             │
│  Privacy router: local inference for sensitive turns                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Layered policy evaluation sequence

For every tool call in this architecture:

```
1. Okta Agent Gateway:        Is this agent authorized to call this tool class?
2. Palo Alto AIRS (inline):   Does this tool call match known attack patterns?
3. OpenClaw before_tool_call: Does internal policy allow this tool + params?
4. exec-approval-manager:     If shell tool, is this exact command approved?
5. OpenShell policy engine:   Does this action violate filesystem/network/process policy?
6. OpenShell Landlock/BPF:    Kernel enforces regardless of above answers
```

Layers 1–5 are policy decisions; Layer 6 is an unconditional kernel boundary.

### Implementation priority

| Priority | Action                                                                     | Vendor            | Closes OWASP   |
| -------- | -------------------------------------------------------------------------- | ----------------- | -------------- |
| P0       | Register OpenClaw instance as NHI in Okta; move credentials to vault       | Okta              | A2, A7, A10    |
| P0       | Deploy OpenShell sandbox wrapping OpenClaw process (Linux hosts)           | NVIDIA            | A2, A3, A5, A7 |
| P1       | Enable AIRS runtime monitoring on OpenClaw tool calls and MEMORY.md writes | PANW              | A1, A4, A8     |
| P1       | Populate `before_tool_call` with ClawKeeper L2 handler                     | OpenClaw internal | A1, A3         |
| P1       | Fix mutable platform ID allowlists across all channel adapters             | OpenClaw internal | A1             |
| P2       | Enable AIRS pre-deployment red teaming on agent configs before promotion   | PANW              | A3, A6         |
| P2       | Ship `before_skill_install` with agreed WG enforcement semantics           | OpenClaw internal | A6             |
| P2       | Implement `InputProvenance` tagging on all context inputs                  | OpenClaw internal | A4, A8         |
| P3       | Implement STAC detection in `before_tool_call` (cross-call chain analysis) | OpenClaw internal | A3             |
| P3       | Add semantic command interpretation to `exec-approval-manager`             | OpenClaw internal | A3             |

---

## 7. Structural Observations

### The governance gap is identity, not just code

88% of organizations report suspected or confirmed AI agent security incidents, yet only
22% treat AI agents as independent identity-bearing entities (Okta, 2026). The primary gap
is not missing firewall rules but missing governance: no one knows which agents exist,
what they can access, or who owns them.

### OpenShell as the answer to OpenClaw's sandbox default-off problem

OpenClaw's most significant architectural security gap is that `sandbox.mode = off` by
default — exec tools run directly on the gateway host. The external research taxonomy
(arXiv:2603.27517) found the Docker bind-mount escape to be the only Critical-severity
advisory in a 190-advisory corpus, and it was enabled precisely because the sandbox was
opt-in with no validation of its configuration.

OpenShell addresses this at a structural level: it wraps the entire agent process, not
individual tool calls. The protection applies regardless of whether `sandbox.mode` is
configured.

### The soft-vs-hard enforcement distinction

The NVIDIA architecture explicitly frames this: "Constraints exist in the runtime
environment rather than behavioral prompts, preventing agents from overriding them even
if compromised." NeMo Guardrails (NemoClaw Layer 1) and OpenClaw's `before_tool_call` hook
are soft enforcement — they shape behavior but can be bypassed by a sufficiently
manipulated model. OpenShell's Landlock, seccomp, and OPA proxy are hard enforcement —
the kernel ignores what the model asked.

Defense-in-depth requires **both**: soft controls for the common case (lower latency,
better context awareness), hard controls for the adversarial case (unconditional boundary).

### Vendor lock-in vs. open-source flexibility

| Approach            | Vendor             | Lock-in risk                  | Production readiness                    |
| ------------------- | ------------------ | ----------------------------- | --------------------------------------- |
| Identity governance | Okta               | Medium (proprietary platform) | GA April 2026                           |
| Network/detection   | Palo Alto AIRS 3.0 | High (platform integration)   | GA March 2026                           |
| Runtime isolation   | NVIDIA OpenShell   | Low (open-source, Apache/MIT) | Alpha March 2026                        |
| Internal controls   | OpenClaw hooks     | None                          | Partial (hooks exist; handlers missing) |

For security-sensitive deployments: OpenShell's open-source kernel-level controls are
the most portable and least lock-in-prone. Okta and PANW provide commercial governance
and detection capabilities that complement but do not replace them.

---

## 8. Gaps Not Addressed by Any Vendor

Even with all three vendor layers deployed alongside OpenClaw's internal controls, the
following gaps remain unaddressed:

| Gap                                                 | Why it persists                                                                                                |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Context compression evicting safety constraints     | No vendor monitors compaction content; OpenClaw `before_compaction` hook exists but has no safety-turn tagging |
| Sequential Tool Attack Chains (STAC)                | PANW monitors individual tool calls; no cross-call sequence analysis currently                                 |
| Mutable platform ID in channel allowlists (13 CVEs) | OpenClaw architectural fix required; no vendor can substitute                                                  |
| Exec approval lexical parsing bypasses (3 CVEs)     | OpenClaw code fix required; OpenShell mitigates but does not fix the allowlist bypass                          |
| SOUL.md integrity                                   | No vendor monitors or signs SOUL.md; OpenClaw has no hash verification                                         |

These gaps require OpenClaw source-level changes regardless of which external vendors are
deployed. See [stride-threat-model.md](../gateway/security/study/stride-threat-model.md)
threats 14–21 and [external-research-landscape.md](external-research-landscape.md) for
detailed proposals.

---

## See Also

- [External Research Landscape](external-research-landscape.md) — four 2026 academic papers on OpenClaw security
- [Security WG Panel Agenda](security-wg-panel-agenda.md) — decisions needed on hooks and trust model
- [Plugin Security Hooks](plugin-security-hooks.md) — `before_tool_call` and `before_skill_install` design
- [STRIDE Threat Model](../../gateway/security/study/stride-threat-model.md) — 21 threats with proposals
- [Tool Security Inventory](../../gateway/security/study/tool-security-inventory.md) — per-tool risk classification
- [Architecture Synthesis](../../gateway/security/study/architecture-synthesis.md) — full system architecture

**External references:**

- [Okta for AI Agents](https://www.okta.com/products/govern-ai-agent-identity/)
- [Okta Blueprint for the Secure Agentic Enterprise](https://www.okta.com/newsroom/press-releases/showcase-2026/)
- [Palo Alto Prisma AIRS 3.0](https://www.paloaltonetworks.com/blog/2026/03/prisma-airs-3-0-autonomous-ai/)
- [NVIDIA NemoClaw](https://www.nvidia.com/en-us/ai/nemoclaw/)
- [NVIDIA OpenShell blog](https://developer.nvidia.com/blog/run-autonomous-self-evolving-agents-more-safely-with-nvidia-openshell/)
- [OWASP Top 10 for Agentic Applications 2026](https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/)
