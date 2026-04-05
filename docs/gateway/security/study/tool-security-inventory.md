---
title: "Tool Security Inventory"
summary: "Security-classified inventory of all OpenClaw agent tools — exec risk, policy pipeline placement, HTTP deny list membership, approval requirements, and per-tool threat notes"
read_when:
  - Evaluating the security impact of a new tool
  - Reviewing tool policy configuration for a deployment
  - Auditing which tools are reachable over HTTP vs WebSocket
---

# Tool Security Inventory

This document classifies every OpenClaw agent tool from a security perspective.
It answers three questions for each tool:

1. **Reachability** — is this tool reachable over HTTP, WebSocket, both, or neither?
2. **Approval requirement** — does it require explicit user approval before execution?
3. **Risk class** — what category of harm can it cause if misused?

Sources: `src/security/dangerous-tools.ts`, `src/gateway/exec-approval-manager.ts`,
`src/gateway/node-invoke-system-run-approval.ts`, `src/gateway/node-command-policy.ts`,
and the source walkthrough (Sessions 6 and 7).

---

## Classification Schema

| Field                | Values                                                                      |
| -------------------- | --------------------------------------------------------------------------- |
| **HTTP reachable**   | Yes / **No (static deny)** / Conditional                                    |
| **WS reachable**     | Yes / Gated (operator role + scope required)                                |
| **Approval**         | None / Exec approval manager / Node approval / UI prompt                    |
| **Risk class**       | RCE · Data destruction · Exfiltration · Pivot · Config · Availability · Low |
| **Policy placement** | Where in the 8-step pipeline this tool is controlled                        |

---

## Tier 1 — Statically Denied on HTTP

These tools are in `DEFAULT_GATEWAY_HTTP_TOOL_DENY` (`src/security/dangerous-tools.ts:9`).
They cannot be invoked over `POST /tools/invoke` regardless of bearer token or
policy configuration. The deny is applied after the full policy pipeline as a final
hard guard.

| Tool             | Reason (code comment)                      | WS reachable   | Risk class                                |
| ---------------- | ------------------------------------------ | -------------- | ----------------------------------------- |
| `sessions_spawn` | "spawning agents remotely is RCE"          | Yes (operator) | **RCE** — sub-agent with full tool access |
| `sessions_send`  | Cross-session message injection            | Yes (operator) | **Pivot** — inject into another session   |
| `cron`           | Persistent automation control plane        | Yes (operator) | **Persistence** — survives session end    |
| `gateway`        | Gateway reconfiguration                    | Yes (operator) | **Config** — can change auth, policies    |
| `whatsapp_login` | "interactive terminal flow, hangs on HTTP" | Yes (operator) | **Availability** + credential exposure    |

**Security note on `sessions_spawn`:** Even over WS, an authenticated operator can
spawn a sub-agent with the parent session's full tool policy unless an explicit
`delegatedTools` override is provided. This is the sub-agent RCE vector described
in the STRIDE model (Component 2, E).

---

## Tier 2 — Dangerous ACP Tools (Require Explicit Approval)

These tools are in `DANGEROUS_ACP_TOOL_NAMES` (`src/security/dangerous-tools.ts:26`).
They are subject to the exec approval system (`exec-approval-manager.ts`) or the
node approval system (`node-invoke-system-run-approval.ts`). Absent a stored
approval entry, the gateway pauses and requests UI confirmation before executing.

| Tool             | Approval type         | Risk class            | Notes                                               |
| ---------------- | --------------------- | --------------------- | --------------------------------------------------- |
| `exec`           | Exec approval manager | **RCE**               | Shell exec on gateway host; exact cmd/cwd/env bound |
| `spawn`          | Exec approval manager | **RCE**               | Process spawn variant of exec                       |
| `shell`          | Exec approval manager | **RCE**               | Shell invocation; same risk as exec                 |
| `fs_write`       | Exec approval manager | **Data destruction**  | Writes arbitrary files on host                      |
| `fs_delete`      | Exec approval manager | **Data destruction**  | Deletes files; irreversible without backup          |
| `fs_move`        | Exec approval manager | **Data modification** | Can overwrite destination                           |
| `apply_patch`    | Exec approval manager | **Data modification** | Patch application to source files                   |
| `sessions_spawn` | Node approval (WS)    | **RCE**               | Also in Tier 1 deny for HTTP                        |
| `sessions_send`  | Node approval (WS)    | **Pivot**             | Also in Tier 1 deny for HTTP                        |
| `gateway`        | Exec approval manager | **Config**            | Also in Tier 1 deny for HTTP                        |

**Approval binding:** Approvals bind `{command, cwd, env snapshot, optional file hash}`.
An approval is point-in-time — it does not model interpreter-level substitution
(variable expansion, `eval`, dynamic `require`). A command string that looks identical
but expands differently in a different environment would pass a cached approval check.

---

## Tier 3 — Node Command Allowlist (Remote Node Execution)

When a tool is dispatched to a remote node (mobile device, second machine) via
`node.invoke`, it passes through `node-command-policy.ts` which applies a
**platform-specific allowlist** (`resolveNodeCommandAllowlist`). Commands not in the
platform's base allowlist require either:

- An explicit `exec-approval-manager` entry for that command, or
- An explicit `elevated` flag in the tool policy

**Opt-in-only commands** (`DEFAULT_DANGEROUS_NODE_COMMANDS`) — not in any platform
base allowlist:

| Command             | Risk class               | Why opt-in only                                 |
| ------------------- | ------------------------ | ----------------------------------------------- |
| `camera.snap`       | **Privacy**              | Silently captures photos                        |
| `screen.record`     | **Privacy/Exfiltration** | Records display content                         |
| `contacts.add`      | **Data modification**    | Adds to device contacts                         |
| `sms.send`          | **Pivot/Exfiltration**   | Sends messages from device number               |
| `call.initiate`     | **Pivot**                | Initiates phone calls                           |
| `microphone.record` | **Privacy**              | Records ambient audio                           |
| `location.get`      | **Privacy**              | Reads precise device location                   |
| `clipboard.read`    | **Exfiltration**         | Reads clipboard content (may contain passwords) |

---

## Tier 4 — Standard Tools (Policy Pipeline Only)

These tools are not in any static deny list but are subject to the 8-step tool
policy pipeline. Their security properties depend entirely on the operator's
configured policy.

### Execution and File System

| Tool           | Risk class            | Default policy notes                                                   |
| -------------- | --------------------- | ---------------------------------------------------------------------- |
| `bash` / `run` | **RCE**               | Typically requires exec approval or sandbox; same risk class as `exec` |
| `fs_read`      | **Exfiltration**      | Can read any file the gateway process can read, including credentials  |
| `fs_list`      | **Exfiltration**      | Directory listing; can enumerate sensitive paths                       |
| `apply_diff`   | **Data modification** | Targeted diff application; lower risk than `apply_patch`               |
| `computer`     | **RCE + Privacy**     | Computer use (screen/mouse/keyboard); full desktop control             |

### Network and External

| Tool           | Risk class                | Notes                                                                |
| -------------- | ------------------------- | -------------------------------------------------------------------- |
| `browser`      | **Exfiltration + Pivot**  | CDP browser; accesses authenticated sessions, local network          |
| `web_search`   | **Indirect injection**    | Search results enter conversation context unfiltered                 |
| `http_request` | **SSRF**                  | Gated by `src/infra/net/ssrf.ts`; private network blocked by default |
| `mcp_*`        | **Depends on MCP server** | External server; tool result indirect injection risk                 |

### Session and Agent Control

| Tool               | Risk class            | Notes                                                    |
| ------------------ | --------------------- | -------------------------------------------------------- |
| `sessions_list`    | **Exfiltration**      | Lists all session IDs; information disclosure            |
| `sessions_preview` | **Exfiltration**      | Reads recent session history                             |
| `sessions_reset`   | **Data modification** | Renames JSONL (not destroyed); recoverable               |
| `sessions_compact` | **Data modification** | Only operation that overwrites transcript content        |
| `sessions_steer`   | **Control**           | Interrupts running agent; different from `sessions_send` |
| `agent`            | **RCE**               | Runs embedded agent; full tool access in spawned run     |

### Memory

| Tool                           | Risk class                   | Notes                                                 |
| ------------------------------ | ---------------------------- | ----------------------------------------------------- |
| `memory_write` / `memory_read` | **Exfiltration + Integrity** | Reads/writes MEMORY.md; can inject persistent context |
| `memory_search`                | **Exfiltration**             | Semantic search over Layer 4 (vector index)           |
| `context_write`                | **Integrity**                | Writes to session context; persisted in JSONL         |

### Media and Communication

| Tool               | Risk class | Notes                                              |
| ------------------ | ---------- | -------------------------------------------------- |
| `image_generate`   | Low        | External API call; no local execution risk         |
| `tts` / `stt`      | Low        | Audio processing; no file system access            |
| `media_understand` | Low        | Image/audio analysis; reads local files            |
| `send_message`     | **Pivot**  | Sends message on behalf of agent; channel-specific |
| `send_reaction`    | Low        | Adds reaction to message                           |

---

## HTTP vs WebSocket Reachability Summary

```
HTTP /tools/invoke          WebSocket (operator role)
─────────────────────────   ──────────────────────────────────────────
ALL tools EXCEPT:           ALL tools including:
  sessions_spawn              sessions_spawn  (approval required)
  sessions_send               sessions_send   (approval required)
  cron                        cron
  gateway                     gateway         (approval required)
  whatsapp_login              whatsapp_login

Bearer token = full         Role + scope gates apply:
operator access             operator.write for most tools
No scope granularity        operator.admin for control plane
```

---

## Tool Policy Pipeline — 8-Step Evaluation Order

Each tool call through `applyToolPolicyPipeline()` traverses these steps in order.
A `deny` at any step short-circuits the rest:

| Step | Policy source    | Scope                                                              |
| ---- | ---------------- | ------------------------------------------------------------------ |
| 1    | `tools.profile`  | Agent-level named profile (e.g. "minimal", "default", "elevated")  |
| 2    | `byProvider`     | Per-LLM-provider restrictions                                      |
| 3    | Global           | `tools.global` allow/deny                                          |
| 4    | Agent            | `agents.<id>.tools` allow/deny                                     |
| 5    | Group            | Channel-context group policy (`x-openclaw-message-channel` header) |
| 6    | Subagent         | Spawned session inherits or overrides parent                       |
| 7    | HTTP static deny | `DEFAULT_GATEWAY_HTTP_TOOL_DENY` (HTTP path only)                  |
| 8    | Plugin hooks     | `before_tool_call` registered handlers                             |

**Policy bypass vector (step 5):** The `x-openclaw-message-channel` header is
caller-supplied over HTTP. An attacker who can set this header selects which group
policy applies. If different groups have different tool access levels, this header
enables policy escalation without changing authentication credentials.

**Policy bypass vector (step 6):** Sub-agents inherit the parent's effective policy
unless a `delegatedTools` list is explicitly provided at spawn time. A parent with a
broad policy spawns broad-access sub-agents by default.

---

## High-Risk Tool Combinations

Some individually-benign tools compose into high-risk chains:

| Chain                              | Risk                                                                          |
| ---------------------------------- | ----------------------------------------------------------------------------- |
| `web_search` + `exec`              | Search result contains shellcode; model executes it as a "helpful command"    |
| `fs_read` + `send_message`         | Read credentials file, exfiltrate via outbound message                        |
| `memory_write` + `session_compact` | Write adversarial instructions to MEMORY.md; compact removes evidence         |
| `browser` + `http_request`         | Browser establishes authenticated session; http_request leverages it for SSRF |
| `sessions_spawn` + `gateway`       | Sub-agent spawned with gateway reconfiguration tool; changes auth token       |
| `apply_patch` + `cron`             | Patch a cron job script to a malicious version; cron executes it on schedule  |

---

## See Also

- [Architecture Synthesis](architecture-synthesis.md)
- [Attack Surface Map](attack-surface-map.md)
- [STRIDE Threat Model](stride-threat-model.md)
- [Session 6 — Tool Dispatch](sessions/session6-tool-dispatch.md)
- [Plugin Security Hooks](../../security/study/plugin-security-hooks.md)
