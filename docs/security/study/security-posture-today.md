# Security Posture (today)

This note summarizes OpenClaw’s current security posture as it relates to plugins, hooks/webhooks, and tool execution. It is written as a “what is a security boundary vs what is a guardrail” map for maintainers.

## 1) Core framing: what is and is not a boundary

- Plugins run in-process with the same OS privileges as the Gateway. This is not a sandbox boundary.
- Most of today’s security controls are guardrails against common pitfalls (path traversal, unsafe extraction, accidental unsafe tool execution), plus configurable policy surfaces for operators.

Primary reference:

- `SECURITY.md`

## 2) Install-time guardrails

Install-time controls focus on making installs safer and more predictable, but do not provide a guarantee that installed code is safe.

### 2.1 Safe install paths and ID validation

Goals:

- Prevent path traversal and “install outside the managed directory” bugs.
- Normalize plugin IDs and enforce canonical install locations.

Where to read:

- `src/plugins/install.ts`

### 2.2 Safe archive extraction and filesystem safety

Goals:

- Prevent zip-slip and destination traversal.
- Detect and prevent symlink escape patterns.
- Apply size budgets and structural checks for archives.

Where to read:

- `src/infra/archive.ts`
- `src/infra/fs-safe.ts`

### 2.3 Best-effort security scanning during install

Goal:

- Provide warnings/findings based on heuristic scanning of install sources.

Notes:

- This is best-effort and primarily a warning surface, not a hard sandbox or formal verification step.

Where to read:

- `src/plugins/install-security-scan.runtime.ts`
- Call sites: `src/plugins/install.ts`

### 2.4 Skills installer hardening

Goals:

- Constrain spec strings used to install skills to reduce injection risk.
- For Node installs, avoid lifecycle scripts (`--ignore-scripts`) as a safety default.

Where to read:

- `src/agents/skills-install.ts`

## 3) Runtime loading and operator trust controls (plugins)

These controls help operators avoid unintentionally running untrusted code.

### 3.1 Allow/deny lists

Goals:

- Support explicit trust pinning (allowlist) and explicit blocking (denylist).

Where to read:

- Config validation: `src/config/validation.ts`
- Enforcement and warnings: `src/plugins/loader.ts`

### 3.2 Provenance warnings for untracked local code

Goal:

- Flag locally discovered plugins that are not clearly installed/managed, to encourage explicit trust decisions.

Where to read:

- `src/plugins/loader.ts`

## 4) Hooks / webhooks: treat ingress as untrusted

OpenClaw has multiple “hook-like” surfaces. A key security principle is that inbound HTTP/webhook ingress is untrusted and must be authenticated and safely parsed.

### 4.1 Token separation

Goal:

- Ensure webhook/hook authentication is not accidentally coupled to Gateway auth tokens.

Where to read:

- `src/gateway/startup-auth.ts`

### 4.2 Hooks mapping / transforms path safety

Goals:

- Restrict transforms to an allowed directory.
- Block traversal and symlink escape on existing path segments.

Where to read:

- `src/gateway/hooks-mapping.ts`

### 4.3 Template evaluation hardening

Goals:

- Prevent prototype-chain traversal in templating (`__proto__`, `constructor`, `prototype`) to reduce prototype pollution / data exfil patterns.

Where to read:

- `src/gateway/hooks-mapping.ts`

## 5) Tool execution controls: policy + approvals + allowlists

These controls aim to reduce the risk that an agent run triggers dangerous host actions by accident. They are not per-user authentication boundaries; they are operator safety interlocks.

### 5.1 Tool allowlist policy pipeline

Goal:

- Allow operators to restrict which tools can be called (including plugin-provided tools), with normalization and warnings for unknown entries.

Where to read:

- `src/agents/tool-policy.ts`
- `src/agents/tool-policy-pipeline.ts`

### 5.2 Exec approvals and allowlists

Goals:

- Enforce local allowlists and/or explicit user approval before host command execution.
- Use safe-bin policy and resolved-path pinning to mitigate PATH shadowing.
- Bind approvals to execution context (argv/cwd/env) and, when possible, to a concrete file operand.

Where to read:

- `src/infra/exec-approvals-allowlist.ts`
- `src/infra/exec-safe-bin-runtime-policy.ts`
- `src/node-host/invoke-system-run-allowlist.ts`

## 6) Plugin runtime auth boundaries (plugin → gateway)

Some gateway method surfaces require explicit scope/authorization. The plugin runtime uses request-scoped context when possible, and avoids implicitly granting admin-only capabilities to unauthenticated plugin-owned entrypoints.

Where to read:

- `src/gateway/server-plugins.ts`
- `src/plugins/runtime/gateway-request-scope.ts`

## 7) Sandboxing surfaces (when used)

OpenClaw includes sandbox-related safety checks (especially around filesystem mounts and bridge paths) intended to prevent common escape patterns when a sandbox runner is configured.

Where to read:

- `src/agents/sandbox/validate-sandbox-security.ts`
- `src/agents/sandbox/fs-bridge-path-safety.ts`

## 8) Auditing and operator visibility

OpenClaw has a security audit surface that checks for risky configurations and supply-chain related issues (for example permissive tool policy or unpinned installs).

Where to read:

- `src/security/audit-extra.async.ts`

## 9) Implications for Security WG discussions

When proposing “security plugins” and “security hooks”, it helps to classify each hook as either:

- Advisory (findings/warnings; best-effort acceptable), or
- Enforcement-capable (must be reliable; requires explicit trust gating and clear fail-open vs fail-closed semantics).

Because plugins run with high privilege, a practical trust story usually requires some combination of:

- Publisher identity / signing / revocation
- Enterprise policy controls for privileged hook registration
- Payload minimization by default, with capability-gated access to sensitive fields
