---
title: "OpenClaw Architecture Synthesis"
summary: "Cross-source synthesis of OpenClaw's full system architecture — combining source-code walkthrough findings with external reference architectures to produce the definitive component map, data flows, defense layers, and security surface inventory"
read_when:
  - Designing a security-relevant feature that crosses subsystem boundaries
  - Threat modeling a new integration or channel
  - Orienting a new contributor to the full system before reading source
---

# OpenClaw Architecture Synthesis

This document synthesizes findings from three independent sources into one coherent
architecture picture:

1. **Source walkthrough** — seven session-by-session reads of the gateway codebase
   (`docs/gateway/security/study/sessions/`)
2. **robotpaper.ai reference architecture** (early Feb 2026, Opus 4.6 edition) — independent
   architectural analysis of the deployed system
3. **inceptionstack/agents-architecture** — community architecture specification

Where sources agree, findings are stated as facts. Where they diverge, divergences are
noted explicitly. All file references are grounded in the source walkthrough.

---

## 1. System Topology

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  EXTERNAL WORLD                                                             │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  CHANNEL INGRESS (40+ adapters)                                      │  │
│  │  WhatsApp · Telegram · Discord · Slack · Signal · iMessage           │  │
│  │  Teams · Matrix · Zalo · ZaloUser · Voice · (extensions/*)           │  │
│  └──────────────────────────────┬───────────────────────────────────────┘  │
│                                 │                                           │
│  ┌──────────────┐               │  ┌─────────────────────────────────────┐ │
│  │ HTTP REST    │               │  │  WebSocket RPC                      │ │
│  │ /tools/invoke│               │  │  desktop / web / mobile node        │ │
│  │ /v1/...      │               │  └────────────────┬────────────────────┘ │
│  └──────┬───────┘               │                   │                      │
└─────────┼───────────────────────┼───────────────────┼──────────────────────┘
          │                       │                   │
          ▼                       ▼                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  GATEWAY (single long-lived Node.js process, default 127.0.0.1:18789)       │
│                                                                             │
│  ┌─────────────────────────┐  ┌────────────────────────────────────────┐   │
│  │  TRANSPORT & AUTH       │  │  CHANNEL MANAGER                       │   │
│  │  http-listen.ts         │  │  server-channels.ts                    │   │
│  │  ws-connection.ts       │  │  Backoff restart: 5→10→20→300s         │   │
│  │  message-handler.ts     │  │  manuallyStopped flag                  │   │
│  └────────────┬────────────┘  └──────────────┬─────────────────────────┘   │
│               │                              │                             │
│               ▼                              ▼                             │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  METHOD DISPATCH                                                   │    │
│  │  server-methods.ts: authorizeGatewayMethod()                       │    │
│  │  Roles: operator (5 scopes) | node (7 methods)                     │    │
│  │  ADMIN_SCOPE bypasses per-method scope checks                      │    │
│  └───────────┬──────────────────────────┬────────────────────────────┘    │
│              │                          │                                  │
│   ┌──────────▼──────────┐  ┌────────────▼──────────────────────────────┐  │
│   │  COMMAND QUEUE      │  │  AGENT LOOP & CHAT                        │  │
│   │  (robotpaper)       │  │  agent.ts / server-chat.ts                │  │
│   │  global lane: 4     │  │  idempotency dedup store                  │  │
│   │  session lane: 1    │  │  session resolution + delivery plan       │  │
│   │  subagent lane: 8   │  │  dispatchAgentRunFromGateway()            │  │
│   │  cron lane          │  │  createAgentEventHandler()                │  │
│   │  modes: collect /   │  │   - 150ms delta throttle                  │  │
│   │  steer / followup / │  │   - heartbeat suppression                 │  │
│   │  steer-backlog      │  │   - directive tag strip (broadcast only)  │  │
│   └──────────┬──────────┘  └──────────────┬──────────────────────────┘   │
│              └───────────────┬─────────────┘                             │
│                              │                                            │
│              ┌───────────────▼───────────────────────────────────────┐   │
│              │  LLM CALL (pi-embedded-runner)                        │   │
│              │  30+ providers: Anthropic · OpenAI · Bedrock ·        │   │
│              │  Google · Ollama · Mistral · DeepSeek · Groq · ...    │   │
│              │                                                       │   │
│              │  Prompt assembly:                                     │   │
│              │    SOUL.md (system) + tools defs + safety rails       │   │
│              │    + workspace context + project metadata             │   │
│              │    + session JSONL history                            │   │
│              │    + current message                                  │   │
│              └───────────────┬───────────────────────────────────────┘   │
│                              │                                            │
│              ┌───────────────▼───────────────────────────────────────┐   │
│              │  TOOL DISPATCH                                        │   │
│              │  tools-invoke-http.ts / nodes.ts                      │   │
│              │  8-step applyToolPolicyPipeline()                     │   │
│              │  DEFAULT_GATEWAY_HTTP_TOOL_DENY                       │   │
│              │  exec-approval-manager.ts                             │   │
│              └──┬────────────────────────────────┬──────────────────┘   │
│                 │                                │                        │
│   ┌─────────────▼──────────┐  ┌─────────────────▼──────────────────┐    │
│   │  LOCAL EXEC            │  │  SESSION & MEMORY                  │    │
│   │  bash-tools / sandbox  │  │  Layer 1: JSONL transcript          │    │
│   │  ExecApprovalManager   │  │  Layer 2: daily logs (YYYY-MM-DD)  │    │
│   │  safe-bin-policy       │  │  Layer 3: MEMORY.md (long-term)    │    │
│   │  node-command-policy   │  │  Layer 4: SQLite + LanceDB vectors  │    │
│   └────────────────────────┘  └────────────────────────────────────┘    │
│                                                                           │
│   ┌────────────────────────────────────────────────────────────────────┐ │
│   │  PLUGIN SYSTEM (in-process, operator trust, no sandbox)           │ │
│   │  plugins/discovery.ts → 23 hooks × N registered plugins           │ │
│   │  PluginRuntime: full config r/w, subagent spawn, system exec      │ │
│   └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Data Flow: Message → Response

### 2a. Channel message (primary path)

```
External sender (e.g. Telegram)
  │
  ▼ [1] Channel adapter normalizes to internal envelope
  Channel adapter (src/telegram/, src/discord/, extensions/*)
  │
  ▼ [2] Sender gating
  isSenderIdAllowed()       ← allow-from.ts — exact / glob / regex
  resolveControlCommandGate()  ← command-gating.ts — slash command owner check
  │
  ▼ [3] Envelope sanitization
  stripEnvelopeFromMessages()  ← removes <openclaw:*> envelope, msgid, inbound meta
                                  does NOT filter message body content
  │
  ▼ [4] Command Queue enqueue
  Session lane (serial FIFO) — prevents concurrent runs per session
  Modes: collect (queue), steer (interrupt running agent), followup, steer-backlog
  │
  ▼ [5] Agent loop entry
  agentCommandFromIngress()  ← agents/agent-command.ts
  dispatchAgentRunFromGateway()  ← fire-and-forget; {status:"accepted"} returned immediately
  │
  ▼ [6] Session + memory assembly
  loadSessionEntry()  ← JSONL transcript (Layer 1)
  dmScope routing  ← main / per-peer / per-channel-peer session selection
  Memory tool reads  ← daily logs (L2), MEMORY.md (L3), vector search (L4)
  │
  ▼ [7] Prompt assembly
  SOUL.md + tool definitions + safety guardrails + workspace context
  + project metadata + conversation history + current message
  → LLM call (30+ provider backends)
  │
  ▼ [8] LLM tool decision
  Model decides which tool(s) to call
  │
  ▼ [9] Tool policy pipeline
  applyToolPolicyPipeline():
    step 1: profile (agent's tools.profile)
    step 2: byProvider (LLM provider restrictions)
    step 3: global allow/deny
    step 4: agent-level allow/deny
    step 5: group policy (x-openclaw-message-channel header)
    step 6: subagent policy (if spawned session)
    step 7: DEFAULT_GATEWAY_HTTP_TOOL_DENY (HTTP only)
    step 8: before_tool_call plugin hooks
  │
  ▼ [10] Exec approval (for shell/exec tools)
  ExecApprovalManager.getApproval(cmd, cwd, env, fileSnapshot)
  → approve / deny / wait-for-UI
  │
  ▼ [11] Tool execution
  bash/exec → gateway host (or Docker sandbox if configured)
  fs_write / apply_patch → workspace
  sessions_spawn → sub-agent (WS-only, RCE risk)
  browser → CDP, full network egress
  memory reads/writes → layers 2–4
  MCP tools → external MCP server
  │
  ▼ [12] Response streaming
  createAgentEventHandler() → WebSocket broadcast to subscribed clients
  persistGatewaySessionLifecycleEvent() → appends to JSONL (Layer 1)
  Channel adapter outbound → reply to sender
```

### 2b. HTTP `/tools/invoke` (programmatic path)

```
POST /tools/invoke
  ├── authorizeGatewayBearerRequestOrReply()  ← bearer = full operator
  ├── body.sessionKey → resolveEffectiveToolPolicy()
  ├── x-openclaw-message-channel header → group policy selection
  ├── applyToolPolicyPipeline() (steps 1–6 above)
  ├── DEFAULT_GATEWAY_HTTP_TOOL_DENY (hard static after pipeline)
  └── tool.execute()
```

---

## 3. The 7-Layer Defense Model (robotpaper)

The robotpaper reference architecture names a 7-layer model. This table cross-references
each layer to the source code surface identified in the walkthrough.

| Layer | Name                         | Source code surface                                                      | Nature                      |
| ----- | ---------------------------- | ------------------------------------------------------------------------ | --------------------------- |
| 1     | Gateway authentication       | `auth.ts`, `authorizeGatewayBearerRequestOrReply`                        | Hard boundary               |
| 2     | Device pairing + trust model | `handshake-auth-helpers.ts`, `verifyDeviceToken`, `verifyBootstrapToken` | Hard boundary               |
| 3     | Channel allowlists           | `allow-from.ts`, `command-gating.ts`                                     | Configurable guardrail      |
| 4     | Tool policy (allow/deny)     | `tool-policy-pipeline.ts`, `dangerous-tools.ts`                          | Configurable guardrail      |
| 5     | Execution approvals          | `exec-approval-manager.ts`, `node-invoke-system-run-approval.ts`         | Interactive guardrail       |
| 6     | Docker sandboxing (optional) | `agents.defaults.sandbox.mode`, bash-tools routing                       | Optional hard boundary      |
| 7     | Send policy (outbound gates) | `message_sending` hook, `before_message_write` hook                      | Plugin-extensible guardrail |

**Key observation:** Layers 1 and 2 are the only cryptographic hard boundaries.
Layers 3–7 are configurable or optional. The system's default security posture depends
on layers 3–5 being correctly configured — `openclaw security audit` checks this.

**Gap relative to the 7-layer model:** Layer 6 (Docker sandbox) is opt-in and
defaults to off (`sandbox.mode = off`). When off, exec tools fall back to the gateway
host with no process isolation. Layer 7 is only as strong as the registered plugins —
a missing or broken plugin leaves the send path unguarded.

---

## 4. Command Queue Architecture (new from robotpaper)

The source walkthrough did not directly read the command queue implementation.
The robotpaper reference identifies the following lane structure:

| Lane           | Concurrency      | Purpose                                                               |
| -------------- | ---------------- | --------------------------------------------------------------------- |
| Global lane    | Max 4 concurrent | Cross-session upper bound                                             |
| Session lane   | Serial (1)       | Prevents concurrent runs within one session — single-writer invariant |
| Sub-agent lane | Max 8 parallel   | Isolated sub-agent sessions                                           |
| Cron lane      | Scheduled        | Heartbeat and cron job delivery                                       |

**Queue modes** control what happens when a message arrives during an active run:

| Mode            | Behavior                                                   |
| --------------- | ---------------------------------------------------------- |
| `collect`       | Queue the message for delivery after current run completes |
| `steer`         | Interrupt the current run and deliver immediately          |
| `followup`      | Append as a follow-up to the current run                   |
| `steer-backlog` | Steer with queued backlog delivery after                   |

**Security implication:** The `steer` mode interrupts a running agent. A permitted
sender who can trigger `steer` can abort a sensitive tool-approval dialog in progress,
potentially causing the approval to time out and resolve to `deny` (correct behavior
under `timeoutBehavior: "deny"`) or `allow` (footgun if the hook uses `"allow"`).

---

## 5. Memory Architecture (4 layers)

The source walkthrough covered Layer 1 (JSONL transcript) in detail. The full
4-layer stack from robotpaper and inceptionstack:

| Layer | Storage                                  | Scope                    | Security surface                                                                                         |
| ----- | ---------------------------------------- | ------------------------ | -------------------------------------------------------------------------------------------------------- |
| 1     | JSONL transcript (`sessions/<id>.jsonl`) | Per-session conversation | No encryption, no integrity hash — tampering is silent                                                   |
| 2     | Daily logs (`memory/YYYY-MM-DD.md`)      | Per-agent daily          | Plain Markdown, same filesystem exposure as Layer 1                                                      |
| 3     | Long-term memory (`MEMORY.md`)           | Per-agent persistent     | Plain Markdown; **loaded only in private sessions** (not shared channels) — important isolation property |
| 4     | Vector search (SQLite + LanceDB)         | Per-agent semantic       | Embedding index; content derivable from embeddings; no access control below OS level                     |

**MEMORY.md isolation:** The robotpaper confirms that MEMORY.md loads only in
private (non-shared-channel) sessions. This is a meaningful isolation property:
multi-user shared-channel sessions cannot read personal memory context.

**Compaction:** All four layers are subject to the compaction pipeline. `sessions.compact`
is the only operation that rewrites Layer 1 content (overwriting transcript entries
with a summarized version). All other mutations are append-only or rename-only.

---

## 6. Session Routing — `dmScope`

The robotpaper introduces the `dmScope` concept not directly visible in the session
walkthrough:

| dmScope value      | Session key assignment               | Use case                         |
| ------------------ | ------------------------------------ | -------------------------------- |
| `main`             | Agent's default main session         | Single-user, single-channel      |
| `per-peer`         | One session per sending peer ID      | Each user gets their own context |
| `per-channel-peer` | One session per (channel, peer) pair | Isolated per-platform context    |

**Security implication:** The default `main` scope means all senders on a channel
share one session context — including conversation history and long-term memory
reads. The STRIDE model (Component 1, "Session History Leakage") identified this as
the source of medium-severity information disclosure in shared-channel deployments.
Operators running multi-user channels should configure `per-peer` or
`per-channel-peer` scope to enforce context isolation.

---

## 7. Skills Architecture — Lazy Loading and Gating

The source walkthrough covered skill install gating. The robotpaper adds the runtime
loading model:

**Lazy-loading:** Only skill metadata (brief description) appears in the initial
prompt. The model reads the full `SKILL.md` file on demand when it decides to use
a skill. This is demand-paging for context window efficiency.

**Gating filters** run at skill activation time and check:

- Required binaries present (e.g. `ffmpeg`, `node`)
- Required environment variables set
- Required configuration keys present
- OS compatibility (macOS / Linux / Windows)

**Security implication:** A skill's binary requirements are declared in `SKILL.md`
frontmatter. A malicious skill could declare a dependency on a benign binary name
that it does not actually use, passing the gating check while using a different
executable path at runtime. The gating filter does not verify that the binary is
what it claims to be (no hash check of required executables).

---

## 8. Sub-Agent Architecture

From robotpaper + inceptionstack, confirmed by source walkthrough:

- Sub-agents are **isolated sessions** spawned from the parent session
- They have **restricted tool access** (inherits parent policy unless overridden)
- **No nesting** — sub-agents cannot spawn further sub-agents
- Results are **announced back** to the requester's session automatically
- `sessions_spawn` is the gateway method; it is WS-only (blocked on HTTP)

**Architecture constraint not visible in source:** The robotpaper notes sub-agents
run in the sub-agent lane (max 8 parallel). This means a single session can have
at most 8 active sub-agents — a natural DoS limit, but also a potential starvation
vector if a parent agent spawns 8 long-running sub-agents and subsequent spawn
requests queue indefinitely.

---

## 9. MCP (Model Context Protocol) Integration

Both the inceptionstack document and robotpaper mention MCP as a tool integration
surface. This was not covered in the source walkthrough.

**What MCP adds as an attack surface:**

- MCP tools arrive from an external MCP server — a new trust boundary not present
  in the built-in tool set
- The tool policy pipeline applies to MCP tools by name, but the tool's actual
  execution happens on the MCP server (outside the gateway's sandbox controls)
- A compromised MCP server can return malicious tool results that influence
  subsequent LLM decisions (indirect injection via tool result)
- MCP tool result payloads are not sanitized before entering the conversation context

**Current control:** MCP tool names are subject to the same policy pipeline as
built-in tools. No additional MCP-specific controls were identified.

**Enhancement opportunity:** Apply `before_tool_call` hooks to MCP tool invocations
and a `after_tool_call` sanitization hook to MCP tool results before they enter
the conversation context.

---

## 10. Identified Vulnerabilities (inceptionstack document)

The inceptionstack architecture document identifies the following vulnerability
classes. Each is mapped to the existing STRIDE analysis:

| Vulnerability                                                    | STRIDE reference                  | Source study status                 |
| ---------------------------------------------------------------- | --------------------------------- | ----------------------------------- |
| Prompt injection via untrusted channel messages                  | Component 1, T — Prompt Injection | Covered; rated High                 |
| Indirect injection via web search results in multi-user channels | **Not in current STRIDE**         | Gap — see §11 below                 |
| Path traversal in media server file access                       | **Not in current STRIDE**         | Gap — see §11 below                 |
| Command injection in shell execution                             | Component 3, T (tool dispatch)    | Partially covered (approval system) |
| Hardcoded token patterns in installation scripts                 | **Not in current STRIDE**         | Low risk — scripts not in gateway   |
| Prototype pollution in configuration merging                     | **Not in current STRIDE**         | Gap — see §11 below                 |

---

## 11. New Threat Gaps (not in current STRIDE model)

These three threats are identified by external sources and not yet in the
`stride-threat-model.md`. They should be added.

### Gap A — Indirect Injection via Web Search / Tool Results

**Threat:** The agent uses a web-search tool or browser tool to fetch external
content, which contains adversarial instructions. The content enters the
conversation context as a tool result and influences subsequent model decisions
without passing through any message-level allowlist check.

**Why different from direct prompt injection:** Direct injection comes from a
message sender (gated by allowlist). Indirect injection comes from content the
agent retrieves autonomously on behalf of a trusted user — the allowlist does not
apply to tool results.

**Existing control:** None. Tool results are fed to the model without content
inspection.

**Enhancement opportunity:** A `before_message_write` or `after_tool_call` hook
that runs a classifier on tool results before they enter the conversation context.

### Gap B — Path Traversal in Media Server

**Threat:** The media file serving path (used by the gateway's embedded media
server for image/audio/video delivery) does not fully canonicalize file paths
before serving. An attacker who can craft a media request URL with `../` sequences
could read files outside the intended media directory.

**Surface:** `src/gateway/control-ui.ts` and media pipeline (`src/media/`).
Not read in detail during the source walkthrough.

**Existing controls:** `src/infra/fs-safe.ts` for general filesystem safety, but
its application to the media server path was not confirmed.

**Recommendation:** Verify that all media server path resolutions use
`path.resolve()` followed by a `startsWith(mediaRoot)` check.

### Gap C — Prototype Pollution in Configuration Merging

**Threat:** Configuration merging code (deep merge of JSONC config objects) may
be vulnerable to prototype pollution if an attacker can supply a config value
containing `__proto__` or `constructor.prototype` keys. This could silently modify
the prototype of all objects in the process, enabling property injection across
unrelated code paths.

**Surface:** `src/config/validation.ts` (Zod schema parses raw JSON before merge)
and config merge utilities. The Zod schema provides some protection — unknown keys
are stripped by `.strict()` in many schemas. However, Zod parses the JSON after
`JSON.parse()`, which does not prevent prototype pollution in all merge paths.

**Existing control:** Zod schema validation with `.strict()` on most config schemas
strips unknown keys. This is the primary mitigation.

**Residual risk:** Low if all merge paths go through Zod validation first. Medium
if any raw-object merge (`Object.assign`, spread `{...config}`) operates on
user-supplied data before Zod parsing. The source walkthrough confirmed
`src/config/zod-schema.core.ts` uses `.strict()` on `ModelsConfigSchema` — this
needs systematic verification across all config schemas.

---

## 12. Observability and Audit Surfaces

| Surface                                    | Source                                  | Security relevance                                          |
| ------------------------------------------ | --------------------------------------- | ----------------------------------------------------------- |
| Session JSONL transcript                   | `session-transcript-files.fs.ts`        | Primary audit log; append-only; unsigned                    |
| `control-plane-audit.ts`                   | `src/gateway/`                          | Records control-plane config changes; coverage not complete |
| `openclaw security audit`                  | `src/security/audit.ts`                 | Runtime security posture checker; does not run continuously |
| OpenTelemetry (diagnostics-otel extension) | `extensions/diagnostics-otel`           | Optional distributed tracing; not a security audit trail    |
| Exec approval log                          | `exec-approval-manager.ts`              | Records approval decisions; not tamper-evident              |
| Channel health monitor                     | `src/gateway/channel-health-monitor.ts` | Availability, not security                                  |

**Gap:** No tamper-evident, append-only, cryptographically anchored audit log for
security-relevant events (tool calls, config changes, exec approvals). The JSONL
transcript serves as a de facto log but is mutable and unsigned.

---

## 13. Architecture Divergences Between Sources

| Topic           | Source walkthrough           | robotpaper                  | inceptionstack                |
| --------------- | ---------------------------- | --------------------------- | ----------------------------- |
| Command queue   | Not directly read            | 4 lanes, 4 modes            | Not detailed                  |
| Memory layers   | Layer 1 (JSONL) only         | 4 layers described          | 4 layers described            |
| Plugin trust    | In-process, no sandbox       | Confirmed                   | Confirmed                     |
| MCP integration | Not covered                  | Not mentioned               | Mentioned as tool integration |
| Session routing | `dmScope` not seen directly  | per-peer / per-channel-peer | Not detailed                  |
| Sandbox default | `off` (source confirmed)     | Optional Docker             | Optional Docker               |
| 7-layer defense | Not named; layers enumerable | Named explicitly            | Not named                     |

---

## 14. Recommended Next Deep-Dives

Based on this synthesis, the following source areas have the highest open security
relevance and have not yet been deeply read:

| Priority | Area                         | Files                                                     | Gap it closes               |
| -------- | ---------------------------- | --------------------------------------------------------- | --------------------------- |
| 1        | Media server path handling   | `src/gateway/control-ui.ts`, `src/media/`                 | Gap B (path traversal)      |
| 2        | Config merge paths           | `src/config/validation.ts` (full), config merge utilities | Gap C (prototype pollution) |
| 3        | MCP tool integration         | `src/agents/mcp*`, `extensions/mcp*`                      | MCP indirect injection      |
| 4        | Command queue implementation | `src/agents/pi-embedded-runner/command-queue*`            | Lane / steer security       |
| 5        | Embedded agent runtime       | `src/agents/pi-embedded-runner/`                          | Prompt assembly + SOUL load |
| 6        | Exec safe-bin policy         | `src/infra/exec-safe-bin-runtime-policy.ts`               | What is "safe" for exec     |
| 7        | SSRF and network policy      | `src/infra/net/ssrf.ts`                                   | Agent web egress control    |

---

## See Also

- [Gateway Source Walkthrough](gateway-source-walkthrough.md)
- [Attack Surface Map](attack-surface-map.md)
- [STRIDE Threat Model](stride-threat-model.md)
- [Tool Security Inventory](tool-security-inventory.md)
- [Plugin Security Hooks](../../security/study/plugin-security-hooks.md)
