---
title: "Plugin System Architecture and Security Hooks"
summary: "Detailed study guide for the OpenClaw plugin system — SDK surface, hook types, trust model, sandboxing limits — followed by a design analysis of the before_skill_install and before_tool_call+requireApproval security hooks and proposals for strengthening them"
read_when:
  - Building a plugin that registers security hooks
  - Evaluating the security posture of the plugin system
  - Designing or reviewing changes to before_skill_install or before_tool_call
---

# Plugin System Architecture and Security Hooks

This document has two parts. **Part 1** is a study guide for the OpenClaw plugin
system architecture: what a plugin is, what it receives at runtime, and where the
trust boundary sits. **Part 2** analyzes the two security-relevant hooks that have
recently shipped — `before_skill_install` (PR openclaw/openclaw#56050) and
`before_tool_call` with `requireApproval` (PR openclaw/openclaw#55339) — with a
design critique and concrete proposals for strengthening them.

---

# Part 1 — Plugin System Architecture

## 1. What a plugin is

An OpenClaw plugin is a Node.js ESM package that exports one or more _extension
objects_ declared in `openclaw.plugin.json`. A plugin can provide:

- **Channel adapters** — an inbound/outbound message surface (Discord, Slack, etc.)
- **Tool factories** — new agent tools injected into the tool list
- **Gateway request handlers** — custom WS RPC methods
- **Hook handlers** — callbacks at 23+ named decision points
- **Skill/memory providers** — custom skill bundles or memory backends
- **Provider auth** — LLM provider credentials and model discovery

Plugins live under `extensions/*` (workspace packages) or are installed from npm.
At runtime they run **in-process**: no subprocess, no VM, no WebAssembly boundary.

---

## 2. The plugin manifest (`openclaw.plugin.json`)

Every installed plugin must have a manifest. Key fields:

```jsonc
{
  "id": "my-plugin", // canonical ID, must match package name pattern
  "name": "My Plugin",
  "version": "1.0.0",
  "description": "...",
  "enabledByDefault": false,
  "kind": "memory", // optional: "memory" | "context-engine"

  // Schema for the plugin's config block
  "configSchema": {
    /* JSON Schema */
  },
  "uiHints": {
    "apiKey": { "sensitive": true, "label": "API Key", "help": "..." },
  },

  // Fast-path declarations (read without loading the plugin)
  "contracts": {
    "tools": ["tool_name_1"],
    "speechProviders": ["my-tts"],
    "webSearchProviders": ["my-search"],
  },

  // Channel integration metadata
  "channels": ["discord"],
  "channelConfigs": {
    "discord": {
      "schema": {
        /* JSON Schema */
      },
    },
  },

  // Provider auth declarations
  "providerAuthEnvVars": { "my-provider": ["MY_API_KEY"] },
  "providerAuthChoices": [
    { "provider": "my-provider", "method": "api-key", "choiceId": "main-key" },
  ],

  // Backwards compatibility
  "legacyPluginIds": ["old-name"],
}
```

The manifest is validated before any plugin code runs. The plugin ID must be
consistent across `openclaw.plugin.json:id`, the directory name, and the npm
package name. This is enforced by a repo invariant test.

---

## 3. SDK surface — what a plugin receives

The plugin SDK public surface is `openclaw/plugin-sdk/*`. Plugins must import
only from this path, not from `src/**`. At runtime the plugin receives a
`PluginRuntime` object:

```
PluginRuntime
  ├── version         — semver string (same as CLI VERSION)
  ├── config          — loadConfig() + writeConfigFile()   ← full read/write
  ├── agent           — lifecycle, defaults, workspace dir, embedded Pi runner
  ├── subagent        — spawn child agents (gateway requests only)
  ├── system          — heartbeat trigger, process.exec, system events
  ├── media           — image/audio/video load and manipulation
  ├── tts / stt       — speech synthesis / transcription
  ├── mediaUnderstanding — image description, audio transcription
  ├── channel         — channel-specific runtime methods
  ├── events          — subscribe to agent event bus
  ├── logging         — structured logging
  ├── state           — resolve state directories
  ├── modelAuth       — provider auth resolution
  ├── webSearch       — web search integration
  └── imageGeneration — image generation integration
```

**Config access is full read/write.** A plugin that calls
`runtime.config.writeConfigFile(...)` can change any setting in `~/.openclaw/openclaw.json`,
including gateway auth tokens, provider keys, and tool policies. This is
intentional by design — auth-setup plugins need it — but it means a malicious
plugin has complete control over the gateway configuration.

**`subagent` is gated to gateway request scope.** It is bound via a
`Symbol.for("openclaw.plugin.gatewaySubagentRuntime")` global set by
`withPluginRuntimeGatewayRequestScope`. Outside a gateway request the call
throws. This prevents skill-context plugins from launching sub-agents, but it
is a soft gate (nothing stops a plugin from checking the global directly).

---

## 4. Hook system

Plugins register hooks via `PluginHookHandlerMap`. The `PluginHookName` union in
`src/plugins/types.ts` defines **26 named hooks**. (`before_skill_install`, discussed in
Part 2, is a design proposal — it is **not present** in the current source and has not
shipped as of this writing.)

| Hook                       | Category            | Exec model               | Reads payload?                   | Write / Modify?                    | Block?                        | Require approval? | Quarantine / Rollback? |
| -------------------------- | ------------------- | ------------------------ | -------------------------------- | ---------------------------------- | ----------------------------- | ----------------- | ---------------------- |
| `before_model_resolve`     | Agent lifecycle     | Modifying                | Yes — model/provider config      | Yes — substitute model or provider | No                            | No                | No                     |
| `before_prompt_build`      | Agent lifecycle     | Modifying                | Yes — prompt being built         | Yes — inject system/user text      | No                            | No                | No                     |
| `before_agent_start`       | Agent lifecycle     | Modifying                | Yes — startup context            | Yes — add startup context          | No                            | No                | No                     |
| `llm_input`                | Agent lifecycle     | Void                     | Yes — full LLM input             | No                                 | No                            | No                | No                     |
| `llm_output`               | Agent lifecycle     | Void                     | Yes — full LLM response          | No                                 | No                            | No                | No                     |
| `agent_end`                | Agent lifecycle     | Void                     | Yes — final agent state          | No                                 | No                            | No                | No                     |
| **`before_tool_call`**     | Tool execution      | **Modifying**            | **Yes — tool name + all params** | **Yes — rewrite params**           | **Yes**                       | **Yes**           | No                     |
| `after_tool_call`          | Tool execution      | Void                     | Yes — tool result                | No                                 | No                            | No                | No                     |
| `message_received`         | Message flow        | Void                     | Yes — inbound message            | No                                 | No                            | No                | No                     |
| `message_sending`          | Message flow        | Modifying                | Yes — outbound message           | Yes — modify content               | Yes — cancel send             | No                | No                     |
| `message_sent`             | Message flow        | Void                     | Yes — sent message               | No                                 | No                            | No                | No                     |
| `inbound_claim`            | Channel routing     | Claiming                 | Yes — inbound message            | Yes — claim routing                | No                            | No                | No                     |
| `before_dispatch`          | Message dispatch    | **Claiming**             | Yes — target + content           | Yes — modify target or content     | **Yes — via `handled: true`** | No                | No                     |
| `session_start`            | Session lifecycle   | Void                     | Yes — session context            | No                                 | No                            | No                | No                     |
| `session_end`              | Session lifecycle   | Void                     | Yes — session summary            | No                                 | No                            | No                | No                     |
| `subagent_spawning`        | Subagent lifecycle  | Modifying                | Yes — spawn params               | Yes — affect spawn                 | No                            | No                | No                     |
| `subagent_delivery_target` | Subagent lifecycle  | Modifying                | Yes — delivery target            | Yes — override target              | No                            | No                | No                     |
| `subagent_spawned`         | Subagent lifecycle  | Void                     | Yes — spawn result               | No                                 | No                            | No                | No                     |
| `subagent_ended`           | Subagent lifecycle  | Void                     | Yes — end state                  | No                                 | No                            | No                | No                     |
| `gateway_start`            | Gateway lifecycle   | Void                     | Yes — gateway config             | No                                 | No                            | No                | No                     |
| `gateway_stop`             | Gateway lifecycle   | Void                     | Yes — shutdown context           | No                                 | No                            | No                | No                     |
| **`before_message_write`** | Message persistence | **Sync**                 | **Yes — transcript entry**       | **Yes — rewrite entry**            | **Yes**                       | No                | No                     |
| `tool_result_persist`      | Tool results        | **Sync**                 | Yes — persisted result           | Yes — rewrite result               | No                            | No                | No                     |
| `before_compaction`        | Compaction          | Void                     | Yes — pre-compact state          | No                                 | No                            | No                | No                     |
| `after_compaction`         | Compaction          | Yes — post-compact state | No                               | No                                 | No                            | No                |
| `before_reset`             | Session reset       | Void                     | Yes — reset context              | No                                 | No                            | No                | No                     |

**Hook execution models:**

Four distinct models govern how hook handlers run:

| Model         | Runner             | Execution              | Async? | Interception                                                            |
| ------------- | ------------------ | ---------------------- | ------ | ----------------------------------------------------------------------- |
| **Void**      | `runVoidHook`      | Parallel (Promise.all) | Yes    | None — return values discarded                                          |
| **Claiming**  | `runClaimingHook`  | Sequential by priority | Yes    | First `{ handled: true }` wins; remaining handlers skipped              |
| **Modifying** | `runModifyingHook` | Sequential by priority | Yes    | Results merged across chain; `block: true` is sticky and short-circuits |
| **Sync**      | inline sync loop   | Sequential             | **No** | Block via `block: true`; async handlers warned and ignored              |

Additional rules:

- Higher `priority` value runs first.
- Errors are caught and logged; they do not propagate to lower-priority handlers.
- For `before_tool_call` (Modifying): once any handler sets `requireApproval`, lower-priority handlers cannot add a second gate — first-set-wins. Param rewrites from lower-priority plugins are frozen once approval is claimed.
- `before_message_write` and `tool_result_persist` are **synchronous only** — they run on the transcript hot path. Do not register async handlers for these hooks.

**Hook registration:**

```typescript
plugin.hooks.register("before_tool_call", {
  priority: 100, // higher runs first
  handler: async (event, ctx) => {
    if (event.toolName === "exec" && isSuspicious(event.params)) {
      return {
        requireApproval: {
          title: "Exec approval needed",
          description: `Command: ${event.params.command}`,
          severity: "warning",
          timeoutBehavior: "deny",
        },
      };
    }
  },
});
```

---

## 5. Plugin discovery and filesystem security

Plugin discovery runs at startup (and on config reload). Key security checks in
`src/plugins/discovery.ts`:

| Check                       | What it detects                                       | Outcome                         |
| --------------------------- | ----------------------------------------------------- | ------------------------------- |
| Symlink escape              | `realpath(source)` is outside `pluginRootDir`         | Block                           |
| Stat failure                | File permissions unreadable                           | Block                           |
| World-writable directory    | `mode & 0o002 !== 0`                                  | Block (auto-repair for bundled) |
| Ownership mismatch          | `stat.uid !== process.getuid()` and not a system path | Block                           |
| GID-writable without sticky | `mode & 0o020 !== 0` and no sticky bit                | Warn                            |

These checks prevent:

- A symlink in `~/.openclaw/plugins/` pointing outside the plugins directory
- A plugin directory that any local user can write to (replacing files between
  discovery and import)
- A plugin file owned by a different UID that could be swapped after the check

**Gap:** The checks happen at discovery time, before `import()`. Between
discovery (stat + ownership check) and the actual `await import(source)` there
is a small time-of-check/time-of-use window. On a multi-user machine a hostile local user could
swap a file after the stat passes. The mitigation is that `mode & 0o002` prevents
world-writable directories, so the only vector is a file owned by the same UID.

**Caching:** The plugin registry is cached with a 1-second TTL (128-entry LRU).
This collapses bursty reloads during startup and config changes, but it also
means a newly installed plugin is visible within 1 second without an explicit
reload command.

---

## 6. Trust model summary

OpenClaw's plugin system is a **trusted-extension model**, not a
sandbox model:

- Plugins run as the same OS process with the same UID.
- No process, memory, network, or filesystem isolation.
- The SDK gives full config read/write access.
- Hooks with modify-or-block power run synchronously in the agent loop.

The security boundary is **install-time**, not runtime. Once a plugin is
installed, it is trusted. A `before_skill_install` hook is the proposed
mechanism for enforcing install-time policy (see Part 2) — it has not yet
shipped. This is a correct design direction for a developer tool where the
operator controls the machine.

---

# Part 2 — Security Hook Analysis and Design Proposals

## 7. `before_skill_install` — design analysis (proposed, not yet shipped)

> **Implementation status:** `before_skill_install` is **not present** in the current
> `PluginHookName` union (`src/plugins/types.ts`). The analysis below documents the
> intended design. Cross-reference `proposal-security-skill-scan-suppression.md` for
> how the existing scanner works in the absence of this hook.

### What it does

`before_skill_install` fires during `installSkill` before a skill's files are
copied to the target directory. The hook handler receives:

```typescript
{
  skillName: string;
  sourceDir: string;         // absolute path — plugin reads files directly
  source?: string;           // "openclaw-bundled" | "workspace" | ...
  builtinFindings: Finding[] // what the built-in scanner already found
}
```

The handler can return:

- `{ block: true, blockReason: "..." }` — prevent installation
- `{ findings: [...] }` — augment the built-in scanner's findings
- `undefined` — no action

The built-in scanner runs **first** and its findings are passed as
`builtinFindings`. This design is correct: a security scanner that cannot read
what it is scanning is useless, and pre-populating findings lets the plugin
avoid duplicating built-in work.

**Error handling:** Hook errors are non-fatal. If all registered `before_skill_install`
hooks throw, installation continues with only the built-in scan findings. This is
a deliberate resilience choice — a broken scanner plugin should not permanently
block all skill installs.

### Strengths

1. **Full source access.** The plugin receives `sourceDir` — it can read any
   file in the skill tree. An AST-level scanner, a YARA scanner, or an LLM-based
   scanner all work without special gateway privileges.
2. **Built-in findings forwarded.** The hook receives what the built-in scanner
   already found. The plugin can escalate severity, add context, or suppress
   false positives based on policy.
3. **Block is terminal.** A `block: true` return from any plugin stops the
   installation before any file copy happens. This is the right semantics —
   block before mutation, not after.
4. **Hook errors non-fatal.** This is correct for resilience; a broken scanner
   plugin should not permanently break skill installation.

### Concerns and proposals

**C1 — No verification of `sourceDir` integrity.**
Between the time `sourceDir` is passed to the hook and when the actual file
copy happens, the directory contents can change (time-of-check/time-of-use race). A hostile skill source
that is a symlink to a benign tree during scan but swapped before copy would
bypass the scanner. Mitigation: the install layer should compute a content hash
of the files it scanned and verify that hash matches the files it actually copies.

_Proposal:_ After the hook runs (and before file copy), compute a SHA-256 hash
of every file in `sourceDir` and store it. After copy, verify the hashes match.
If they diverge, fail the install with `INSTALL_TIME_OF_CHECK_DETECTED`.

**C2 — `builtinFindings` is informational only; cannot suppress.**
A hook cannot downgrade a built-in `critical` finding. If the built-in scanner
fires on legitimate code (false positive), the operator has no way to tell the
gateway to proceed. Currently they would have to disable or modify the built-in
scanner.

_Proposal:_ Add a `suppressBuiltinFinding?: string[]` field to the hook result
(array of `ruleId` strings). Suppressed built-in findings are demoted to `info`
in the final merged list. This gives policy scanners the ability to whitelist known
patterns without disabling the entire built-in scanner.

**C3 — `blockReason` is last-defined wins across hooks.**
If plugin A returns `blockReason: "policy violation"` and plugin B (lower
priority) also blocks with `blockReason: "CVE-2025-1234"`, the final reason
string is B's. This is confusing for debugging.

_Proposal:_ Merge block reasons additively: return an array of
`{ pluginId, reason }` pairs in the install result so operators can see every
plugin that fired.

**C4 — No hook for the npm _download_ phase.**
`before_skill_install` fires after the npm package is fetched and unpacked into a
temp directory. A malicious package with a `postinstall` script runs npm-side
_before_ the hook can block. This is a significant gap for npm-sourced skills.

_Proposal A (near-term):_ Run the built-in scanner on the **tarballed** package
before `npm install` runs. This requires reading the tarball from the npm registry
cache rather than the unpacked directory.

_Proposal B (long-term):_ Add a `before_npm_install` hook that fires with the
resolved package name and version (before any npm side effects). Handlers can
consult a vulnerability database (OSV, npm audit, Snyk) or an internal allowlist
and block before any code runs.

**C5 — No integrity binding between scan and install.**
The hook scan result is in-memory. An adversary with write access to the plugin
state directory could technically swap the installed skill files after the hook
passes. The `.bak` archive pattern used by session management is not applied here.

_Proposal:_ Write a signed manifest of scanned file hashes alongside the installed
skill (`.scan-receipt.json`), signed with an operator key. Subsequent executions
can verify the receipt before loading the skill.

---

## 8. `before_tool_call` + `requireApproval` — design analysis

### What it does

`before_tool_call` fires inside `runBeforeToolCallHook` before any tool executes.
Every tool call in every code path — LLM-initiated, HTTP `POST /tools/invoke`,
and node-dispatched `node.invoke` — calls this function first.

A handler receives:

```typescript
{
  toolName: string;
  params: Record<string, unknown>;
  toolCallId: string;
  agentId?: string;
  sessionKey?: string;
  sessionId?: string;
  runId?: string;
}
```

And can return one of:

- `{ params: {...} }` — silently rewrite params before the tool runs
- `{ block: true, blockReason: "..." }` — hard block
- `{ requireApproval: { title, description, severity, timeoutMs, timeoutBehavior, onResolution } }` — pause and ask user

When `requireApproval` is returned:

1. The gateway generates a `plugin:`-prefixed approval UUID (plugins cannot forge this).
2. The gateway broadcasts an approval request event to connected clients (web UI, Telegram, Discord, CLI).
3. The agent run is suspended awaiting the decision.
4. On resolution, `onResolution(decision)` is called (best-effort, not awaited).
5. `timeoutBehavior` controls what happens if no decision arrives in `timeoutMs`.

### Strengths

1. **Universal coverage.** The hook fires from all three tool dispatch surfaces —
   WS agent, HTTP `/tools/invoke`, and node.invoke. There is no way to call a
   tool that bypasses this hook once a handler is registered.
2. **Params rewriting is transparent.** A plugin can sanitize dangerous arguments
   (strip `--force`, cap file count, redact secrets) without the user knowing.
   This is useful for compliance — block or sanitize without surfacing noise.
3. **Async approval with multiple UIs.** The approval can be resolved from
   wherever the user is — web UI, Telegram, Discord, or CLI `/approve`. This is
   the right UX for an always-on agent: the operator is not always at a terminal.
4. **`pluginId` is stamped by the runner, not the plugin.** A plugin cannot
   claim another plugin's `pluginId` on an approval request. This prevents a
   low-trust plugin from hijacking an approval chain registered by a high-trust
   plugin.
5. **Races `AbortSignal`.** If the agent run is cancelled, the approval wait
   is aborted rather than hanging for `timeoutMs`. This prevents a stalled
   approval from keeping a dead run in memory.
6. **First-block short-circuits.** The chain stops at the first `block: true`.
   Lower-priority plugins do not continue running after a block decision has been
   made.

### Concerns and proposals

**C1 — `requireApproval` owner wins; lower-priority plugins cannot also approve.**
When plugin A (priority 100) sets `requireApproval`, lower-priority plugin B
(priority 50) cannot add a second approval gate. Only the first `requireApproval`
in the chain is honored. A defense-in-depth scenario where two independent
security plugins must both approve is not expressible.

_Proposal:_ Allow multiple `requireApproval` entries to be collected (keyed by
`pluginId`). The tool is blocked until **all** collected approvals resolve. The
UI shows a multi-step approval dialog. Each plugin's approval record is independent.
This enables a two-key / m-of-n approval model for high-risk tools.

**C2 — `timeoutBehavior: "allow"` is a footgun.**
If a plugin registers `before_tool_call` with `timeoutBehavior: "allow"` and
`timeoutMs: 5000`, any tool call for which the user is unreachable for 5 seconds
automatically proceeds. A plugin that is misconfigured or a timeout that is too
short creates a silent allow path.

_Proposal:_ Make `"deny"` the default and emit a gateway log warning whenever
`timeoutBehavior: "allow"` is evaluated (tool was allowed due to no decision).
Consider requiring operators to explicitly opt in via gateway config to allow
`timeoutBehavior: "allow"`.

**C3 — `onResolution` callback is not awaited.**
The `onResolution` callback lets a plugin react to the approval decision (log to
SIEM, update policy, etc.). Because it is not awaited, a plugin that performs
async I/O in `onResolution` (e.g., writing to a database) can race against the
tool execution that follows. If the I/O fails, the failure is silent.

_Proposal:_ Await `onResolution` with a short timeout (e.g., 2 s) before
proceeding. Log and continue if it times out or throws, but do not silently drop
the error.

**C4 — Params rewriting is applied before `requireApproval` too.**
If plugin A modifies `params` and lower-priority plugin B then sets
`requireApproval`, the approval UI shows the _rewritten_ params (from A), not the
original. The user approves a tool call they may not have authorized in its
original form. There is also no way to show the diff between original and rewritten
params in the approval dialog.

_Proposal:_ Pass both `originalParams` and `params` (current after rewrites) in
the `requireApproval` payload. The approval UI should highlight the diff. The user
then approves the _final_ form, making the rewrite visible.

**C5 — No hook for `requireApproval: "allow-always"` persistence.**
When a user chooses `allow-always`, this approval decision is persisted in the
`exec.approvals` store. However, there is currently no hook for a plugin to
observe or veto the _persistence_ of an `allow-always` decision. A plugin that
only wants one-time approvals for a given tool cannot prevent the user from
creating a standing permission.

_Proposal:_ Add a `maxDecision` field to `requireApproval`:

```typescript
requireApproval: {
  title: "...",
  maxDecision: "allow-once"  // "allow-once" | "allow-always" (default)
}
```

When `maxDecision: "allow-once"` is set, the approval UI does not offer the
`allow-always` option.

**C6 — Tool loop detection is a separate path from `requireApproval`.**
The existing tool loop detection (`critical` threshold blocks the tool) runs
_before_ `before_tool_call` hooks. If a plugin's `requireApproval` handler is
registered for the same tool that loop detection blocks, the hook never fires.
The loop block takes effect silently without giving the plugin a chance to observe
and log the event.

_Proposal:_ When loop detection fires, still invoke `before_tool_call` hooks with
a `loopDetected: true` field in the event context. Hooks see the event as
read-only (they cannot unblock a loop detection block), but they can log, alert,
or update external policy.

---

## 9. Architectural property: full access is correct

Both hooks share the same architectural principle: a registered handler gets full
access to its decision point. This raises the obvious concern — what stops a
malicious plugin from using this access destructively?

The answer is the **install-time trust boundary**, and this is the right answer:

1. **Sandboxing hooks is self-defeating.** A security scanner that cannot read the
   files it is scanning is useless. An approval plugin that cannot see the tool
   params cannot write a meaningful approval prompt. Restricting hook access
   to prevent abuse also prevents legitimate security use cases.

2. **The `before_skill_install` hook secures its own surface.** A plugin that
   implements `before_skill_install` can block the installation of other plugins
   that register hooks. This creates a bootstrapping chain: the first security
   plugin installed (possibly by the operator or enterprise IT) can prevent
   subsequent plugins from registering hooks it considers dangerous.

3. **The correct mitigation is strong install-time controls**, not runtime
   sandboxing:
   - `before_skill_install` is the gate for skill installs.
   - Plugin npm integrity checking (`npm install --ignore-scripts` + hash
     verification) is the gate for plugin installs.
   - Discovery-time ownership checks are the gate against local substitution.

4. **What is still missing:** A hook for _plugin_ installation (not just skill
   installation). Today there is no `before_plugin_install` hook equivalent to
   `before_skill_install`. An enterprise security scanner can block a skill but
   cannot block a plugin from being installed via `openclaw plugins install <npm>`.
   This gap is discussed in Section 10 below.

---

## 10. Proposed: `before_plugin_install` hook

The most important missing piece in the current security hook architecture is a
hook for _plugin_ installation. Skills are wrapped by `before_skill_install`, but
plugins — which have dramatically more capability (full SDK access, gateway
handlers, hook registration) — have no equivalent gate.

Proposed design:

```typescript
type PluginHookBeforePluginInstallEvent = {
  npmSpec: string;                // raw npm install spec
  resolvedPackageName: string;    // after resolution
  resolvedVersion: string;
  targetDir: string;
  tarballPath?: string;           // path to cached tarball before extraction
  manifest: PluginManifest;       // parsed openclaw.plugin.json
  declaredHooks: string[];        // hook names the plugin declares in its manifest
  declaredCapabilities: {
    hasConfigWrite: boolean;      // plugin reads/writes config
    hasSubagentRuntime: boolean;  // plugin uses subagent spawning
    hasCoreGatewayHandlers: boolean; // plugin registers gateway RPC methods
    hasToolFactories: boolean;    // plugin registers tools
  };
};

type PluginHookBeforePluginInstallResult = {
  block?: boolean;
  blockReason?: string;
  requireApproval?: { ... };     // same as before_tool_call
};
```

This hook would fire from `src/plugins/install.ts` before `npm install` runs,
receiving the tarball path and the parsed manifest. A security plugin can:

- Consult an npm vulnerability database
- Enforce an enterprise allowlist of approved plugin package names
- Require operator approval for plugins that declare `hasConfigWrite: true`
- Block plugins that declare gateway RPC handler registration without approval

The `declaredCapabilities` fields are computed from the manifest — they do not
require executing any plugin code. This makes the hook safe to run before
installation without a time-of-check/time-of-use risk.

---

## 11. Summary: security hooks as a system

```
Install time (proposed)               Runtime (shipped)
──────────────────────────────────    ──────────────────────────────────────
before_skill_install [NOT YET SHIPPED] before_tool_call + requireApproval
  ↓                                     ↓
Source dir available for scanning     All tool params available before exec
Built-in findings forwarded           Can rewrite, block, or pause for approval
Block is terminal (pre-copy)          Block is terminal (pre-exec)
Non-fatal: scanner errors don't       Non-fatal: hook errors don't block tool
  block all installs                    (fail-open by default)
Gap: npm postinstall runs before hook Gap: multi-approval (first-set-wins only)
Gap: no plugin install hook           Gap: no before_plugin_install
                                      Gap: allow-always veto not supported

Current install-time protection (no hook):
  static scanner in skills-install.ts + audit-extra.async.ts
  — use proposal-security-skill-scan-suppression.md for suppression design
```

The shipped runtime gate (`before_tool_call`) covers all three tool dispatch
surfaces (WS agent, HTTP `/tools/invoke`, `node.invoke`). The install-time gate
(`before_skill_install`) is the most important missing piece — until it ships,
the static scanner is the only install-time defense, with no plugin extensibility.
The architectural choice to give each hook full access to its decision point is
correct. The remaining work is to ship `before_skill_install`, close the
time-of-check/time-of-use gap in skill install, add `before_plugin_install`,
and address the multi-approval
limitation in `before_tool_call`.

---

## Appendix: Hook registration quick reference

```typescript
// In your plugin's entry file:
import type { OpenClawPlugin } from "openclaw/plugin-sdk/core";

const plugin: OpenClawPlugin = {
  id: "my-security-plugin",

  async setup(runtime) {
    // Skill install gate
    runtime.hooks.register("before_skill_install", {
      priority: 100,
      handler: async (event) => {
        const hasDangerousCode = await myScanner.scan(event.sourceDir);
        if (hasDangerousCode) {
          return { block: true, blockReason: "Malicious pattern detected" };
        }
        return { findings: await myScanner.getFindings(event.sourceDir) };
      },
    });

    // Tool call approval gate
    runtime.hooks.register("before_tool_call", {
      priority: 50,
      handler: async (event) => {
        if (event.toolName === "exec" && needsApproval(event.params)) {
          return {
            requireApproval: {
              title: `Exec: ${event.params.command}`,
              description: `Session: ${event.sessionKey}`,
              severity: "warning",
              timeoutMs: 120_000,
              timeoutBehavior: "deny", // always deny on timeout
              onResolution: async (decision) => {
                await myAuditLog.record({ tool: event.toolName, decision });
              },
            },
          };
        }
      },
    });
  },
};

export default plugin;
```
