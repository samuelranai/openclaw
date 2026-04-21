---
title: "Technical Design — Universal Confirmation Gate and Tool Result Rewrite"
summary: "Technical design for two requirements: (1) a channel-agnostic second-confirmation mechanism that replaces requireApproval's gateway RPC dependency; (2) modifying tool result content before the LLM analyzes it using tool_result_persist + before_prompt_build"
read_when:
  - Designing a universal tool approval flow that works across all channels
  - Understanding why requireApproval is channel-dependent and what to use instead
  - Implementing tool result content enrichment + LLM analysis instruction injection
  - Building a security plugin that needs both confirmation and result rewriting
---

# Technical Design — Universal Confirmation Gate and Tool Result Rewrite

This document covers two related requirements raised by security plugin authors:

1. **Universal confirmation gate** — `requireApproval` depends on each channel having
   an approval UI registered at the gateway level; a more universal mechanism is needed.

2. **Tool result rewrite before LLM analysis** — when not intercepting (i.e. the tool
   runs to completion), the returned content should be modifiable and the LLM should
   receive analysis instructions alongside it.

---

## Part 1 — Universal Confirmation Gate

### 1.1 Why `requireApproval` is channel-dependent

The current `requireApproval` field in `PluginHookBeforeToolCallResult` works via a
two-phase gateway RPC
([src/agents/pi-tools.before-tool-call.ts:227-332](../../../../../src/agents/pi-tools.before-tool-call.ts)):

```
before_tool_call returns { requireApproval: { title, description, ... } }
        │
        ▼
callGatewayTool("plugin.approval.request", ...)   ← gateway RPC
        │
        ▼
Gateway looks for a registered approval handler for this channel
        │
        ├── handler found  → shows channel-specific UI (button, dialog, inline reply)
        │                     waits for "plugin.approval.waitDecision"
        │
        └── no handler     → decision === null → auto-blocks: "no approval route"
```

The failure mode is explicit in the source
([src/agents/pi-tools.before-tool-call.ts:265-270](../../../../../src/agents/pi-tools.before-tool-call.ts)):

```typescript
if (decision === null) {
  safeOnResolution(PluginApprovalResolutions.CANCELLED);
  return {
    blocked: true,
    reason: "Plugin approval unavailable (no approval route)",
  };
}
```

This means:

- Channels without a registered approval handler silently auto-block — no confirmation, just rejection.
- The approval UX is entirely channel-specific: Telegram shows a different dialog than Discord or the web UI.
- Plugin authors cannot control the confirmation experience across channels.

---

### 1.2 Design: Conversation-Level Approval Gate

**Core idea:** instead of delegating to the channel's UI layer (gateway RPC), implement
the confirmation at the **conversation layer** — the agent itself asks the user, the
user replies, and the plugin intercepts that reply to gate the next tool call.

This works on **every channel** because it uses the message flow, not channel UI.

#### State machine

```
State: IDLE
  │
  │  before_tool_call detects tool needing confirmation
  ▼
State: PENDING_APPROVAL { sessionKey, toolName, params, timestamp }
  │
  │  before_tool_call returns { block: true, blockReason: "PENDING_CONFIRM" }
  │  before_prompt_build injects confirmation request directive
  │  LLM generates: "I need to run X. Please reply 'confirm' or 'cancel'."
  │
  │  User replies "confirm" / "yes" / "proceed"
  │
  │  before_dispatch detects confirmation → sets State: APPROVED
  │  Returns { handled: false } → normal dispatch continues
  │
  ▼
State: APPROVED { sessionKey, toolName, params }
  │
  │  Agent runs again (same session, user context)
  │  LLM re-issues the same tool call (guided by conversation history)
  │
  │  before_tool_call sees APPROVED state for this tool → allows
  │  Clears APPROVED state after allowing (one-shot)
  │
  ▼
State: IDLE
```

#### Implementation

```typescript
import type { OpenClawPluginApi } from "openclaw/plugin-sdk/core";

// ── Shared state ──────────────────────────────────────────────────────────

type PendingApproval = {
  toolName: string;
  params: Record<string, unknown>;
  requestedAt: number;
  expiresAt: number; // auto-expire stale pending approvals
};

// Keyed by sessionKey — one pending approval per session at a time
const pendingApprovals = new Map<string, PendingApproval>();
// Keyed by sessionKey — one approved slot per session at a time
const approvedCalls = new Map<string, { toolName: string; params: Record<string, unknown> }>();

// How long to wait for confirmation before the pending approval expires
const APPROVAL_TTL_MS = 5 * 60 * 1000; // 5 minutes

// Tools that require confirmation before running
const CONFIRM_REQUIRED_TOOLS = new Set(["bash", "execute_command", "shell", "write_file"]);

// Patterns that count as user confirmation
const CONFIRM_PATTERNS = /^\s*(yes|confirm|proceed|ok|go ahead|允许|确认|继续)\s*[.!]?\s*$/i;
// Patterns that count as user denial
const DENY_PATTERNS = /^\s*(no|cancel|stop|deny|abort|不|取消|拒绝)\s*[.!]?\s*$/i;

function isParamsMatch(a: Record<string, unknown>, b: Record<string, unknown>): boolean {
  // Shallow equality sufficient for tool call deduplication
  return JSON.stringify(a) === JSON.stringify(b);
}

function isPendingExpired(pending: PendingApproval): boolean {
  return Date.now() > pending.expiresAt;
}

// ── Plugin ────────────────────────────────────────────────────────────────

export function register(api: OpenClawPluginApi) {
  // ── Hook 1: before_tool_call ─────────────────────────────────────────────
  // Gate tool calls that require confirmation.
  // Three outcomes:
  //   a) Tool not in confirm list          → allow immediately
  //   b) APPROVED state for this call      → allow + clear state
  //   c) No approval yet                   → block + arm PENDING state

  api.registerHook(
    "before_tool_call",
    async (event, ctx) => {
      const { toolName, params } = event as {
        toolName: string;
        params: Record<string, unknown>;
      };
      const sessionKey =
        (ctx as { sessionKey?: string }).sessionKey ??
        (event as { sessionKey?: string }).sessionKey;
      if (!sessionKey) return undefined;

      // Not a tool that needs confirmation
      if (!CONFIRM_REQUIRED_TOOLS.has(toolName)) return undefined;

      // Check if this specific call was approved
      const approved = approvedCalls.get(sessionKey);
      if (approved && approved.toolName === toolName && isParamsMatch(approved.params, params)) {
        // Consume the approval — one-shot
        approvedCalls.delete(sessionKey);
        api.logger.info(`[confirm-gate] approved call consumed: ${toolName} (${sessionKey})`);
        return undefined; // allow
      }

      // Check if there is a stale pending approval to clean up
      const existing = pendingApprovals.get(sessionKey);
      if (existing && isPendingExpired(existing)) {
        pendingApprovals.delete(sessionKey);
      }

      // Block and set PENDING state
      const pending: PendingApproval = {
        toolName,
        params,
        requestedAt: Date.now(),
        expiresAt: Date.now() + APPROVAL_TTL_MS,
      };
      pendingApprovals.set(sessionKey, pending);
      api.logger.warn(
        `[confirm-gate] blocking ${toolName} — awaiting user confirmation (${sessionKey})`,
      );

      const preview = JSON.stringify(params).slice(0, 200);
      return {
        block: true,
        blockReason:
          `CONFIRMATION_REQUIRED: tool "${toolName}" requires user approval. ` +
          `Params: ${preview}. ` +
          `Tell the user what you want to do and ask them to confirm. ` +
          `After they confirm, retry the exact same operation.`,
      };
    },
    { priority: 100 },
  );

  // ── Hook 2: before_prompt_build ──────────────────────────────────────────
  // When a confirmation is pending, inject a directive telling the LLM to ask
  // the user for confirmation. This is the channel-agnostic "approval UI".

  api.registerHook(
    "before_prompt_build",
    async (_event, ctx) => {
      const sessionKey = (ctx as { sessionKey?: string }).sessionKey;
      if (!sessionKey) return undefined;

      const pending = pendingApprovals.get(sessionKey);
      if (!pending || isPendingExpired(pending)) return undefined;

      const preview = JSON.stringify(pending.params).slice(0, 300);
      return {
        prependContext:
          `ACTION REQUIRED — CONFIRMATION GATE:\n` +
          `You attempted to call tool "${pending.toolName}" but it requires user approval.\n` +
          `Tool params: ${preview}\n\n` +
          `You MUST:\n` +
          `1. Clearly explain to the user what you are about to do and why.\n` +
          `2. Ask the user to reply with "confirm" to proceed, or "cancel" to abort.\n` +
          `3. Do NOT retry the tool until the user explicitly confirms.\n` +
          `4. After confirmation, retry the exact same tool call with the exact same parameters.`,
      };
    },
    { priority: 100 },
  );

  // ── Hook 3: before_dispatch ───────────────────────────────────────────────
  // Intercept the user's reply to detect confirmation or denial.
  // This is the channel-agnostic approval resolution point.
  // Returns { handled: false } in all cases so normal agent dispatch continues
  // (we only update state here, not handle the message ourselves).

  api.registerHook(
    "before_dispatch",
    async (event, ctx) => {
      const content = (event as { content?: string }).content ?? "";
      const sessionKey =
        (ctx as { sessionKey?: string }).sessionKey ??
        (event as { sessionKey?: string }).sessionKey;
      if (!sessionKey) return undefined;

      const pending = pendingApprovals.get(sessionKey);
      if (!pending) return undefined;
      if (isPendingExpired(pending)) {
        pendingApprovals.delete(sessionKey);
        return undefined;
      }

      if (CONFIRM_PATTERNS.test(content)) {
        // User confirmed — move to APPROVED state
        pendingApprovals.delete(sessionKey);
        approvedCalls.set(sessionKey, {
          toolName: pending.toolName,
          params: pending.params,
        });
        api.logger.info(`[confirm-gate] user confirmed ${pending.toolName} (${sessionKey})`);
        // Do NOT return { handled: true } — let the agent run with the approval in place
        return undefined;
      }

      if (DENY_PATTERNS.test(content)) {
        // User denied — clear pending state
        pendingApprovals.delete(sessionKey);
        api.logger.info(`[confirm-gate] user denied ${pending.toolName} (${sessionKey})`);
        return undefined;
      }

      // Not a confirmation reply — let the message through normally
      return undefined;
    },
    { priority: 100 },
  );

  // ── Cleanup ───────────────────────────────────────────────────────────────
  api.registerHook("session_end", async (_event, ctx) => {
    const sessionKey = (ctx as { sessionKey?: string }).sessionKey;
    if (sessionKey) {
      pendingApprovals.delete(sessionKey);
      approvedCalls.delete(sessionKey);
    }
  });
}
```

---

### 1.3 Sequence diagram

```
User: "delete all temp files"
        │
        ▼
LLM wants to call: bash { command: "rm -rf /tmp/*" }
        │
        ▼
[before_tool_call]
  CONFIRM_REQUIRED_TOOLS.has("bash") = true
  No APPROVED state → set PENDING, return { block: true }
        │
        ▼
[before_prompt_build] (next LLM turn, same agent run)
  PENDING state found → prependContext: "CONFIRMATION GATE: explain and ask user"
        │
        ▼
LLM reply: "I want to run 'rm -rf /tmp/*' to delete temp files.
            Please reply 'confirm' to proceed or 'cancel' to abort."
        │
        ▼
User: "confirm"
        │
        ▼
[before_dispatch]
  CONFIRM_PATTERNS matches → PENDING → APPROVED, return undefined (let through)
        │
        ▼
Agent runs (new turn, same session, "confirm" message in context)
        │
        ▼
LLM re-issues: bash { command: "rm -rf /tmp/*" }
        │
        ▼
[before_tool_call]
  APPROVED state matches toolName + params → consume approval → allow
        │
        ▼
Tool executes normally
```

---

### 1.4 Comparison with `requireApproval`

| Aspect                           | `requireApproval` (current)                  | Conversation-level gate (this design)    |
| -------------------------------- | -------------------------------------------- | ---------------------------------------- |
| Channel support required         | Yes — channel must register approval handler | No — works on any channel                |
| No handler fallback              | Auto-blocks silently                         | N/A — always works                       |
| UX consistency                   | Per-channel (button vs inline vs dialog)     | Consistent — always a conversation reply |
| Async approval (separate person) | Possible if channel supports it              | Possible via same conversation           |
| Approval timeout                 | Configurable `timeoutMs`                     | Configurable `APPROVAL_TTL_MS`           |
| Works in CLI/web/mobile          | Depends on implementation                    | Always                                   |
| Params passed to handler         | Yes (`approvalParams` in result)             | Yes (via APPROVED state params)          |
| Implementation complexity        | Framework-level (gateway RPC)                | Plugin-level (conversation state)        |

---

### 1.5 Limitations and mitigations

**LLM may not reliably re-issue the exact tool call.**
Mitigation: the `before_prompt_build` injection explicitly instructs "retry the exact
same tool call with the exact same parameters." For critical operations, the plugin can
also store the params in `approvedCalls` and match loosely (e.g. just by `toolName`).

**Confirmation patterns may produce false positives.**
A user saying "yes I understand" in unrelated context could trigger an approval.
Mitigation: narrow `CONFIRM_PATTERNS` to exact words, check that content is short
(single-word replies are more likely to be confirmations), and check timing relative
to the pending approval's `requestedAt`.

**Multiple pending approvals across parallel sessions.**
The Map is keyed by `sessionKey` — one pending approval per session. Parallel sessions
are independent.

---

## Part 2 — Tool Result Rewrite Before LLM Analysis

### 2.1 The requirement

After a tool runs (no interception — tool result is kept), the plugin should:

1. Modify or enrich the tool result content before the LLM sees it
2. Inject analysis instructions so the LLM knows how to reason about the enriched result

This is different from interception (which replaces content with `[BLOCKED]`). Here
the content is preserved but transformed, and the LLM receives a tailored analysis task.

---

### 2.2 Two-layer design

```
Tool runs → produces raw result
        │
        ▼
[tool_result_persist] (SYNC) — Layer 1: Content Transform
        │  Annotates or restructures the raw content.
        │  The annotated version is what gets stored in session.
        │  The LLM sees this annotated version.
        │
        ▼
Session transcript updated
        │
        ▼
[before_prompt_build] (async) — Layer 2: Analysis Instructions
        │  Injects analysis task + criteria into the LLM's context.
        │  The LLM receives both the annotated result AND these instructions.
        │
        ▼
LLM call — analyzes enriched result with specific instructions
```

---

### 2.3 Layer 1: Content transform in `tool_result_persist`

```typescript
// MUST be synchronous — no async, no await, no network calls.

api.registerHook(
  "tool_result_persist",
  (event, ctx) => {
    const toolName = (ctx as { toolName?: string }).toolName ?? "unknown";
    const content = event.message.content;
    if (typeof content !== "string") return undefined;

    // Enrich the content with structured annotations
    const enriched = enrichToolResult(toolName, content);
    if (!enriched) return undefined;

    return {
      message: {
        ...event.message,
        content: enriched,
      } as typeof event.message,
    };
  },
  { priority: 100 },
);

function enrichToolResult(toolName: string, rawContent: string): string | null {
  const lines: string[] = [];
  let modified = false;

  // Tag the source
  lines.push(`[tool:${toolName}]`);

  // Add structural markers for known content patterns
  if (rawContent.includes("error") || rawContent.includes("Error")) {
    lines.push(`[contains-errors:yes]`);
    modified = true;
  }

  if (rawContent.length > 5_000) {
    lines.push(`[size:large content=${rawContent.length}chars]`);
    modified = true;
  }

  // PII detection markers (sync regex only)
  const piiTypes: string[] = [];
  if (/\b\d{3}-\d{2}-\d{4}\b/.test(rawContent)) piiTypes.push("ssn");
  if (/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b/.test(rawContent)) piiTypes.push("email");
  if (/(AKIA[0-9A-Z]{16}|sk-[a-zA-Z0-9]{32,})/i.test(rawContent)) piiTypes.push("credential");
  if (piiTypes.length > 0) {
    lines.push(`[pii-detected:${piiTypes.join(",")}]`);
    modified = true;
  }

  if (!modified) return null; // no enrichment needed

  lines.push(""); // blank separator
  lines.push(rawContent);
  return lines.join("\n");
}
```

---

### 2.4 Layer 2: Analysis instructions in `before_prompt_build`

```typescript
type AnalysisDirective = {
  toolName: string;
  annotations: string[]; // extracted from the enriched content tags
};

const pendingAnalysis = new Map<string, AnalysisDirective>();

// In after_tool_call: extract annotations synchronously (before first await)
api.registerHook(
  "after_tool_call",
  async (event, ctx) => {
    const { toolName, result } = event as { toolName: string; result?: unknown };
    const runId = (ctx as { runId?: string }).runId ?? (event as { runId?: string }).runId;
    if (!runId) return;

    const text = typeof result === "string" ? result : JSON.stringify(result ?? "");

    // Sync annotation detection (same set as tool_result_persist)
    const annotations: string[] = [];
    if (/\b\d{3}-\d{2}-\d{4}\b/.test(text)) annotations.push("ssn");
    if (/\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b/.test(text)) annotations.push("email");
    if (/(AKIA[0-9A-Z]{16}|sk-[a-zA-Z0-9]{32,})/i.test(text)) annotations.push("credential");
    if (text.length > 5_000) annotations.push("large-result");
    if (text.includes("error") || text.includes("Error")) annotations.push("contains-errors");

    if (annotations.length > 0) {
      pendingAnalysis.set(runId, { toolName, annotations });
    }
  },
  { priority: 100 },
);

// In before_prompt_build: inject analysis task based on what was detected
api.registerHook(
  "before_prompt_build",
  async (_event, ctx) => {
    const runId = (ctx as { runId?: string }).runId;
    if (!runId) return undefined;

    const directive = pendingAnalysis.get(runId);
    if (!directive) return undefined;

    const instructions: string[] = [`ANALYSIS TASK for tool "${directive.toolName}" result:`];

    if (directive.annotations.includes("credential")) {
      instructions.push(
        "- CREDENTIAL DETECTED: Summarize findings without quoting credential values. " +
          "Do not include raw credentials in your response.",
      );
    }
    if (directive.annotations.includes("ssn") || directive.annotations.includes("email")) {
      instructions.push(
        `- PII DETECTED (${directive.annotations.filter((a) => ["ssn", "email"].includes(a)).join(", ")}): ` +
          "Refer to individuals by role or anonymized placeholder, not by their data.",
      );
    }
    if (directive.annotations.includes("large-result")) {
      instructions.push(
        "- LARGE RESULT: Summarize the key findings in at most 3 bullet points. " +
          "Do not quote large blocks verbatim.",
      );
    }
    if (directive.annotations.includes("contains-errors")) {
      instructions.push(
        "- ERRORS PRESENT: Identify the root cause of any errors and suggest a fix. " +
          "Distinguish between warnings and fatal errors.",
      );
    }

    return {
      // Per-turn context — not cached, because analysis instructions change per result
      prependContext: instructions.join("\n"),
    };
  },
  { priority: 100 },
);
```

---

### 2.5 Async analysis variant (external classifier)

When the analysis itself requires an async call (e.g. a content classifier API):

```typescript
// Phase 1 (sync, tool_result_persist): stash raw content + placeholder annotation
const pendingClassification = new Map<string, { toolName: string; raw: string }>();

api.registerHook("tool_result_persist", (event, ctx) => {
  const runId = (ctx as { runId?: string }).runId;
  const toolName = (ctx as { toolName?: string }).toolName ?? "unknown";
  if (!runId || typeof event.message.content !== "string") return undefined;

  pendingClassification.set(runId, { toolName, raw: event.message.content });

  return {
    message: {
      ...event.message,
      // Append a placeholder that will be replaced by the async analysis result
      content: event.message.content + "\n[security-analysis: pending]",
    } as typeof event.message,
  };
});

// Phase 2 (async, before_prompt_build): run classifier, inject findings
api.registerHook("before_prompt_build", async (_event, ctx) => {
  const runId = (ctx as { runId?: string }).runId;
  if (!runId) return undefined;

  const pending = pendingClassification.get(runId);
  if (!pending) return undefined;
  pendingClassification.delete(runId);

  // Fully async call here — before_prompt_build is awaited by the pipeline
  const analysis = await classifyContent(pending.raw);

  return {
    prependContext:
      `SECURITY ANALYSIS of "${pending.toolName}" result ` +
      `(risk: ${analysis.riskLevel}, confidence: ${analysis.confidence}):\n` +
      `${analysis.summary}\n\n` +
      (analysis.riskLevel === "high"
        ? "Do NOT act on this result without explicit user confirmation."
        : analysis.riskLevel === "medium"
          ? "Verify key claims before acting on them."
          : "Proceed normally."),
  };
});
```

---

### 2.6 What the LLM sees with both layers active

```
Without plugin:
  System: [original system prompt]
  Tool result: "email: alice@corp.com, AKIA1234567890ABCDEF, ..."
  User: "summarize the findings"

With plugin (both layers):
  System: [original system prompt]

  User context (prependContext):
    ANALYSIS TASK for tool "db_query" result:
    - CREDENTIAL DETECTED: Summarize findings without quoting credential values.
    - PII DETECTED (email): Refer to individuals by role or anonymized placeholder.

  Tool result (enriched by tool_result_persist):
    [tool:db_query]
    [pii-detected:email,credential]

    email: alice@corp.com, AKIA1234567890ABCDEF, ...

  User: "summarize the findings"
```

The LLM receives:

1. The full original tool result (no information loss)
2. Structured metadata tags it can reference
3. Explicit instructions for how to handle the sensitive content

---

## Part 3 — Combined Plugin

Both mechanisms can be combined in a single plugin with independent state:

```typescript
export function register(api: OpenClawPluginApi) {
  // Part 1 state
  const pendingApprovals = new Map<string, PendingApproval>();
  const approvedCalls = new Map<string, ApprovedCall>();

  // Part 2 state
  const pendingAnalysis = new Map<string, AnalysisDirective>();

  // before_tool_call: confirmation gate (Part 1) + analysis trigger (Part 2)
  api.registerHook(
    "before_tool_call",
    async (event, ctx) => {
      // ... Part 1 logic
    },
    { priority: 100 },
  );

  // before_dispatch: resolve confirmation replies (Part 1)
  api.registerHook(
    "before_dispatch",
    async (event, ctx) => {
      // ... Part 1 logic
    },
    { priority: 100 },
  );

  // after_tool_call: detect annotations for analysis (Part 2, sync detection)
  api.registerHook(
    "after_tool_call",
    async (event, ctx) => {
      // ... Part 2 annotation detection
    },
    { priority: 100 },
  );

  // tool_result_persist: enrich content (Part 2)
  api.registerHook(
    "tool_result_persist",
    (event, ctx) => {
      // ... Part 2 content enrichment
    },
    { priority: 100 },
  );

  // before_prompt_build: inject confirmation directive (Part 1) OR analysis task (Part 2)
  api.registerHook(
    "before_prompt_build",
    async (_event, ctx) => {
      const sessionKey = (ctx as { sessionKey?: string }).sessionKey;
      const runId = (ctx as { runId?: string }).runId;

      // Part 1 takes priority if a confirmation is pending
      if (sessionKey) {
        const pending = pendingApprovals.get(sessionKey);
        if (pending && !isPendingExpired(pending)) {
          return buildConfirmationDirective(pending);
        }
      }

      // Part 2: inject analysis instructions
      if (runId) {
        const directive = pendingAnalysis.get(runId);
        if (directive) return buildAnalysisDirective(directive);
      }

      return undefined;
    },
    { priority: 100 },
  );

  // Cleanup
  api.registerHook("session_end", async (_event, ctx) => {
    const sessionKey = (ctx as { sessionKey?: string }).sessionKey;
    if (sessionKey) {
      pendingApprovals.delete(sessionKey);
      approvedCalls.delete(sessionKey);
    }
  });
}
```

---

## Summary

| Requirement                       | Solution                                            | Key hooks                                                      | Works without channel support? |
| --------------------------------- | --------------------------------------------------- | -------------------------------------------------------------- | ------------------------------ |
| Universal confirmation gate       | Conversation-level state machine                    | `before_tool_call` + `before_dispatch` + `before_prompt_build` | Yes                            |
| Modify result before LLM analysis | Two-layer content transform + instruction injection | `tool_result_persist` (sync) + `before_prompt_build` (async)   | Yes                            |
| Combined                          | Both in one plugin, independent state               | All five hooks                                                 | Yes                            |
