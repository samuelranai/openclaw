---
title: "STRIDE Threat Model"
summary: "Per-component STRIDE analysis for the OpenClaw Gateway — spoofing, tampering, repudiation, information disclosure, denial of service, and elevation of privilege"
read_when:
  - Evaluating security impact of a change to a gateway component
  - Designing a new feature that touches the message pipeline or tool dispatch
  - Assessing risk posture for a deployment
---

# STRIDE Threat Model

This document applies the STRIDE framework to each major OpenClaw Gateway component, grounded in
the source code surfaces identified in the [Attack Surface Map](attack-surface-map.md). Each entry
names the threat, identifies the specific code surface, assesses severity within OpenClaw's
personal-assistant trust model, and describes the existing control (or its absence).

---

## Framework Quick Reference

| Letter | Threat                 | Question                                              |
| ------ | ---------------------- | ----------------------------------------------------- |
| **S**  | Spoofing               | Can an attacker impersonate a trusted principal?      |
| **T**  | Tampering              | Can an attacker modify data in transit or at rest?    |
| **R**  | Repudiation            | Can an attacker deny having taken an action?          |
| **I**  | Information Disclosure | Can an attacker read data they should not?            |
| **D**  | Denial of Service      | Can an attacker disrupt availability?                 |
| **E**  | Elevation of Privilege | Can an attacker gain capabilities beyond their grant? |

Severity ratings reflect impact **within the personal-assistant trust model**. Threats that require
prior operator-boundary compromise are rated lower because they are explicitly out of scope per the
OpenClaw security policy.

---

## Component 1: Channel Ingress (Message Pipeline)

Sources: `src/channels/`, `src/gateway/chat-sanitize.ts`, `src/gateway/agent-prompt.ts`

### S — Spoofing: Sender Identity Fabrication

**Threat:** An attacker sends a message that falsely claims to be from a trusted sender or the
operator, causing the agent to treat it as owner-level input.

**Surface:** `stripEnvelopeFromMessages` extracts `senderLabel` from message envelope and inbound
metadata. An inbound message that mimics the operator's sender label format could confuse
downstream authorization checks.

**Existing control:** `allow-from.ts` allowlisting gates which senders can reach the agent.
`command-gating.ts` enforces owner-only commands.

**Residual risk:** Medium. Allowlist bypass via alias/display-name spoofing on platforms that don't
validate sender identity cryptographically (for example WhatsApp display names, Telegram usernames
in group contexts).

---

### T — Tampering: Prompt Injection via Message Body

**Threat:** An adversarial message body contains instructions that manipulate the agent's behavior
— overriding SOUL context, redirecting tool calls, or exfiltrating session memory.

**Surface:** `buildAgentMessageFromConversationEntries` assembles the full conversation history
plus current message into the LLM prompt. Message body content is not sanitized for adversarial
instructions. `stripEnvelopeFromMessages` removes structural envelope metadata only.

**Existing control:** None at the message-content level. Sender allowlisting is the upstream gate;
if a permitted sender sends adversarial content (for example a forwarded message, a web-fetched
summary, or a hijacked contact), the content reaches the model unfiltered.

**Residual risk:** High. Prompt injection is the dominant novel threat class for agent systems.
OpenClaw's security policy explicitly scopes prompt-injection-only chains as out of scope for
vulnerability reports, but this does not reduce the operational risk in a deployed gateway.

**Enhancement opportunity (Phase 3):** An injection guard skill that runs an LLM-as-judge
pre-screen on each inbound message before it enters the conversation loop — analogous to a WAF
inline mode.

---

### I — Information Disclosure: Session History Leakage

**Threat:** A permitted sender in a group context (for example a shared Slack channel) reads
session history from another participant's prior interactions.

**Surface:** `buildAgentMessageFromConversationEntries` includes prior turn history in the prompt.
Session scoping (`sessionKey`, `sessionId`) is a routing control, not a per-user isolation
boundary.

**Existing control:** Session-scoped contexts; `tools.profile` restrictions on memory tools.

**Residual risk:** Medium in shared-channel deployments. Expected behavior per the trust model:
multiple senders on one channel share the same delegated tool authority and context.

---

### D — Denial of Service: Token Exhaustion via Message Flood

**Threat:** A permitted sender (or a compromised channel account) floods the agent with messages,
exhausting LLM token budget and gateway resources.

**Surface:** Channel adapters ingest messages without a per-sender rate limit at the message-content
level.

**Existing control:** `auth-rate-limit.ts` covers gateway auth attempts. No explicit per-channel
or per-sender message-rate policy was identified in the surface read.

**Residual risk:** Medium. Allowlist controls reduce the attack surface to permitted senders only.

---

## Component 2: Tool Dispatch (`POST /tools/invoke`)

Source: `src/gateway/tools-invoke-http.ts`, `src/security/dangerous-tools.ts`

### S — Spoofing: Token Theft → Operator Impersonation

**Threat:** An attacker who obtains the gateway bearer token can call `/tools/invoke` as a trusted
operator, invoking any allowed tool.

**Surface:** `authorizeGatewayBearerRequestOrReply` accepts bearer tokens. The token is equivalent
to full operator access.

**Existing control:** Token is stored at `~/.openclaw/credentials/`. Rate limiting on auth
failures. No per-tool scope segmentation.

**Residual risk:** High if the credential is exposed. Mitigated by the personal-assistant model:
single-user deployments have a small credential-exposure surface.

---

### T — Tampering: Tool Policy Bypass

**Threat:** A caller manipulates tool name, session key, or headers to bypass the policy pipeline
and invoke a denied tool.

**Surface:** Tool name comes from `body.tool` (raw string); session key from `body.sessionKey` or
the `resolveMainSessionKey` fallback. Policy resolution in `resolveEffectiveToolPolicy` uses
session key to select agent/group policies.

**Existing control:** `DEFAULT_GATEWAY_HTTP_TOOL_DENY` is applied after policy pipeline as a
static deny regardless of configured policy. `applyToolPolicyPipeline` applies profile, global,
agent, group, and subagent steps in sequence.

**Residual risk:** Low for the statically denied tools. Medium for policy-layer bypasses (for
example injecting a session key that resolves to a broader policy context).

---

### E — Elevation of Privilege: `sessions_spawn` as RCE

**Threat:** If `sessions_spawn` were reachable over HTTP, a remote caller could spawn a sub-agent
with full tool access — effectively arbitrary code execution on the gateway host.

**Surface:** `sessions_spawn` is in `DEFAULT_GATEWAY_HTTP_TOOL_DENY` with comment "spawning agents
remotely is RCE."

**Existing control:** Statically denied on HTTP. Available only over authenticated WebSocket.

**Residual risk:** Low on HTTP. Over WS, available to any authenticated operator — appropriate
within the trust model, but high-impact if the WS session is compromised.

**Enhancement opportunity (Phase 3):** A scoped session token model for `sessions_spawn` so
sub-agents inherit only explicitly delegated capabilities, not the full parent scope.

---

### D — Denial of Service: Tool Execution Timeout / Resource Exhaustion

**Threat:** A caller invokes a long-running or resource-intensive tool (for example `browser`,
`exec`) to exhaust gateway resources or block the tool dispatch queue.

**Surface:** `/tools/invoke` with `exec` or `browser` tools against a large payload.

**Existing control:** `DEFAULT_BODY_BYTES = 2MB` body limit. Exec approval system gates shell
commands. Exec approval checks bind specific command patterns.

**Residual risk:** Medium. A trusted operator can invoke approved long-running commands. DoS via
legitimate-but-heavy tool calls is within the operator trust model.

---

## Component 3: Session Storage

Source: `src/gateway/session-transcript-files.fs.ts`, `src/sessions/`

### T — Tampering: Session Transcript Modification

**Threat:** An attacker with local filesystem access (or a tool that escapes the sandbox) modifies
a session JSONL file. On next agent run, the modified history is re-read into the conversation
context, effectively achieving persistent prompt injection.

**Surface:** JSONL files at `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`. No
integrity verification, no hash, no signature.

**Existing control:** None at the file level. OS filesystem permissions are the only barrier.

**Residual risk:** High if local filesystem access is compromised. Within the personal-assistant
model, local filesystem access is equivalent to operator compromise — but the lack of any integrity
check means a compromise is silent (no alert, no detection path).

**Enhancement opportunity (Phase 3):** Hash-signed session headers or a manifest file with per-line
checksums, enabling tamper detection on read.

---

### I — Information Disclosure: Plaintext Transcript Exposure

**Threat:** Session transcripts, stored as plaintext JSONL, may contain sensitive personal data,
credentials, or private messages. Any process that can read `~/.openclaw/sessions/` reads full
conversation history.

**Surface:** Archive operation (`archiveFileOnDisk`) uses `fs.renameSync` — no encryption.
Cleanup of archived files is TTL-based, not content-based.

**Existing control:** OS file permissions. No encryption at rest.

**Residual risk:** Medium in single-user deployments (expected behavior). Elevated on shared hosts
or if backup software reads the home directory.

---

### R — Repudiation: Missing Audit Trail for Tool Calls

**Threat:** A tool call that causes a significant side effect (for example sending a message,
deleting a file) has no tamper-evident log entry that would allow forensic reconstruction after the
fact.

**Surface:** Session transcripts record conversation history, including tool calls. But the
transcript file is mutable and unsigned — a log entry could be removed or altered.

**Existing control:** Session JSONL is append-only in normal operation. Exec approval manager logs
approvals. `control-plane-audit.ts` records some control plane actions.

**Residual risk:** Medium. Append-only semantics help but are not enforced by the filesystem
directly.

---

## Component 4: SOUL.md — Agent Identity

Source: `docs/reference/templates/SOUL.md`, assembled per LLM call

### T — Tampering: Silent SOUL.md Modification

**Threat:** An attacker who can write to the workspace modifies SOUL.md to inject persistent
instructions into the agent's identity — a system-prompt-level persistent injection.

**SOUL.md itself notes:** "If you change this file, tell the user — it's your soul, and they should
know." This is a social contract, not a technical control.

**Surface:** Plain Markdown file on the local filesystem. Read at each agent run. No hash-signing
or runtime tamper detection.

**Existing control:** SOUL.md modification is an operator action; within the trust model, local
filesystem access = trusted operator. No cryptographic integrity check exists.

**Residual risk:** High for an adversary who achieves local write access (for example via a
sandbox-escaping tool call or a supply-chain-compromised skill). The modification is silent — there
is no alert path in the current code.

**Enhancement opportunity (Phase 3):** A hash-signed SOUL.md manifest (or embedded checksum
header) with runtime tamper detection on load — any change produces a warning event and potentially
pauses the agent run pending operator acknowledgment.

---

### S — Spoofing: Identity Substitution via SOUL Replacement

**Threat:** An attacker replaces SOUL.md with a file that impersonates the operator's intent,
instructing the agent to trust different senders, widen tool permissions, or exfiltrate data.

**Surface:** Same as above — SOUL.md is loaded without integrity check.

**Existing control:** None beyond filesystem permissions.

**Residual risk:** High (same as tampering). Combined with `sessions_spawn`, a modified SOUL could
propagate to sub-agents.

---

## Component 5: Skill Supply Chain

Source: `src/security/skill-scanner.ts`, `src/agents/skills/`

### T — Tampering: Malicious Skill Installation

**Threat:** A user installs a skill from ClawHub or a third-party registry that contains malicious
code. Because skills run in-process with the gateway, the malicious code inherits full operator
trust.

**Surface:** `skill-scanner.ts` performs static pattern-based analysis on installed JS/TS files.
No behavioral sandboxing during analysis or at runtime.

**Existing control:** `SkillScanFinding[]` with severity levels. `plugins.allow` for explicit
allowlisting of trusted plugin IDs. ClawHub security analysis checks skill declarations against
behavior.

**Residual risk:** Medium. Static analysis catches obvious patterns but cannot detect obfuscated or
deferred malicious behavior. Runtime trust is full operator-level.

---

### E — Elevation of Privilege: Skill Declares Excessive Env Var Requirements

**Threat:** A skill's `SKILL.md` frontmatter declares requirements for sensitive env vars (LLM API
keys, messaging credentials). A careless install exposes those variables to skill code.

**Surface:** Skill env var injection at install/run time. Skill code can read any injected env var.

**Existing control:** ClawHub security analysis checks declared requirements against actual skill
behavior. Users see required env vars before install.

**Residual risk:** Medium. Relies on operator review at install time and ClawHub analysis quality.

---

## Component 6: Gateway Exposure (Network)

Source: `src/gateway/server-tailscale.ts`, `src/gateway/server-http.ts`

### S — Spoofing: Trusted Proxy Header Injection

**Threat:** An attacker who can reach the gateway port sends `X-Real-IP` or
`X-Forwarded-For` headers to spoof a trusted IP, bypassing loopback-only restrictions.

**Surface:** `trustedProxies` and `allowRealIpFallback` config options. `getHeader` in
`tools-invoke-http.ts` reads `x-openclaw-message-channel`, `x-openclaw-account-id`,
`x-openclaw-message-to`, `x-openclaw-thread-id` from request headers — these influence policy
resolution.

**Existing control:** `trusted-proxy-auth.ts` validates proxy trust. `gateway.bind=loopback`
(default) prevents non-local connections entirely.

**Residual risk:** Low on default loopback config. Medium if the gateway is exposed behind a
reverse proxy that forwards untrusted headers.

---

### D — Denial of Service: Public Funnel Exposure

**Threat:** A gateway exposed via Tailscale Funnel or a public reverse proxy is reachable by the
internet, enabling unauthenticated auth-brute-force or resource exhaustion.

**Surface:** `server-tailscale.ts` can configure Funnel exposure. `gateway.bind=0.0.0.0` binds all
interfaces.

**Existing control:** `openclaw security audit` flags Funnel and non-loopback bind as critical
findings. Auth rate limiting on all exposed surfaces.

**Residual risk:** High if operator ignores audit findings and exposes to the public internet
without strong auth.

---

## Threat Priority Matrix

| #   | Component            | STRIDE                                             | Severity              | Existing Control                    | Enhancement                           |
| --- | -------------------- | -------------------------------------------------- | --------------------- | ----------------------------------- | ------------------------------------- |
| 1   | Message pipeline     | **T** — Prompt injection                           | High                  | Sender allowlist (upstream only)    | Injection guard skill                 |
| 2   | SOUL.md              | **T/S** — Silent modification                      | High                  | OS file permissions only            | Signed manifest + tamper alert        |
| 3   | Session storage      | **T** — Transcript tampering                       | High                  | OS file permissions only            | Session integrity checksums           |
| 4   | Tool dispatch        | **S** — Token theft → operator impersonation       | High                  | Rate limiting, loopback default     | Scoped tokens per surface             |
| 5   | Skill supply chain   | **T/E** — Malicious skill                          | Medium                | Static scanner, allowlist           | Behavioral sandbox at runtime         |
| 6   | sessions_spawn (WS)  | **E** — Sub-agent RCE                              | Medium                | WS-only, auth-gated                 | Scoped delegation tokens              |
| 7   | Channel ingress      | **S** — Sender spoofing                            | Medium                | Allowlist, command gating           | Cryptographic sender verification     |
| 8   | Session storage      | **I** — Plaintext transcripts                      | Medium                | OS file permissions                 | Encryption at rest                    |
| 9   | Gateway network      | **D** — Public Funnel DoS                          | High if misconfigured | Audit flags it; rate limiting       | Enforce loopback-only default         |
| 10  | Tool dispatch        | **T** — Policy pipeline bypass                     | Low-Medium            | Static deny list applied last       | OPA/WASM policy engine                |
| 11  | Web search / browser | **T** — Indirect prompt injection via tool results | High                  | None — tool results unfiltered      | after_tool_call sanitization hook     |
| 12  | Config merge         | **T** — Prototype pollution                        | Low-Medium            | Zod `.strict()` on most schemas     | Audit all merge paths pre-Zod         |
| 13  | Media server         | **I** — Path traversal                             | Medium                | `fs-safe.ts` (coverage unconfirmed) | Verify canonicalization in media path |

---

## Phase 3 Enhancement Proposals (Summary)

These are derived directly from the gaps identified above. Each maps to the threat it closes.

### 1. Injection Guard Skill (closes threat #1)

An inline skill that runs an LLM-as-judge pre-screen on each inbound message before it enters the
conversation loop. Analogous to a WAF in inline mode. Can classify messages as benign, suspicious,
or injected and either annotate the context or block delivery.

Hook point: between `stripEnvelopeFromMessages` and
`buildAgentMessageFromConversationEntries`.

---

### 2. SOUL.md Integrity Verification (closes threats #2, #3 partially)

Hash-signed SOUL.md (or a `SOUL.md.sha256` manifest file). On agent startup, compute the SHA-256
of the loaded SOUL.md and compare to the stored manifest. If they differ, emit a `SecurityAuditFinding`
with `severity: "critical"` and optionally pause pending operator acknowledgment.

Hook point: SOUL.md load path, before the system prompt is assembled.

---

### 3. Session Transcript Integrity (closes threat #3)

Append a per-line HMAC (keyed to the agent's session key or a local secret) to each JSONL entry on
write. On read-back, verify the HMAC chain. Tampered entries fail verification and are surfaced as
a `SecurityAuditFinding`.

Hook point: `session-transcript-files.fs.ts` read/write paths.

---

### 4. Scoped Sub-Agent Delegation (closes threat #6)

Extend `sessions_spawn` with an explicit capability manifest: the spawning session declares a
`delegatedTools` array. The child session's tool policy is initialized from this list only, not
inherited from the parent's full policy. Add `sandbox: "require"` as the default (currently
`"inherit"`).

Hook point: `server-methods/sessions-spawn.ts` and `session-subagent-reactivation.ts`.

---

### 6. Indirect Injection Guard (closes threat #11)

An `after_tool_call` hook that runs a classifier on tool results from web-search,
browser, and MCP tools before they enter the conversation context. Distinguishes
between adversarial instructions embedded in retrieved content and legitimate data.
Can annotate the result with a `[UNVERIFIED EXTERNAL CONTENT]` label so the model
can weight it appropriately.

Hook point: `after_tool_call` registered hook, specifically for tools with
`source: "external"` classification.

---

### 7. Config Merge Prototype Pollution Audit (closes threat #12)

Audit all config object merge paths to ensure user-supplied JSON always passes
through Zod schema parsing _before_ any `Object.assign` or spread merge operation.
Add a `hasOwnProperty` guard on merge utilities and a lint rule blocking
`{...userSupplied}` spreads on unvalidated data.

---

### 5. OPA/WASM Tool Policy Engine (closes threat #10)

Replace or augment the current sequential policy pipeline
(`applyToolPolicyPipeline`) with a declarative OPA (or WASM) policy module. Policy rules are
versioned, testable, and auditable independently of the gateway code. The engine evaluates tool
name, arguments, session context, and caller identity before execution.

Hook point: `tool-policy-pipeline.ts` as a new pipeline step.

---

## See Also

- [Attack Surface Map](attack-surface-map.md)
- [Security (hardening guide)](index.md)
- [Sandboxing](/gateway/sandboxing)
- [Sandbox vs Tool Policy vs Elevated](/gateway/sandbox-vs-tool-policy-vs-elevated)
- [Secrets](/gateway/secrets)
