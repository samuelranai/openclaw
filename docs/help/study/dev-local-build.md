---
title: "Local Dev Build"
summary: "Build and run OpenClaw from source on macOS — prerequisites, install, dev gateway, debugging, and reset"
read_when:
  - Building OpenClaw from source for the first time
  - Setting up a local gateway for development or study
  - Debugging gateway behavior with breakpoints or logs
---

# Local Dev Build

This guide walks through building OpenClaw from a cloned source tree and running
it locally on macOS. It is intended for developers who want to study the code,
set breakpoints, and run the gateway against a real LLM key without installing
from npm.

---

## Prerequisites

### Node.js

OpenClaw requires **Node 22.14 or newer**. Node 24 is recommended.

Check your current version:

```bash
node -v
```

If it is below v22.14, upgrade via Homebrew:

```bash
brew install node
# or to pin a specific version:
brew install node@24
```

### pnpm

The repo uses pnpm as its package manager. The required version is pinned in
`package.json` at the repo root (`"packageManager": "pnpm@10.32.1"`). Any
10.x release works for local development.

If `npm install -g` fails with SSL errors (common on corporate networks), use
the pnpm standalone installer instead:

```bash
curl -fsSL https://get.pnpm.io/install.sh | sh -
source ~/.bashrc
```

Verify:

```bash
pnpm -v
# 10.32.x or newer is fine
```

### Bun

Bun is used for running TypeScript build scripts. It is not used at runtime.

```bash
curl -fsSL https://bun.sh/install | bash
```

On macOS the installer adds `~/.bun/bin` to `~/.bash_profile`. Source it to
activate bun in the current shell:

```bash
source ~/.bash_profile
```

Verify:

```bash
bun -v
```

---

## Install Dependencies

From the repo root:

```bash
cd /path/to/openclaw
pnpm install
```

This installs all workspace packages (root, `extensions/*`, `packages/*`, `ui/`).
First run downloads native binaries (~150 MB on arm64) and takes 1–2 minutes.
Subsequent runs are fast unless `pnpm-lock.yaml` changes.

> If you see `ERR_PNPM_WORKSPACE_PKG_NOT_FOUND` or similar workspace errors,
> make sure you are running `pnpm install` from the repo root, not a subdirectory.

---

## Build

```bash
pnpm build
```

This runs the full build pipeline:

1. Bundle UI assets (`canvas:a2ui:bundle`)
2. Compile TypeScript → `dist/` via tsdown
3. Run post-build scripts (build stamp, plugin-sdk type stubs, build info)
4. Generate CLI startup metadata and compat shims

Build takes ~1 minute on first run.

> If the build fails with `bun: command not found` or `tsx: command not found`,
> make sure bun is on your PATH (`source ~/.bash_profile`) and re-run
> `pnpm install`.

### Verify the build

```bash
node dist/index.js --version
# expected: OpenClaw 2026.x.x (commit)
```

---

## Configure an LLM Provider

### How provider auth works

For built-in providers (OpenAI, Anthropic), **do not set `models.providers.*`
in the config**. That path is for custom OpenAI-compatible endpoints and requires
a full schema including a `models` array — setting only `apiKey` or `baseUrl`
there will fail validation with:

```
Error: Config validation failed: models.providers.openai.models: Invalid input: expected array, received undefined
```

For standard use, set the API key as an environment variable and configure the
model separately:

```bash
# Set the API key (add to ~/.bash_profile to persist across sessions)
export OPENAI_API_KEY="sk-proj-..."
# or for Anthropic:
export ANTHROPIC_API_KEY="sk-ant-..."
```

> **Never paste API keys into chat.** Always set them in the terminal only.

### Minimal `~/.openclaw/openclaw.json`

The config file only needs the model selection. The API key lives in the
environment, not in this file:

```json5
// ~/.openclaw/openclaw.json
{
  agents: {
    defaults: {
      model: {
        primary: "openai/gpt-5.4",
      },
    },
  },
}
```

---

## Run the Gateway

> **Do not use `node dist/index.js gateway start` or `gateway stop`.**
> Those manage the launchd system daemon, which is not installed in a source
> build. You will see `Gateway service not loaded` — this is expected, not an
> error. Use `pnpm gateway:dev` instead.

### Dev mode (recommended for study)

`pnpm gateway:dev` starts the gateway with all messaging channels skipped
(`OPENCLAW_SKIP_CHANNELS=1`). No WhatsApp, Telegram, or other channel
connections are made. Safe to run locally.

**Critical:** `pnpm gateway:dev` uses an **isolated dev state directory**
`~/.openclaw-dev/` on port **19001** — not `~/.openclaw/` or port 18789.
Config written to `~/.openclaw/` is invisible to the dev gateway. You must
configure `~/.openclaw-dev/` explicitly.

**Step 1 — configure the dev state dir (one time):**

```bash
# Set the model
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js config set agents.defaults.model.primary openai/gpt-5.4

# Set the port so the CLI can find the dev gateway
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js config set gateway.port 19001

# Verify
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js config validate
# expected: Config valid: ~/.openclaw-dev/openclaw.json
```

**Step 2 — start the gateway:**

```bash
export OPENAI_API_KEY="sk-proj-..."
pnpm gateway:dev
```

Expected startup log (key lines):

```
Dev config ready: ~/.openclaw-dev/openclaw.json
[gateway] agent model: openai/gpt-5.4
[gateway] listening on ws://127.0.0.1:19001, ws://[::1]:19001
[gateway/channels] skipping channel start (OPENCLAW_SKIP_CHANNELS=1)
```

### Test the agent

With the gateway running in Terminal 1, open Terminal 2:

```bash
# Find the configured agent id
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js agents list
# in dev mode the agent id is "dev" (not "default")

# Send a test message
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js agent --agent dev \
  --message "Hello, what model are you?" --thinking low
```

Expected response (clean, no fallback warning):

```
I'm using openai/gpt-5.4.
```

> **Embedded fallback warning** — if you see `Gateway agent failed; falling back
to embedded`, it means the CLI connected to the wrong port. Make sure
> `gateway.port 19001` is set in the dev config (Step 1 above) and
> `OPENCLAW_STATE_DIR=~/.openclaw-dev` is set on the CLI command.

### Changing the model

Stop the gateway (`Ctrl+C`), update the model, and restart:

```bash
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js config set agents.defaults.model.primary openai/gpt-5.4-nano
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js config validate

export OPENAI_API_KEY="sk-proj-..."
pnpm gateway:dev
# look for: [gateway] agent model: openai/gpt-5.4-nano
```

### With file watch (auto-restart on source changes)

```bash
pnpm gateway:watch
```

Useful when you are editing source files and want the gateway to pick up changes
automatically.

---

## Run Tests

No LLM keys are required for the unit and integration suites.

```bash
# Fast unit tests only
pnpm test:fast

# Gateway-specific tests (auth, routing, tool dispatch, sessions)
pnpm test:gateway

# Scoped to a single file (most useful for studying a module)
pnpm test -- src/gateway/tools-invoke-http.test.ts
pnpm test -- src/gateway/server.auth.modes.test.ts
pnpm test -- src/gateway/server-methods/agent.test.ts
```

Run a specific test by name pattern:

```bash
pnpm test -- src/gateway/auth.test.ts -t "token auth"
```

---

## Debugging in VS Code

Add to `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Gateway Dev",
      "type": "node",
      "request": "launch",
      "program": "${workspaceFolder}/scripts/run-node.mjs",
      "args": ["--dev", "gateway"],
      "cwd": "${workspaceFolder}",
      "env": {
        "OPENCLAW_SKIP_CHANNELS": "1",
        "OPENAI_API_KEY": "sk-proj-..."
      },
      "sourceMaps": true,
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "resolveSourceMapLocations": ["${workspaceFolder}/src/**", "${workspaceFolder}/dist/**"]
    }
  ]
}
```

**Source maps are not generated by the default `pnpm build`.** Build with source
maps enabled first, then attach the debugger:

```bash
OPENCLAW_BUILD_SOURCEMAP=1 pnpm build
```

This passes `--sourcemap` to tsdown and writes `.map` files alongside the `dist/`
output. No source code changes are required — `OPENCLAW_BUILD_SOURCEMAP` is an
existing env var read by `tsdown.config.ts`.

Set breakpoints directly in `src/gateway/*.ts` files. Source maps in `dist/`
resolve them correctly.

### Useful breakpoint locations

| What to observe            | File                                                  | Where                          |
| -------------------------- | ----------------------------------------------------- | ------------------------------ |
| Every inbound WS message   | `src/gateway/server/ws-connection/message-handler.ts` | Top of message handler         |
| Auth result per connection | `src/gateway/auth.ts`                                 | `resolveGatewayAuthResult`     |
| Method authorization       | `src/gateway/server-methods.ts`                       | `authorizeGatewayMethod`       |
| Tool policy evaluation     | `src/gateway/tools-invoke-http.ts`                    | `applyToolPolicyPipeline` call |
| Agent run start            | `src/gateway/server-methods/agent.ts`                 | `agentCommandFromIngress` call |
| Exec approval check        | `src/gateway/exec-approval-manager.ts`                | `getApproval`                  |
| Session JSONL write        | `src/gateway/session-transcript-files.fs.ts`          | `archiveFileOnDisk`            |

---

## Inspect Live State

While the gateway is running you can inspect dev state directly.
All commands need `OPENCLAW_STATE_DIR=~/.openclaw-dev`:

```bash
# List configured agents (shows agent ids and models)
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js agents list

# Gateway log (written during dev run)
tail -f /tmp/openclaw/openclaw-$(date +%Y-%m-%d).log

# Read a session transcript (JSONL)
cat ~/.openclaw-dev/agents/dev/sessions/<sessionId>.jsonl | jq .
```

Dev state directory layout:

```
~/.openclaw-dev/
  openclaw.json          ← dev config (model, gateway port, auth token)
  agents/
    dev/
      sessions.json      ← session index
      sessions/
        <sessionId>.jsonl  ← per-session conversation transcript
~/.openclaw/
  workspace-dev/         ← agent workspace (SOUL.md, memory/, etc.)
```

---

## Reset Dev State

### Sessions only

```bash
rm -rf ~/.openclaw-dev/agents/*/sessions/
rm -f ~/.openclaw-dev/agents/*/sessions.json
```

### Full dev reset (keeps credentials)

```bash
rm -rf ~/.openclaw-dev/
```

The next `pnpm gateway:dev` run recreates the dev config automatically — re-run
Step 1 of the gateway setup to restore your model and port config.

---

## Common Issues

### `npm install -g pnpm` fails with SSL error

Use the standalone installer:

```bash
curl -fsSL https://get.pnpm.io/install.sh | sh -
source ~/.bashrc
```

### `bun: command not found` during build

Bun installed but not on PATH. Source the profile it updated:

```bash
source ~/.bash_profile
# or add permanently:
export PATH="$HOME/.bun/bin:$PATH"
```

### `Config validation failed: models.providers.openai.models`

You wrote into `models.providers.openai.*` which requires a full custom provider
schema. Remove it and use the built-in provider instead:

```bash
# Remove the bad config entry if it exists
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js config unset models.providers.openai

# Use env var for auth, config for model selection only
export OPENAI_API_KEY="sk-proj-..."
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js config set agents.defaults.model.primary openai/gpt-5.4
```

### `Gateway service not loaded`

This appears when running `gateway start` or `gateway stop`. Those commands
manage the launchd daemon, which is not installed in a source build. Use
`pnpm gateway:dev` instead. The message is not an error.

### `Unknown agent id "default"`

The dev mode agent id is `dev`, not `default`. List agents to confirm:

```bash
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js agents list
```

### `Gateway agent failed; falling back to embedded`

The CLI connected to port 18789 instead of the dev gateway on 19001. Fix:

```bash
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js config set gateway.port 19001
```

### Port 19001 already in use

A previous dev gateway is still running. Kill it:

```bash
lsof -i :19001
kill <PID>
```

---

## Next Steps

With a running local gateway:

- Follow the [Gateway Source Walkthrough](../gateway/security/gateway-source-walkthrough.md) —
  start Session 1 (`server.impl.ts`) with a debugger attached
- Run tests scoped to the file you are studying:
  `pnpm test -- src/gateway/<file>.test.ts`
- Run the security audit against your local config:
  ```bash
  OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js security audit
  OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js security audit --deep
  ```
