---
title: "Session 1 — server.impl.ts: Gateway Startup Deep Dive"
summary: "Step-by-step walkthrough of the gateway startup sequence in server.impl.ts with VS Code debugger attached — covering config bootstrap, auth, runtime state, and event subscriptions"
read_when:
  - Following the Gateway Source Walkthrough, Session 1
  - Learning how the gateway initializes from source
  - Setting up debugger breakpoints for gateway development
---

# Session 1 — `server.impl.ts`: Gateway Startup Deep Dive

This is a hands-on walkthrough of `src/gateway/server.impl.ts` — the file that owns the entire
gateway startup sequence. You will attach a VS Code debugger to a live `pnpm gateway:dev` run and
step through the startup, observing each phase as it executes.

**Prerequisite:** Complete [Local Dev Build](../../help/dev-local-build.md) and verify that a basic
`pnpm gateway:dev` run works on your machine before starting this session.

---

## 1. What this file does

`server.impl.ts` exports one function:

```typescript
export async function startGatewayServer(
  port = 18789,
  opts: GatewayServerOptions = {},
): Promise<GatewayServer>;
```

Everything that happens at gateway startup — config loading, auth bootstrap, HTTP/WS server
creation, plugin loading, channel start, cron, event subscriptions — happens inside this one
function. It is 1,496 lines long. This session covers the full sequence.

**Module-level side effect (line 148):**

```typescript
ensureOpenClawCliOnPath();
```

This runs the moment the module is imported, before `startGatewayServer` is called. It ensures
the `openclaw` CLI binary is on `PATH` for any subprocess or exec call made later.

---

## 2. Full call chain: `pnpm gateway:dev` → `startGatewayServer`

Understanding the chain between the npm script and the function you are debugging
removes the guesswork about what is setting up the environment before you hit your
first breakpoint.

---

### Step 1 — `pnpm gateway:dev` (package.json)

```
"gateway:dev": "OPENCLAW_SKIP_CHANNELS=1 node scripts/run-node.mjs --dev gateway"
```

Sets `OPENCLAW_SKIP_CHANNELS=1` in the environment, then invokes
`scripts/run-node.mjs` with args `["--dev", "gateway"]`.

---

### Step 2 — `scripts/run-node.mjs` → `runNodeMain()`

`run-node.mjs` is a build-aware launcher. It decides whether the `dist/` output is
stale by comparing:

- The `.buildstamp` file mtime vs. the latest source mtime under `src/` and
  `extensions/`
- The current git HEAD vs. the stamped HEAD
- `git status --porcelain` for untracked/modified build-relevant files

If `dist/` is stale, it runs `node scripts/tsdown-build.mjs --no-clean` to
rebuild first. Once `dist/` is fresh it calls:

```js
// run-node.mjs: runOpenClaw()
spawn(process.execPath, ["openclaw.mjs", "--dev", "gateway"], { stdio: "inherit" });
```

---

### Step 3 — `openclaw.mjs`

The thin wrapper at the repo root:

1. Checks Node version (≥ 22.12; exits if not)
2. Calls `module.enableCompileCache()` (best-effort)
3. Tries `await import("./dist/entry.js")` (falls back to `dist/entry.mjs`)

---

### Step 4 — `dist/entry.js` = `src/entry.ts`

The real entry point. Key steps for the `--dev gateway` argv:

```
parseCliProfileArgs(["--dev", "gateway"])
  → profile = "dev", stripped argv = ["gateway"]

applyCliProfileEnv({ profile: "dev" })
  → OPENCLAW_PROFILE = "dev"
  → OPENCLAW_STATE_DIR = ~/.openclaw-dev   (resolveProfileStateDir: ~/.openclaw + "-dev")
  → OPENCLAW_CONFIG_PATH = ~/.openclaw-dev/openclaw.json

process.argv = ["node", "openclaw.mjs", "gateway"]

runMainOrRootHelp(["node", "openclaw.mjs", "gateway"])
  → import("./cli/run-main.js").then(({ runCli }) => runCli(argv))
```

---

### Step 5 — `src/cli/run-main.ts` → `runCli(argv)`

```
parseCliContainerArgs(argv)   → no container target
parseCliProfileArgs(argv)     → --dev already stripped; no profile flag
tryRouteCli(["...", "gateway"])
  → getCommandPathWithRootOptions → path = ["gateway"]
  → findRoutedCommand(["gateway"]) → no match (only "gateway status" is pre-routed)
  → returns false

enableConsoleCapture()
buildProgram()                → creates the Commander root program

primary = getPrimaryCommand(argv) = "gateway"
registerCoreCliByName(program, ctx, "gateway", argv)
  → lazy import("./gateway-cli.js")
  → registerGatewayCli(program)   ← wires all gateway subcommands

program.parseAsync(["node", "openclaw.mjs", "gateway"])
  → Commander matches the "gateway" command
  → "gateway" command action fires (registered by addGatewayRunCommand)
```

Key file: `src/cli/gateway-cli/register.ts` — `addGatewayRunCommand` receives the
Commander `.command("gateway")` object, attaches all `--port`, `--bind`, `--dev`
options to it, and wires `.action(async (opts) => runGatewayCommand(opts))`.

---

### Step 6 — `src/cli/gateway-cli/run.ts` → `runGatewayCommand(opts)`

Performs all pre-flight validation before touching the server:

```
devMode = true  (opts.dev=true via isDevProfile)
ensureDevGatewayConfig({ reset: false })
  → creates ~/.openclaw-dev/ skeleton and sets gateway.mode=local if missing

loadConfig()         → reads ~/.openclaw-dev/openclaw.json
port = 19001         → from config gateway.port
bind = "loopback"    → from config gateway.bind
resolveGatewayAuth() → reads auth token from config or env

Validates:
  - gateway.mode === "local" (or --allow-unconfigured)
  - bind !== "loopback" requires a shared secret
  - auth mode = password requires a configured password

startLoop = async () =>
  runGatewayLoop({
    start: async () => startGatewayServer(19001, { bind: "loopback", ... })
  })

await startLoop()
```

---

### Step 7 — `src/cli/gateway-cli/run-loop.ts` → `runGatewayLoop(params)`

Wraps `startGatewayServer` in a restart/signal loop so the process stays alive
across `SIGUSR1`-triggered in-process restarts:

```
acquireGatewayLock({ port: 19001 })
  → writes ~/.openclaw-dev/gateway-19001.lock (PID file)

Installs OS signal handlers:
  SIGTERM / SIGINT → graceful shutdown (drain active tasks → server.close())
  SIGUSR1          → graceful restart (drain → server.close() → re-call start())

for (;;) {
  server = await params.start()   ← calls startGatewayServer()
  await new Promise(resolve => { restartResolver = resolve })
  // blocks here until SIGUSR1 fires
}
```

---

### Step 8 — `src/gateway/server.impl.ts` → `startGatewayServer(port, opts)`

Finally arrives here. The 12 startup phases documented in Section 3 run in sequence.

---

### Full chain summary

```
pnpm gateway:dev
  └─ OPENCLAW_SKIP_CHANNELS=1 node scripts/run-node.mjs --dev gateway
       └─ run-node.mjs: runNodeMain()
            ├─ [if stale] node scripts/tsdown-build.mjs --no-clean
            └─ runOpenClaw() → node openclaw.mjs --dev gateway
                 └─ openclaw.mjs
                      └─ import dist/entry.js  (src/entry.ts)
                           ├─ parseCliProfileArgs: profile="dev" → OPENCLAW_STATE_DIR=~/.openclaw-dev
                           └─ runMainOrRootHelp → runCli()  (src/cli/run-main.ts)
                                ├─ tryRouteCli: no match for "gateway"
                                ├─ buildProgram() + registerGatewayCli()
                                └─ program.parseAsync → gateway command action
                                     └─ runGatewayCommand()  (src/cli/gateway-cli/run.ts)
                                          ├─ ensureDevGatewayConfig()
                                          ├─ loadConfig() → ~/.openclaw-dev/openclaw.json
                                          └─ runGatewayLoop()  (src/cli/gateway-cli/run-loop.ts)
                                               ├─ acquireGatewayLock(port=19001)
                                               └─ startGatewayServer(19001, opts)
                                                    └─ src/gateway/server.impl.ts (Phases 1–12)
```

---

## 3. Attach the VS Code debugger

### Before you start — rebuild with source maps

Breakpoints in `src/` files require source maps in `dist/`. The default `pnpm build`
does not generate them. Run this once before attaching the debugger:

```bash
OPENCLAW_BUILD_SOURCEMAP=1 pnpm build
```

This sets `sourcemap: true` in `tsdown.config.ts` via the `OPENCLAW_BUILD_SOURCEMAP`
env flag and writes `dist/*.js.map` alongside the JS bundles. VS Code uses these files
to resolve `src/` breakpoints to locations in the bundled output. Without them,
breakpoints never fire.

> You must rerun `OPENCLAW_BUILD_SOURCEMAP=1 pnpm build` after any source change you
> want to observe under the debugger.

---

### Option A — VS Code launch configuration (recommended)

Add the following to `.vscode/launch.json` in the repo root (create the file if it doesn't exist):

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
        "OPENAI_API_KEY": "your-key-here"
      },
      "sourceMaps": true,
      "outFiles": ["${workspaceFolder}/dist/**/*.js"],
      "resolveSourceMapLocations": ["${workspaceFolder}/src/**", "${workspaceFolder}/dist/**"]
    }
  ]
}
```

**Important:** Do not commit real API keys. Set `OPENAI_API_KEY` in the `env` block only for local
use, or set it as an OS environment variable and omit it from launch.json.

Open VS Code (`code .` from repo root), go to **Run → Start Debugging** (F5), select
"Gateway Dev". The gateway starts inside the VS Code debugger process.

### Option B — Attach to `pnpm gateway:dev` (if you prefer a separate terminal)

1. Start the gateway with `--inspect`:

```bash
# In Terminal 1
export OPENAI_API_KEY="sk-proj-..."
export OPENCLAW_STATE_DIR=~/.openclaw-dev
node --inspect dist/index.js --dev gateway
```

2. In VS Code, press **F1** → "Debug: Attach to Node Process" → select the process on port 9229.

> Option A is simpler for this session — it gives you breakpoints from the first line.

---

## 3. Startup sequence: the 12 phases

The startup sequence inside `startGatewayServer` has 12 distinct phases. Below is each phase with
the exact line numbers, what it does, and where to set breakpoints to observe it.

---

### Phase 1 — Environment setup and port stamp (lines 370–382)

```typescript
process.env.OPENCLAW_GATEWAY_PORT = String(port);
logAcceptedEnvOption({ key: "OPENCLAW_RAW_STREAM", ... });
```

The first thing the function does is write the runtime port into the process environment.
Every downstream component that needs to know the gateway port reads this env var — it is the
authoritative source at runtime.

**Set a breakpoint at line 370.** When it hits, inspect:

- `port` — should be `19001` for `pnpm gateway:dev`
- `opts` — the `GatewayServerOptions` passed in (bind mode, auth override, etc.)

---

### Phase 2 — Config load and legacy migration (lines 384–427)

```typescript
let configSnapshot = await readConfigFileSnapshot();
if (configSnapshot.legacyIssues.length > 0) {
  // auto-migrate
}
configSnapshot = await readConfigFileSnapshot();
```

The config file (`~/.openclaw-dev/openclaw.json` in dev mode) is read twice:

1. First read detects legacy schema entries and auto-migrates them if found
2. Second read gets the clean post-migration snapshot

After migration, `applyPluginAutoEnable` runs — any plugins with auto-enable conditions (e.g. a
channel extension whose binary is present) get their `plugins.*.enabled = true` written to config.

**Breakpoint at line 384.** Inspect `configSnapshot.config` to see the parsed config object. You
should see the model and port you set in the dev setup:

```json
{ "agents": { "defaults": { "model": { "primary": "openai/gpt-5.4" } } },
  "gateway": { "port": 19001, "bind": "loopback", ... } }
```

---

### Phase 3 — Runtime secrets bootstrap (lines 429–493)

```typescript
const activateRuntimeSecrets = async (config, params) =>
  await runWithSecretsActivationLock(async () => {
    const prepared = await prepareSecretsRuntimeSnapshot({ config });
    if (params.activate) {
      activateSecretsRuntimeSnapshot(prepared);
    }
    ...
  });
```

`activateRuntimeSecrets` is a closure that wraps secret preparation in a serialized execution lock
(`runWithSecretsActivationLock`). This prevents race conditions if config reload and a startup
retry both try to activate secrets concurrently.

The lock is implemented as a promise chain (`secretsActivationTail`) — each operation appends to
the chain, so activations are always sequential regardless of when they are called.

**Breakpoint at line 456** (`activateSecretsRuntimeSnapshot(prepared)`). When this hits:

- `prepared.sourceConfig` contains the config used for secret resolution
- `prepared.warnings` will show any `SECRETS_REF_IGNORED_INACTIVE_SURFACE` messages

For a standard dev setup with only `OPENAI_API_KEY`, you should see zero warnings.

---

### Phase 4 — Auth bootstrap and token generation (lines 495–535)

```typescript
const authBootstrap = await prepareGatewayStartupConfig({
  configSnapshot,
  runtimeConfig: startupRuntimeConfig,
  activateRuntimeSecrets,
});
cfgAtStart = authBootstrap.cfg;
if (authBootstrap.generatedToken) { ... }
```

`prepareGatewayStartupConfig` calls `ensureGatewayStartupAuth`, which does one critical thing: if
no gateway auth token exists in the config, it **generates one and writes it to disk**. This is how
`gateway.auth.token` ends up in `~/.openclaw-dev/openclaw.json` after first run.

After auth bootstrap, two startup migrations run:

- `maybeSeedControlUiAllowedOriginsAtStartup` — seeds CORS origins for existing non-loopback installs
- `runStartupMatrixMigration` — Matrix extension path migration

**Breakpoint at line 505** (the `if (authBootstrap.generatedToken)` check). On first run this will
be `true` — you will see the "Generated a new token" log line. On subsequent runs, `generatedToken`
is `false` because the token already exists in config.

**Security note:** The generated token is the gateway bearer token. Anyone who can read
`~/.openclaw-dev/openclaw.json` has full operator access. On a shared host, this is a high-risk
surface.

---

### Phase 5 — Plugin registry and method list (lines 553–601)

```typescript
initSubagentRegistry();
const { pluginRegistry, gatewayMethods: baseGatewayMethods } = loadGatewayStartupPlugins({ ... });
const channelMethods = listChannelPlugins().flatMap((plugin) => plugin.gatewayMethods ?? []);
const gatewayMethods = Array.from(new Set([...baseGatewayMethods, ...channelMethods]));
```

The plugin registry is loaded, then the full gateway method list is assembled by merging:

1. Core gateway methods (`baseMethods` from `listGatewayMethods()`)
2. Methods added by loaded plugins (`baseGatewayMethods` after plugin load)
3. Methods from channel plugins (`channelMethods`)

De-duplication with `Set` ensures each method name appears exactly once.

`OPENCLAW_SKIP_CHANNELS=1` (set by `pnpm gateway:dev`) causes `startChannels` to be a no-op later
— channel plugins still register their methods, but the underlying connections are never started.

**Breakpoint at line 590** (the `gatewayMethods` assignment). Inspect the array to see all
registered method names. Look for methods grouped by prefix: `agent.*`, `sessions.*`,
`gateway.*`, `chat.*`. The list defines the complete WS RPC surface.

---

### Phase 6 — Runtime config resolution (lines 592–625)

```typescript
const runtimeConfig = await resolveGatewayRuntimeConfig({
  cfg: cfgAtStart, port, bind: opts.bind, host: opts.host, ...
});
const { bindHost, controlUiEnabled, resolvedAuth, tailscaleConfig, ... } = runtimeConfig;
const { rateLimiter: authRateLimiter, browserRateLimiter: browserAuthRateLimiter } =
  createGatewayAuthRateLimiters(rateLimitConfig);
```

`resolveGatewayRuntimeConfig` computes the final resolved values for bind host, TLS, auth mode,
and optional features (Control UI, OpenAI-compatible endpoints). These values are used for the rest
of startup.

Two rate limiters are created here:

- `authRateLimiter` — general auth rate limiter, may exempt loopback connections per config
- `browserAuthRateLimiter` — always enforces throttling on browser-origin WS connections
  (`exemptLoopback: false`, line 193). This is the hardened limiter for web UI connections.

**Breakpoint at line 624.** After the rate limiters are created, inspect:

- `authRateLimiter` — may be `undefined` if no rate limit config is set (unconfigured = no limit)
- `browserAuthRateLimiter` — always defined, loopback connections are NOT exempt

---

### Phase 7 — HTTP server, WSS, and runtime state (lines 691–740)

```typescript
const {
  httpServer, wss, preauthConnectionBudget, clients, broadcast, broadcastToConnIds,
  chatRunState, ...
} = await createGatewayRuntimeState({
  cfg: cfgAtStart, bindHost, port, controlUiEnabled, resolvedAuth,
  rateLimiter: authRateLimiter, ...
});
```

`createGatewayRuntimeState` (`src/gateway/server-runtime-state.ts`) creates:

- The underlying `http.Server` (and optional additional servers for OpenAI/chat completions)
- The `WebSocket.Server` (`wss`) for the real-time RPC channel
- `clients` — the live set of connected WS clients
- `broadcast` / `broadcastToConnIds` — functions to push events to connected clients
- `chatRunState` — in-memory state for streaming agent runs

This is the point at which the HTTP/WS server exists in memory, but is not yet listening. Port
binding happens later.

**Breakpoint at line 712** (inside `createGatewayRuntimeState`'s destructuring). Inspect `wss` —
it should be a `WebSocket.Server` instance. Its `clients` set is empty at this point.

---

### Phase 8 — Node registry, cron, and maintenance timers (lines 802–900)

```typescript
const nodeRegistry = new NodeRegistry();
const cronState = buildGatewayCronService({ cfg: cfgAtStart, deps, broadcast });
let { cron, storePath: cronStorePath } = cronState;

({ tickInterval, healthInterval, dedupeCleanup, mediaCleanup } =
  startGatewayMaintenanceTimers({ ... }));
```

Three key subsystems start here:

1. **NodeRegistry** — tracks connected mobile/remote nodes. Each node has a presence timer.
2. **Cron service** — persistent automation scheduler. The store path is used to save/load cron jobs across restarts.
3. **Maintenance timers** — four `setInterval` loops:
   - `tickInterval` — presence/health broadcast tick
   - `healthInterval` — periodic `refreshGatewayHealthSnapshot` calls
   - `dedupeCleanup` — flushes the message deduplication map
   - `mediaCleanup` — TTL-based cleanup of media files (default 7 days)

**Breakpoint at line 824** (`buildGatewayCronService`). After this line, inspect `cron` — it is
the cron service handle. Note that `cron.start()` is called later (line 1124) — the service object
exists here but is not running yet.

---

### Phase 9 — Event subscriptions (lines 902–1102)

```typescript
agentUnsub = onAgentEvent(createAgentEventHandler({ broadcast, ... }));
heartbeatUnsub = onHeartbeatEvent((evt) => broadcast("heartbeat", evt, { dropIfSlow: true }));
transcriptUnsub = onSessionTranscriptUpdate((update) => { ... });
lifecycleUnsub = onSessionLifecycleEvent((event) => { ... });
```

Four event subscriptions are registered here. These are the bridge between internal events
(emitted by the agent runner, session storage, and heartbeat runner) and the WS clients.

| Subscription      | What it broadcasts                            | WS event name                         |
| ----------------- | --------------------------------------------- | ------------------------------------- |
| `agentUnsub`      | Agent run events (tokens, tool calls, finish) | `agent.*`, `chat.*`                   |
| `heartbeatUnsub`  | Periodic heartbeat                            | `heartbeat`                           |
| `transcriptUnsub` | New JSONL entries written to session files    | `session.message`, `sessions.changed` |
| `lifecycleUnsub`  | Session start/stop/spawn lifecycle            | `sessions.changed`                    |

All four return unsubscribe functions. These are stored in variables that are referenced by the
close handler — when `gateway.close()` is called, these unsubs are called to avoid memory leaks
and stale handler callbacks after shutdown.

**Breakpoint at line 924** (`transcriptUnsub = onSessionTranscriptUpdate(...)`). The callback
registered here is the one that fires every time the agent writes a new JSONL line to a session
file. It is the critical path for streaming agent output to the UI.

**Exercise:** Set a breakpoint inside the `transcriptUnsub` callback body at line 929 (the
`const sessionKey = ...` line). Then, in a second terminal, send a test message:

```bash
OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js agent --agent dev \
  --message "Reply with one word: hello" --thinking low
```

The breakpoint at line 929 will fire for each JSONL entry the agent writes. You will see:

- `update.message` — the raw `ConversationEntry` being appended
- `update.sessionFile` — the path to the JSONL file
- `sessionKey` — resolved from either `update.sessionKey` or the file path

---

### Phase 10 — WS handlers attached and startup log (lines 1270–1303)

```typescript
attachGatewayWsHandlers({
  wss, clients, resolvedAuth, rateLimiter: authRateLimiter,
  browserRateLimiter: browserAuthRateLimiter,
  gatewayMethods, events: GATEWAY_EVENTS, ...
  extraHandlers: {
    ...pluginRegistry.gatewayHandlers,
    ...execApprovalHandlers,
    ...pluginApprovalHandlers,
    ...secretsHandlers,
  },
  context: gatewayRequestContext,
});
logGatewayStartup({ cfg: cfgAtStart, bindHost, port, ... });
```

`attachGatewayWsHandlers` (`src/gateway/server-ws-runtime.ts`) wires the WebSocket server to the
full method dispatch table. After this call, the gateway is ready to accept connections and route
method calls.

`logGatewayStartup` emits the startup log lines you see in the terminal:

```
[gateway] listening on ws://127.0.0.1:19001, ws://[::1]:19001
[gateway/channels] skipping channel start (OPENCLAW_SKIP_CHANNELS=1)
```

**Breakpoint at line 1295** (`logGatewayStartup`). This is the last step before the gateway is
fully open. When this breakpoint hits, the server is live and accepting connections.

**Security note:** The `extraHandlers` block merges plugin-provided handlers with
`execApprovalHandlers` and `secretsHandlers`. If a plugin registers a handler with a name that
collides with a core handler, the spread order here determines which one wins. In the current code,
plugin handlers go first (`pluginRegistry.gatewayHandlers`), then exec and secrets handlers —
meaning core handlers can override plugin handlers if there is a collision.

---

### Phase 11 — Channels, sidecars, and plugin hooks (lines 1325–1356)

```typescript
({ pluginServices } = await startGatewaySidecars({
  cfg: gatewayPluginConfigAtStart, pluginRegistry, defaultWorkspaceDir,
  deps, startChannels, ...
}));

const hookRunner = getGlobalHookRunner();
if (hookRunner?.hasHooks("gateway_start")) {
  void hookRunner.runGatewayStart({ port }, { port }).catch(...);
}
```

`startGatewaySidecars` calls `startChannels` — which, in a normal (non-dev) run, starts all
configured channel adapters (WhatsApp, Telegram, etc.). With `OPENCLAW_SKIP_CHANNELS=1`, the
channel start is a no-op, so you see the "skipping channel start" log line.

After channels, the `gateway_start` plugin hook runs. If any installed skill has a `gateway_start`
hook function, it fires here. This is a fire-and-forget call — errors are logged but do not abort
startup.

**Breakpoint at line 1349** (`hookRunner?.hasHooks("gateway_start")`). Inspect `hookRunner` — in a
fresh dev setup with no installed skills, it will be `null` and this block is skipped.

---

### Phase 12 — Config reloader and close handler (lines 1358–1495)

```typescript
configReloader = startGatewayConfigReloader({
  initialConfig: cfgAtStart,
  readSnapshot: readConfigFileSnapshot,
  onHotReload: async (plan, nextConfig) => { ... },
  onRestart: async (plan, nextConfig) => { ... },
  watchPath: configSnapshot.path,
});

return {
  close: async (opts) => { ... }
};
```

The config reloader watches the config file for changes. When `~/.openclaw-dev/openclaw.json` is
modified, it reads the new snapshot, applies a diff plan, and either hot-reloads (for config keys
that support live reload) or schedules a gateway restart.

The function returns the `GatewayServer` handle with a single `close()` method. `close()` runs in
sequence:

1. `gateway_stop` plugin hook (fire-and-forget, waits for completion)
2. Stops diagnostic heartbeat
3. Clears skills refresh timer
4. Disposes rate limiters
5. Clears secrets snapshot
6. Calls the inner `close` handler (stops cron, channels, WS server, HTTP server)

**Breakpoint at line 1472** (`return { close: ... }`). When this line is hit, startup is fully
complete. The gateway is live, all subscriptions are active, and the server is accepting
connections.

---

## 4. Putting it all together — the dependency graph

The 12 phases have strict ordering dependencies. Here is a simplified view of what depends on what:

```
Phase 1: port → env
Phase 2: configSnapshot → (Phases 3-12)
Phase 3: activateRuntimeSecrets closure → (Phase 4, 12)
Phase 4: cfgAtStart (auth-bootstrapped) → (Phases 5-12)
Phase 5: pluginRegistry, gatewayMethods → (Phase 10)
Phase 6: runtimeConfig (bindHost, resolvedAuth) → (Phase 7)
Phase 7: httpServer, wss, broadcast, clients → (Phases 8-12)
Phase 8: nodeRegistry, cron, maintenance timers
Phase 9: event subscriptions (transcriptUnsub etc.) → (Phase 12 close)
Phase 10: attachGatewayWsHandlers → gateway is open
Phase 11: channels started, plugin hooks fired
Phase 12: configReloader watching, close handler registered
```

Any error thrown in Phases 1–11 calls `closeOnStartupFailure()` (line 759), which tears down
whatever was initialized and re-throws. This is the try/catch block that wraps Phases 8–11
(lines 837–1441).

---

## 5. Exercises

### Exercise 1 — Observe the auth token bootstrap

1. Delete the token from dev config:
   ```bash
   OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js config unset gateway.auth.token
   ```
2. Set a breakpoint at line 505 (`if (authBootstrap.generatedToken)`).
3. Start the debugger. The breakpoint should hit with `generatedToken === true`.
4. Step into `ensureGatewayStartupAuth` in `src/gateway/startup-auth.ts` to see how the token is
   generated and written to disk.

### Exercise 2 — Trace a message through the transcript subscription

1. Set a breakpoint at line 929 inside `transcriptUnsub`.
2. Start the debugger (gateway is running).
3. In Terminal 2, send a message:
   ```bash
   OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js agent --agent dev \
     --message "Say: ping" --thinking low
   ```
4. The breakpoint fires. Step through the handler:
   - `update.sessionFile` — path to the JSONL file being appended
   - `update.message` — the `ConversationEntry` (role: "assistant", content: the response)
   - `sessionKey` — resolved from file path since `update.sessionKey` may be undefined
   - `connIds` — the set of WS clients subscribed to this session (empty if no UI is open)
5. Note: if `connIds.size === 0` (no WS clients), the function returns early without broadcasting.
   Open the Control UI in a browser to make a client connect and see the broadcast path.

### Exercise 3 — Inspect the gateway method table

1. Set a breakpoint at line 590 (`const gatewayMethods = Array.from(new Set(...))`).
2. Start the debugger and inspect the `gatewayMethods` array.
3. Count the methods — a default dev build should have 30–60 methods.
4. Look for any that come from channel plugins (they will have channel-specific prefixes).
5. Cross-reference with the static deny list in `src/security/dangerous-tools.ts` —
   verify that `sessions_spawn` and `cron` are present in `gatewayMethods` (they are WS methods)
   and absent from the HTTP tool endpoint's allow set.

### Exercise 4 — Config hot-reload

1. Start the gateway in debug mode.
2. With the gateway running, edit `~/.openclaw-dev/openclaw.json` — change a non-critical value
   like `gateway.channelHealthCheckMinutes` to `10`.
3. The config reloader (Phase 12) will detect the file change.
4. Set a breakpoint in `startGatewayConfigReloader` (`src/gateway/config-reload.ts`) on the
   `onHotReload` callback. Observe the diff plan that is computed and applied.

---

## 6. Key files to read next

After completing this session, the natural next reads are:

| File                                   | What it covers                                         |
| -------------------------------------- | ------------------------------------------------------ |
| `src/gateway/server-ws-runtime.ts`     | WS connection lifecycle, auth, method dispatch         |
| `src/gateway/server-methods.ts`        | Core method handler implementations                    |
| `src/gateway/startup-auth.ts`          | Auth token generation, persistence, validation         |
| `src/gateway/server-runtime-state.ts`  | HTTP server, WSS creation, broadcast internals         |
| `src/gateway/tools-invoke-http.ts`     | HTTP POST /tools/invoke — the tool dispatch API        |
| `src/gateway/auth.ts`                  | `resolveGatewayAuthResult` — per-request auth decision |
| `src/gateway/exec-approval-manager.ts` | Exec approval state machine                            |
| `src/sessions/transcript-events.ts`    | How JSONL writes emit the events seen in Phase 9       |

These map to Sessions 2–7 in the [Gateway Source Walkthrough](gateway-source-walkthrough.md).

---

## 7. Useful breakpoint summary

| What to catch                       | File             | Line | Inspect                                     |
| ----------------------------------- | ---------------- | ---- | ------------------------------------------- |
| Function entry                      | `server.impl.ts` | 366  | `port`, `opts`                              |
| Config snapshot                     | `server.impl.ts` | 384  | `configSnapshot.config`                     |
| Secrets activation                  | `server.impl.ts` | 456  | `prepared.warnings`                         |
| Token bootstrap                     | `server.impl.ts` | 505  | `authBootstrap.generatedToken`              |
| Method list built                   | `server.impl.ts` | 590  | `gatewayMethods` (array of method names)    |
| Rate limiters created               | `server.impl.ts` | 624  | `authRateLimiter`, `browserAuthRateLimiter` |
| WS server created                   | `server.impl.ts` | 712  | `wss.clients` (empty set at this point)     |
| Cron service built                  | `server.impl.ts` | 824  | `cron` object                               |
| Transcript subscription             | `server.impl.ts` | 929  | `update.message`, `sessionKey`, `connIds`   |
| Gateway open (WS handlers attached) | `server.impl.ts` | 1270 | (gateway is now accepting connections)      |
| Startup log emitted                 | `server.impl.ts` | 1295 | `bindHost`, `port`                          |
| Startup complete                    | `server.impl.ts` | 1472 | (return value — `close` handle)             |

---

## See Also

- [Local Dev Build](../../help/dev-local-build.md)
- [Gateway Source Walkthrough](gateway-source-walkthrough.md)
- [Attack Surface Map](attack-surface-map.md)
- [STRIDE Threat Model](stride-threat-model.md)
