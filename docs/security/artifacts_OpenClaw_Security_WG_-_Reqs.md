# Feedback Reqs. per OpenClaw

Seeking feedback from partner security teams on three core topics:

1. Req#1 - The foundational security policy and trust model;

2. Req#2 - How to establish trust for security plugins; and

3. Req#3 - Whether the current hook promitives cover what security teams actually need;

Feedback by: 04-11(1week).

- Comment directly in this document ;

- Hook design feedback, so we can restore and ship install-time security hooks with a clear security story

## Background

OpenClaw's plugin system includes hooks that run code at key decision points. Two security relevant hooks exist today:

| Hook                               | PR                                                        | Author       | What it does                                                                                                                                                              |
| ---------------------------------- | --------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| before_install                     | [#56050](https://github.com/openclaw/openclaw/pull/56050) | George Zhang | Run policy checks before any plugin/skill is installed. Handlers receive source path, target metadata, built-in scan results. Can add findings or block the installation. |
| before_tool_call + requireApproval | [#55339](https://github.com/openclaw/openclaw/pull/55339) | Josh Avant   | Runs before a tool call executes. Handlers can require explicit user approval. Supports async approval via UI, Telegra, Discord, CLI                                      |

'**before_install**' was temporarily removed for further security discussion but now added [back](https://github.com/openclaw/openclaw/commit/c22233d96c37f843b48459bc58289b4355b862dc).

'**before_tool_call**' is shipped to [v2026.3.31](https://github.com/openclaw/openclaw/releases/tag/v2026.3.31)

Both hooks share the same architectural property:

A registered handler gets full access to its decision point.

This is the correct design--a security scanner that can't read what it's scanning is useless. But it raises two questions this document aims to address.

## Topic 0: Foundational Trust Model and Security Policy

The working group is specifically seeking feedback on the [tentative security document](https://github.com/openclaw/openclaw/blob/main/SECURITY.md), as they define the trust model and scope for vulnerability reporting. This baseline is critical context for evaluating the security plugin architecture discussed in subsequent topics.

Key Foundational Principles

### Operator Trust Model:

OpeClaw is designed as a "one-user trusted-operator" or "personal assistant" system, not a shared multi-tenant bus[1]. Authenticated Gateway callers are treated as trusted operators, and session identifiers are routing controls, not per-user authorization boundaries. Deployments with multally untrusted/adversarial operators sharing a gateway host are explicitly Out of Scope[2] of the security model.

#### Comments from BD Security team:

| No  | Comments                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | The “trusted single operator” trust model is coherent, but the current product surface makes it easy to drift into **multi-sender / quasi-multi-tenant** usage (e.g., group chat) without the guardrails needed for isolation. We recommend formalizing an explicit **deployment mode** distinction:<br><br>• **Personal mode (default):** session identifiers remain routing controls, not authorization boundaries; group-chat is supported only as a convenience, not a secure multi-user boundary.<br>• **Enterprise mode (opt-in):** introduce a sender identity model and policy scoping so that group contexts do not implicitly grant all participants the same tool and history access. This aligns with the gaps called out in the security working plan (IAM workstream) and the Threat Model Atlas’ “Session Isolation” boundary. |
| 2   | Even under a single-operator model, the platform needs “containment controls” to reduce blast radius when a trusted component is compromised or misbehaves. Minimum required mechanisms:<br><br>• **Runtime disable/quarantine** for installed and bundled plugins (no gateway restart required).<br>• **Fail-closed semantics** for enforcement-tier security controls (clear timeouts and defaults).<br>• **Operator-visible audit trail** of security-relevant decisions (install decisions, tool blocks/approvals, policy changes). These should be treated as core safety primitives, not “enterprise-only” niceties.                                                                                                                                                                                                                    |

#### TODOs

1. Do we need to propose multi-tenant features? It seems quite challenging even though we claim it is a mandatory feature for enterprise use cases, we have not made clear use cases for it and so far not be considered by the OpenClaw part.

### Plugin Trust Boundary

Plugins and extensions are loaded in-process with the Gateway and are considered part of OpenClaw's Trusted Computing Base(TCB). Installing or enabling a plugin grants it the same trust level and OS privileges as the local OpenClaw process. The security model relies on **vendor-level trust**, which frames the discussion in Topic 1.

#### Comments from BD Security team:

| No  | Comments                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Privilege model and enforcement tiers.** Today, plugins/hooks are in-process and can read/write config and mutate runtime state. This is acceptable only if the trust boundary is explicit and enforceable. We recommend defining a tiered trust model (e.g., **Enforcement / Verified / Community**) where only the Enforcement tier may register blocking/approval hooks. Lower tiers may run advisory hooks but cannot override enforcement decisions. This matches the Security WG panel agenda focus on “Advisory vs Enforcement hooks” and “Who can register privileged hooks”. |
| 2   | **Isolation for lower-trust plugins (defense-in-depth).** Full sandboxing of all plugins is not realistic for security-enforcement plugins, but it is valuable for the long tail of general plugins. A pragmatic approach is: (a) keep enforcement plugins in-process (TCB) with strong attestation + policy gate; (b) offer a sandboxed execution option (VM/WASM/subprocess) for non-enforcement plugins, with an allowlisted API surface. This reduces systemic risk without blocking security use cases that require broad visibility.                                              |
| 3   | **Tamper-evident audit logging.** The system should provide an append-only, tamper-evident audit log for: plugin install/uninstall/enable/disable, hook registrations, tool call allow/deny/approval decisions, and credential resolution events. At minimum: signed/hashed event chaining + protected storage location + explicit redaction defaults (tokens, absolute paths).                                                                                                                                                                                                         |
| 4   | **Credential custody and lifecycle.** Static secret injection (`env`/`file`/`exec`) is insufficient for enterprise use. We support the Credential Provider direction: a provider API that supports typed credentials, refresh/rotation/revocation, per-session scoping, and audited resolution paths. Security constraint: credentials must never be written back to config or surfaced to the LLM context.                                                                                                                                                                             |

### Vulnerability Report Scope

Reports are deprioritized or closed if they rely on expected behavior within the trust model. For example, **prompt-injection-only attacks** (without a policy/auth/sandbox boundary bypass) are Out of Scope. Reports that only show malicious behavior from a **trusted-installed/enabled plugin** are also Out of Scope, as this is expected behavior within the trust boundary.

Please checkout out more details in the document itself and we look forward to your feedback.

## Topic 1: Trust Model - Who gets to Run Security Plugins?

### The tension

Security plugins require the highest privilege level to do their job. A '**before_install**' scanner needs to read the full source of what's being installed. A runtime policy needs to see every tool call. This is by design - you can't build a firewall that can't see the traffic.

But if a user installs a malicious plugin disguised as a security tool, that plugin gets the keys to the kingdom. It could sideload code, exfiltrate data, or silently allow malicious installs while appearing to scan them.

This is not unique to OpenClaw. Every security product faces this - TDR agents run at kernel level, WAFs inspect all traffic, antivirus has full filesystem access. The industry solution is vendor-level trust, not sandboxing. You trust CS to run at kernel level because of who they are, not because they are sandboxed.

Possible approaches

| No  | Approaches                    |                                                                                                                                                                                                                          |
| --- | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | Vendor attestation[1]         | Security plugins must be signed by a known vedor(e.g. TEN Keenlab, NT, CS). ClawHub verifies the signature before allowing installation. Trust is binary: verified vendor = full privilege, unknown = blocked or warning |
| 2   | Tiered privilege[2]           | Not all hooks require the same access. Read-only scanning (static analysis) could be sandboxed more tightly than runtime interception. Define privilege tiers and match hooks to tiers.                                  |
| 3   | Curated registry[3]           | Security plugins go through a separate review process on **ClawHub** - code audit by maintainers before listing. Higher bar than regular plugins.                                                                        |
| 4   | Transparency + audit          | Security plugins must declare what they access and log all actions. No sandboxing, but full observability. Users and enterprises can audit after the fact.                                                               |
| 5   | Enterprise policy override[4] | Enterprises set a policy: "only install security plugins from [approved vendor list." Individual users get a warning. This pushes the trust decision to the organization.                                                |

#### Comments from BD Security team:

| No  | Comments                                                                                                                                                                                                                                                                                                                                                                                                             |
| --- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Attestation chain:** we recommend an OpenClaw-operated signing service as an upgrade path, analogous to driver signing. Minimum components: (a) verified publisher identity, (b) signed release artifacts, (c) revocation, and (d) a transparency log for issued attestations. This can coexist with third-party scanning; the signature answers “who shipped this artifact”, scanning answers “what it contains”. |
| 2   | **Tiered privilege is required** independent of the signing mechanism. Signed/attested security plugins should be eligible for an **enforcement tier** with access to enforcement-capable hooks and stable ordering/precedence semantics. Priority should not be the only protection; the platform must also prevent lower-trust plugins from disabling or overriding enforcement plugins.                           |
| 3   | **Registry separation:** agree. Security plugins should have a distinct registry surface (separate listing and review bar) so enterprises can apply different policies for “security enforcement” vs “general productivity” plugins. This can be represented as a separate vendor registry or category with stronger metadata requirements (capabilities, audit endpoints, support SLAs).                            |
| 4   | **Transparency/audit is mandatory for enforcement-tier plugins.** For non-enforcement plugins, transparency is still valuable but can be lighter-weight. For enforcement-tier: require structured audit events for all blocks/approvals/overrides and provide operator tooling to review “why was this allowed/blocked?”.                                                                                            |

### Questions for each team

| Team        | Questions                                                                                                                                                                                                                                                                                                                                                                                                       |                 |
| ----------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- |
| TEN Keenlab | You're building threat detection. What privilege level do you actually need? Could a **read-only** sandbox work for static scanning, or does your approach require deeper access? What does the trust model look like for QClaw's security plugins?                                                                                                                                                             | See 郭的说明[4] |
| BD Security | From the enterprise IAM perspective, 1. How do your customers decide which security tools to trust?[1] 2. What attestation or certification would a Fortune 500 require before deploying an OpenClaw security plugin?[2]                                                                                                                                                                                        | See 郭的说明[3] |
| NT          | You build consumer security products. How does Norton establish trust with end users who can't evaluate code? What model from the consumer AV/EDR world applies here?                                                                                                                                                                                                                                           |                 |
| SlowMist    | From a security perspective, should a sandbox pre-installation test be conducted before installation? Should the installation process be run through first, monitoring the entire workflow, and then accessing overall security? What if this happens after releasing the app to an app store? Should this cover pre-installation detection, in-process monitoring, and post-installation tracking and updates? |                 |

#### Comments from BD Security team:

| No  | Comments                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **Enterprise trust decision model:** enterprises typically do not “trust a tool category”; they enforce an org-wide policy combining: allowlisting (publisher/vendor), risk assessment, artifact integrity (signing), and continuous visibility (audit). OpenClaw should provide centralized policy controls for all plugins/skills, with special handling for enforcement-tier plugins (who can install, who can enable, which tenants/users/agents may use them). Also required is **fine-grained binding** (which agent/session can access which tools/skills/plugins), aligning with the IAM workstream gaps in the working plan. |
| 2   | **Integrity and provenance:** agree with co-signing. Recommended baseline: vendor signs the artifact; OpenClaw signs an attestation over (publisher identity, plugin ID/version, hashes, declared capabilities). This enables revocation and enterprise compliance workflows without forcing all trust onto a single party.                                                                                                                                                                                                                                                                                                           |
| 3   | **Install-time capability consent + verification + UX:**<br>• Require an explicit **capability manifest** (filesystem, network, exec, credential access, inbound message access, config write). Gate installation on operator consent and enterprise policy.<br>• Allow only trusted/attested plugins to register enforcement hooks (policy gate).<br>• Support operator choice to block any plugin install regardless of category.<br>• Provide a robust async scan UX: if a scan completes after install request, the platform must notify the operator and be able to quarantine/disable before first execution.                   |
| 4   | **Trusted execution environment for scanning:** if a sandbox is used for pre-install analysis (e.g., unpack + static scan), the sandbox itself becomes part of the trust model. OpenClaw should (a) document the sandbox boundary, (b) support a configurable list of trusted sandbox backends, and (c) record the sandbox identity/version in scan receipts for auditability.                                                                                                                                                                                                                                                        |

## Topic 2: Hook Design - Do the Primitives Cover Security Needs?

The trust model asks who we trust. This section asks: can the hooks actually do what security teams need?

### Current hook capabilities

'[before_install](https://github.com/openclaw/openclaw/pull/56050/changes)' receives:

- Source path + path kind (file or directory)
- Target name and type (skill or plugin)
- Install request metadata (kind, mode, specifier)
- Built-in scan results (what the default scanner already found)
- Skill-specific metadata (install ID, install spec with package manager details)
- Plugin-specific metadata (content type, package name, version, extensions)

Handlers can return block: true (terminal -stops install) or add findings (accumulated across handlers). Hook errors are non-fatal.

'[before_tool_call](https://github.com/openclaw/openclaw/pull/55339/changes)['](https://github.com/openclaw/openclaw/pull/55339/changes) receives:

- Tool call details (name, arguments)
- Plugin context

Handlers can return requireApproval (pauses for user decision) or block (terminal). Block takes precedence over requireApproval across handlers.

### Design questions

| No  | Questions                                                                                                                                                                                                                                                                                                                                                                                |                                                                                                        |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| 1   | Does 'before_isntall' provide enough context for malware detection? The hook passes source file paths, target metadata, and built-in scan results. Is this sufficient for static analysis and threat signature matching? What additional context would a scanner need - file hashes, dependency trees, network call declarations, permission manifests?                                  | See 郭的说明【1】                                                                                      |
| 2   | Does 'before_tool_call' provide enough context for runtime policy? For enterprise IAM enforcement or runtime safety monitoring - does the hook surface the right information? Is tool name + argument enough, or does it need caller identity, session context, permission scope?                                                                                                        | More context is needed, e.g. Owner id, claw id, session id., etc, See 玉涵补充下细节【2】；            |
| 3   | Are there missing hooks? Possible gaps to evaluate: after_install - post install verification (does installed code match what was scanned?) before_network - egress control (should this plugin be allowed to call this URL?) before_exec - shell command policy (should this command run?) on_permission_escalation - detect when a plugin accesses resources beyond its declared scope | Ran 收集【3】                                                                                          |
| 4   | What privilege level does each use case actually need?                                                                                                                                                                                                                                                                                                                                   | Ran 收集【4】                                                                                          |
| 5   | How do multiple security plugins cojinmpose? If TEN runs an install scanner, Gen runs a runtime monitor, and BD enforces IAM policy - can all three hook handlers coexist? What's the execution order? What if one says allow and another says block? Current behavior: block is terminal, findings accumulate, errors are non-fatal                                                     | We need a mechanism to coordinate this. BD actually builds all them, scanner, runtime monitor, IAM.[5] |

#### Comments from BD Security team:

| No  | Comments                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |     |
| --- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --- |
| 1   | **Req#3-Q1 (before_install payload sufficiency):** recommend adding standardized artifacts to enable deterministic malware/supply-chain analysis:<br>• content hash list (file-level hashes) + package-level hash (archive hash),<br>• dependency manifest extraction (package-lock/pnpm-lock/yarn.lock) and dependency tree summary,<br>• declared capabilities/permissions (a required manifest),<br>• provenance metadata (publisher identity, signed attestation, scan receipt). This aligns with the supply chain workstream gaps and the panel agenda’s “payload minimization & redaction defaults”. |
| 2   | **Req#3-Q2 (before_tool_call context):** tool name + arguments are insufficient for enterprise enforcement and cross-sender safety. The hook should include a consistent **execution context** object:<br>• agentId / assistantId, sessionId, channelId, accountId, senderId, groupId (if any),<br>• pluginId/tool origin, and a stable **traceId** for one conversation turn / tool chain,<br>• optional bounded “recent tool call history” (read-only) to enable STAC detection. This maps directly to the WG agenda’s STAC/history question and IAM workstream.                                         |
| 3   | **Req#3-Q3 (missing hooks / missing primitives):** agree with adding explicit policy hooks for the highest-risk surfaces:<br>• `before_exec` (shell policy) and `before_network`/egress hook (URL policy + SSRF context),<br>• `after_install` / post-install verification (does executed code match scanned artifact?),<br>• provenance tagging (`InputProvenance`) and a unified `traceId` propagated across `before_dispatch → before_prompt_build → before_tool_call → after_tool_call → persistence hooks` to support end-to-end correlation and incident response.                                   |
| 4   | **Req#3-Q4 (approval UX and portability):** approvals should not be channel-dependent. Provide a single approval runtime with consistent semantics across CLI/UI and channel adapters. Where a channel cannot support interactive approval, the default should fail closed (or route to a configured approval fallback).                                                                                                                                                                                                                                                                                   |
| 5   | **Req#3-Q5 (composition and reliability):** define execution semantics for multiple security plugins (ordering, precedence, timeouts). We recommend:<br>• deterministic ordering: priority + stable tie-breaker,<br>• precedence: `block` terminal; `requireApproval` only if not blocked; findings accumulate,<br>• explicit timeouts with documented failure defaults per hook tier (advisory vs enforcement). This mirrors the hook execution semantics described in plugin-security-hooks.md and the WG agenda “Reliability & Composition”.                                                            |
| 6   | **ClawHub scanner guidance:** agree that “normal vs suspicious” needs a public guideline plus a suppression mechanism (with audit trail) to unblock legitimate security skills. This is explicitly called out in the supply chain workstream (false positives) and should be tracked as a deliverable (suppression M1/M2 + scanner rules).                                                                                                                                                                                                                                                                 |
| 7   | **Requested artifact:** yes — we recommend sharing a high-level SDL/pipeline diagram showing how install-time scanning, runtime monitoring, and IAM enforcement compose (including control points and logging). This will help the WG converge on hook primitives and composition semantics.                                                                                                                                                                                                                                                                                                               |

### Questions for each team

| Team        | Questions                                                                                                                                                                                                                                                               |              |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| TEN Keenlab | Walk through a concrete malware detection scenario using the 'before_install' payload. What would you check? Where does the current payload fail short? What' would you need for [the VirusTotal replacement scanner?](https://github.com/openclaw/clawhub/issues/1438) |              |
| BD Security | From the enterprise deployment perspective, What policy hooks would a large organization require before deploying OpenClaw? Which hooks (existing or proposed) map to enterprise compliance requirements (SOC2, ISO 27001, etc.)?                                       | See 郭的说明 |
| NT          |                                                                                                                                                                                                                                                                         |              |
| SlowMist    |                                                                                                                                                                                                                                                                         |              |
