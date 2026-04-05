---
title: "Attack Surface Map"
summary: "Component-level attack surface inventory for the OpenClaw Gateway — ingress points, trust boundaries, and data flows grounded in source code"
read_when:
  - Threat modeling a new feature
  - Reviewing a security-relevant change
  - Orienting to the security architecture
---

# Attack Surface Map

This document maps the Gateway's externally reachable and internally sensitive surfaces to their
source locations, trust boundaries, and key security properties. It is grounded in a direct read of
the source code and is intended as the starting point for threat modeling (see
[STRIDE Threat Model](stride-threat-model.md)) and for reviewing changes to high-risk areas.

---

## Trust Boundary Summary

OpenClaw uses a **personal-assistant trust model**: one trusted operator per gateway, one or more
agents inside that gateway, multiple messaging channels as ingress. The relevant trust boundaries
are:

| Boundary                 | Description                                                                                                     |
| ------------------------ | --------------------------------------------------------------------------------------------------------------- |
| **Operator boundary**    | Anyone who can authenticate to the gateway or modify `~/.openclaw` is a trusted operator                        |
| **Channel boundary**     | Messages arriving over WhatsApp, Telegram, Slack, etc. are untrusted content from potentially untrusted senders |
| **Agent/model boundary** | The LLM is not a trusted principal — prompt injection can manipulate behavior                                   |
| **Sandbox boundary**     | Exec tools run on the gateway host unless `agents.defaults.sandbox.mode` is set                                 |
| **Skill boundary**       | Installed skills run in-process with the gateway and inherit operator trust                                     |

---

## Surface Inventory

### 1. Gateway HTTP — `POST /tools/invoke`

**Source:** `src/gateway/tools-invoke-http.ts`

The primary programmatic API surface. Any process that can reach the gateway port and hold a valid
bearer token can invoke tools directly — bypassing the agent conversation loop entirely.

**Authentication:** `authorizeGatewayBearerRequestOrReply` — bearer token or password, with optional
rate limiting via `AuthRateLimiter`.

**Policy pipeline (executed on every call):**

```
inbound POST /tools/invoke
  ├── bearer auth (token / password)
  ├── resolveEffectiveToolPolicy()      — profile + global + agent + group layers
  ├── applyToolPolicyPipeline()         — ordered allow/deny steps
  ├── DEFAULT_GATEWAY_HTTP_TOOL_DENY    — static deny list applied last
  └── tool.execute()
```

**Static deny list** (`src/security/dangerous-tools.ts:9`):

| Denied tool      | Risk rationale                           |
| ---------------- | ---------------------------------------- |
| `sessions_spawn` | Remote code execution via sub-agent      |
| `sessions_send`  | Cross-session message injection          |
| `cron`           | Persistent automation control plane      |
| `gateway`        | Gateway reconfiguration                  |
| `whatsapp_login` | Interactive terminal flow, hangs on HTTP |

**Key security property:** callers that pass gateway bearer auth are treated as full trusted
operators. There is no narrower `operator.write` vs `operator.admin` scope split on this endpoint.

---

### 2. Gateway WebSocket — Agent/Client Sessions

**Source:** `src/gateway/server-ws-runtime.ts`, `src/gateway/server-methods.ts`

The real-time control channel between the gateway and connected clients (desktop app, web UI,
mobile nodes). All `agent`, `chat`, `sessions.*`, and `gateway.*` RPC methods run over this
surface.

**Authentication:** `connection-auth.ts` — handshake at connection open; separate rate limiter for
browser-origin connections (`browserRateLimiter` with loopback exemption disabled).

**Notable methods:**

- `sessions_spawn` — spawns a sub-agent; explicitly blocked on HTTP but available over WS to
  authenticated operators
- `gateway` — gateway reconfiguration; available to authenticated operators

---

### 3. Channel Ingress — Messaging Adapters

**Source:** `src/channels/`, `src/telegram/`, `src/discord/`, `src/slack/`, `src/signal/`,
`src/imessage/`, `src/web/` (WhatsApp), `extensions/` (Teams, Matrix, Zalo, voice)

Each channel adapter normalizes an inbound message from the external platform and delivers it into
the conversation loop. The normalization pipeline strips OpenClaw's own envelope format
(`stripEnvelopeFromMessages` in `src/gateway/chat-sanitize.ts`) but does **not** sanitize
arbitrary message body content.

**Injection surface:** A message body from any channel lands largely intact in the conversation
context. Adversarial content in a WhatsApp or Telegram message can attempt to override SOUL context
or issue tool-call instructions to the model.

**Allowlist controls:** `src/channels/allow-from.ts`, `src/channels/allowlist-match.ts` —
per-channel sender allowlisting. This is the primary gatekeeping layer for untrusted senders.

**Command gating:** `src/channels/command-gating.ts` — governs which senders can invoke slash
commands.

---

### 4. Message Pipeline — Sanitization and History Assembly

**Source:** `src/gateway/chat-sanitize.ts`, `src/gateway/agent-prompt.ts`

Inbound messages flow through:

```
raw channel message
  └── stripEnvelopeFromMessages()    — removes OpenClaw envelope, msgid hints, inbound metadata
        └── buildAgentMessageFromConversationEntries()  — assembles history + current message
              └── LLM call (system prompt = SOUL.md + context)
```

`stripEnvelopeFromMessages` operates on `text` and `content[]` fields, extracting
`senderLabel` for history context. It does not inspect or modify semantic content — adversarial
instructions in the message body are forwarded to the model unchanged.

---

### 5. Session Storage — JSONL Transcripts

**Source:** `src/gateway/session-transcript-files.fs.ts`, `src/sessions/`

Sessions are stored as append-only JSONL files:

```
~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl
```

**Properties:**

- Append-only writes minimize partial-write data loss on crash
- No encryption at rest
- No integrity/tamper detection (no hash, no signature)
- Archive operation uses `fs.renameSync` to `<path>.<reason>.<timestamp>`
- Path resolution has a multi-candidate fallback chain including a legacy `~/.openclaw/sessions/`
  directory — relevant to path traversal checks when `sessionFile` is operator-supplied

**Risk:** An attacker with local filesystem access (or a tool that escapes the sandbox) can silently
modify session history that the agent will re-read on continuation.

---

### 6. SOUL.md — Agent Identity and System Prompt

**Source:** `docs/reference/templates/SOUL.md`, assembled into every LLM call

SOUL.md is the agent's system prompt file. It defines behavioral rules, persona, and trust context.

**Properties:**

- Plain Markdown file on the local filesystem, no integrity verification
- Read at each agent run; modification takes effect on next run without any alert
- No hash-signing or runtime tamper detection

**Risk:** A local attacker, or a tool that can write to the workspace, can silently modify
agent identity — effectively a persistent prompt injection at the system-prompt level.

---

### 7. Tool Execution — Exec / Bash

**Source:** `src/agents/bash-tools.exec-host-gateway.ts`, `src/agents/bash-tools.exec-runtime.ts`,
`src/gateway/exec-approval-manager.ts`, `src/gateway/node-invoke-system-run-approval.ts`

Shell execution runs on the gateway host by default (`agents.defaults.sandbox.mode = off`).

**Exec approval system:** `ExecApprovalManager` tracks allowlisted command patterns. Approvals bind
exact command/cwd/env context and, when identifiable, a snapshot of the target file.

**ACP dangerous tools** (require explicit approval, `src/security/dangerous-tools.ts:26`):
`exec`, `spawn`, `shell`, `sessions_spawn`, `sessions_send`, `gateway`, `fs_write`, `fs_delete`,
`fs_move`, `apply_patch`.

**Sandbox routing:** `tools.exec.host` defaults to `sandbox` as a preference, but if no sandbox
runtime is active for the session, execution falls back to the gateway host.

---

### 8. Skill Supply Chain — Install and Runtime

**Source:** `src/security/skill-scanner.ts`, `src/agents/skills/`

Skills are installed packages that extend the agent's tool set. The security scanner
(`skill-scanner.ts`) performs static analysis on JS/TS files — up to 500 files, 1 MB each —
producing `SkillScanFinding[]` with severity levels (`info`/`warn`/`critical`).

**Scanner properties:**

- Pattern-based static analysis on file content
- Covers `.js`, `.ts`, `.mjs`, `.cjs`, `.jsx`, `.tsx`
- File and directory scan results are cached with mtime invalidation
- No behavioral sandboxing — does not isolate network or filesystem access during analysis

**Runtime trust:** Installed skills run in-process with the gateway and inherit operator-level
trust. Only install skills from trusted sources. Use `plugins.allow` to pin explicit trusted
plugin IDs.

---

### 9. Network Egress — SSRF and Browser

**Source:** `src/infra/net/ssrf.ts`, `src/plugin-sdk/browser-runtime.ts`

`isBlockedHostnameOrIp` and `isPrivateNetworkAllowedByPolicy` gate web requests from the agent
to prevent Server-Side Request Forgery into private network ranges.

Browser tool (`browser`, CDP) is a privileged egress channel — it can access authenticated sessions
and local network resources. CDP URL is redacted in logs (`redactCdpUrl`), and browser control
auth is separated from the general credential store.

---

### 10. Gateway Exposure — Tailscale and Tunnels

**Source:** `src/gateway/server-tailscale.ts`

Remote access can be configured via:

- SSH tunnel (loopback gateway, tunnel from remote host)
- Tailscale Serve (LAN-accessible)
- Tailscale Funnel (public internet — highest risk)

`openclaw security audit` flags non-loopback bind and Funnel exposure. When remote access is
required, the recommended posture is loopback bind + SSH tunnel or Tailscale Serve (not Funnel),
plus strong gateway auth.

---

## Data Flow Diagram (Text)

```
External Sender (WhatsApp / Telegram / Slack / ...)
        │
        ▼
  Channel Adapter  ──── allow-from check ──── DENIED
        │
        ▼
  stripEnvelopeFromMessages()          ← sanitizes envelope, NOT body content
        │
        ▼
  buildAgentMessageFromConversationEntries()
        │                                ▲
        │                         session JSONL transcript (no integrity check)
        ▼
  LLM call  [system prompt = SOUL.md (no integrity check)]
        │
        ▼
  tool call decision
        ├── policy pipeline check (profile / global / agent / group / subagent)
        ├── DEFAULT_GATEWAY_HTTP_TOOL_DENY (static deny)
        ├── exec approval check (allowlist / ask UI)
        └── tool.execute()
              ├── bash / shell  → gateway host (or sandbox if configured)
              ├── fs_write / apply_patch  → workspace (or host)
              ├── sessions_spawn  → sub-agent (RCE risk, WS-only)
              └── browser  → CDP, network egress
```

---

## High-Risk Change Indicators

When reviewing a change, treat any of the following as a signal to apply extra scrutiny:

- Modifying `DEFAULT_GATEWAY_HTTP_TOOL_DENY` or `DANGEROUS_ACP_TOOLS`
- Adding a new path to `/tools/invoke` handler or relaxing bearer-auth enforcement
- Changing `stripEnvelopeFromMessages` or the history assembly pipeline
- Adding new fields to the session JSONL format that get re-read into context
- Adding a new skill or modifying skill install/enable gating
- Changing `exec-approval-manager.ts` allowlist logic
- Relaxing SSRF checks in `src/infra/net/ssrf.ts`
- Changing sandbox fallback behavior (when `sandbox.mode` is unset)
- Exposing the gateway on a non-loopback interface without a corresponding auth requirement

---

## See Also

- [STRIDE Threat Model](stride-threat-model.md)
- [Security (hardening guide)](index.md)
- [Sandboxing](/gateway/sandboxing)
- [Tools Invoke HTTP API](/gateway/tools-invoke-http-api)
- [Trusted Proxy Auth](/gateway/trusted-proxy-auth)
