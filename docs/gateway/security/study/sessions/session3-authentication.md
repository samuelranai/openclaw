---
title: "Session 3 — Authentication"
summary: "Step-by-step walkthrough of the gateway authentication layer with VS Code debugger attached — covering ResolvedGatewayAuth, per-request auth decision, rate limiting, device identity, pairing, roles, and scopes"
read_when:
  - Following the Gateway Source Walkthrough, Session 3
  - Learning how gateway token/password/device/Tailscale auth works
  - Setting up debugger breakpoints for auth debugging
---

# Session 3 — Authentication

This session covers how the gateway decides whether a connecting client is allowed
in. It spans five files: the startup-time auth configuration object, the per-request
decision engine, the handshake-specific auth state machine, the control-UI policy,
and the rate limiter.

**Prerequisite:** Complete Sessions 1 and 2. You should understand the WS handshake
flow and know where `resolveConnectAuthState` and `resolveConnectAuthDecision` are
called from inside `message-handler.ts`.

---

## 1. What this session covers

| File                                                         | Lines | Role                                                                                |
| ------------------------------------------------------------ | ----- | ----------------------------------------------------------------------------------- |
| `src/gateway/auth.ts`                                        | 494   | `ResolvedGatewayAuth` type, `authorizeGatewayConnect`, all auth modes               |
| `src/gateway/server/ws-connection/auth-context.ts`           | 262   | Handshake-layer auth state: `resolveConnectAuthState`, `resolveConnectAuthDecision` |
| `src/gateway/server/ws-connection/handshake-auth-helpers.ts` | 199   | Browser security context, device signature payload, unauthorized context            |
| `src/gateway/server/ws-connection/connect-policy.ts`         | 126   | Control UI device-auth bypass policy, `evaluateMissingDeviceIdentity`               |
| `src/gateway/auth-rate-limit.ts`                             | 232   | In-memory sliding-window rate limiter                                               |
| `src/gateway/role-policy.ts`                                 | 24    | Roles: `operator` / `node`, `roleCanSkipDeviceIdentity`                             |
| `src/gateway/method-scopes.ts`                               | 236   | Scopes, `METHOD_SCOPE_GROUPS`, `authorizeOperatorScopesForMethod`                   |
| `src/security/secret-equal.ts`                               | 12    | Constant-time credential comparison                                                 |

---

## 2. The two layers of auth

Authentication in the gateway has two distinct layers that run at different points
in the lifecycle:

```
Layer 1 — startup-time: resolveGatewayAuth()  (auth.ts)
  → produces ResolvedGatewayAuth (mode, token, password, allowTailscale, trustedProxy)
  → computed once at startup from config + env + CLI overrides
  → stored in server.impl.ts, passed as a constant to every connection handler

Layer 2 — per-connection: resolveConnectAuthState() + resolveConnectAuthDecision()  (auth-context.ts)
  → runs for every inbound WS connect request
  → calls authorizeGatewayConnect() against the startup-time ResolvedGatewayAuth
  → handles device identity, pairing, bootstrap tokens on top of shared-secret auth
  → produces ConnectAuthState → ConnectAuthDecision
```

Understanding this split matters: the gateway never re-reads config to make a
per-request auth decision. If `gateway.auth.token` changes on disk, nothing
changes for existing or new connections until the config reloader triggers a
restart (Phase 12 from Session 1).

---

## 3. Startup-time auth: `resolveGatewayAuth` (`auth.ts`)

**File:** `src/gateway/auth.ts`, line 208

Called from `src/cli/gateway-cli/run.ts` → passed into `startGatewayServer` as
part of `GatewayServerOptions`.

```typescript
export type ResolvedGatewayAuth = {
  mode: "none" | "token" | "password" | "trusted-proxy";
  modeSource?: "override" | "config" | "password" | "token" | "default";
  token?: string;
  password?: string;
  allowTailscale: boolean;
  trustedProxy?: GatewayTrustedProxyConfig;
};
```

### Mode resolution priority

```
1. authOverride.mode      → modeSource = "override"  (--auth CLI flag)
2. authConfig.mode        → modeSource = "config"    (gateway.auth.mode in config)
3. password credential present → mode = "password",  modeSource = "password"
4. token credential present    → mode = "token",     modeSource = "token"
5. fallback                    → mode = "token",     modeSource = "default"
```

The default is always `"token"` even if no token is configured — in that case the
gateway will generate one at startup (`ensureGatewayStartupAuth`, Session 1 Phase 4).

### Credential resolution

```typescript
resolveGatewayCredentialsFromValues({
  configToken: ...,   // from config, if not a secret ref
  configPassword: ...,
  env,
  tokenPrecedence: "config-first",
  passwordPrecedence: "config-first",
})
```

Config values take precedence over environment variables. `OPENCLAW_GATEWAY_TOKEN`
and `OPENCLAW_GATEWAY_PASSWORD` are the env fallbacks. Secret references (e.g.
`{ "$secret": "..." }`) in config skip the inline resolution path — they are
resolved later by the secrets snapshot system.

### Tailscale flag

```typescript
allowTailscale =
  authConfig.allowTailscale ??
  (tailscaleMode === "serve" && mode !== "password" && mode !== "trusted-proxy");
```

Tailscale auth is auto-enabled when `gateway.tailscale.mode = "serve"` and the
auth mode is `"token"`. It is never auto-enabled for `"password"` or `"trusted-proxy"`.

**Breakpoint at line 274** (`return { mode, modeSource, token, ... }`). Inspect
the returned object at gateway startup:

- `mode` — should be `"token"` in dev
- `modeSource` — `"config"` if `gateway.auth.mode` is set, otherwise `"default"`
- `token` — the raw token string (do not log this in real usage)
- `allowTailscale` — `false` in a basic dev setup

---

## 4. Per-request auth: `authorizeGatewayConnect` (`auth.ts`)

**File:** `src/gateway/auth.ts`, line 368

This is the core auth decision function. Two public wrappers call it:

- `authorizeHttpGatewayConnect` — for HTTP `/tools/invoke` and other HTTP endpoints
  (`authSurface = "http"`)
- `authorizeWsControlUiGatewayConnect` — for WS handshakes
  (`authSurface = "ws-control-ui"`)

The only difference: `ws-control-ui` enables Tailscale header auth
(`shouldAllowTailscaleHeaderAuth` returns `true` only for this surface).

### Decision tree

```
authorizeGatewayConnect(params)
  │
  ├── mode === "trusted-proxy"
  │     └── authorizeTrustedProxy()
  │           ├── remoteAddr must be in trustedProxies list
  │           ├── requiredHeaders must all be present and non-empty
  │           ├── trustedProxyConfig.userHeader must be present
  │           └── if trustedProxy.allowUsers set: user must be in list
  │           → { ok: true, method: "trusted-proxy", user: "<user>" }
  │             or { ok: false, reason: "trusted_proxy_*" }
  │
  ├── mode === "none"
  │     └── { ok: true, method: "none" }
  │
  ├── rate limiter check (if limiter provided)
  │     └── if blocked → { ok: false, reason: "rate_limited", retryAfterMs }
  │
  ├── Tailscale header auth (ws-control-ui surface only, allowTailscale=true, !localDirect)
  │     └── resolveVerifiedTailscaleUser()
  │           ├── getTailscaleUser(): reads Tailscale-User-Login header
  │           ├── isTailscaleProxyRequest(): remoteAddr must be loopback + proxy headers present
  │           ├── resolveTailscaleClientIp(): resolves real IP via X-Forwarded-For
  │           └── readTailscaleWhoisIdentity(clientIp): calls Tailscale local API
  │                 → verifies header login matches whois login (normalized lowercase)
  │           → { ok: true, method: "tailscale", user: "<login>" }
  │
  ├── mode === "token"
  │     ├── if !auth.token → { ok: false, reason: "token_missing_config" }
  │     ├── if !connectAuth.token → { ok: false, reason: "token_missing" }  ← no rate-limit hit
  │     ├── safeEqualSecret(connectAuth.token, auth.token)
  │     │     → false: recordFailure() + { ok: false, reason: "token_mismatch" }
  │     │     → true:  reset() +    { ok: true, method: "token" }
  │
  └── mode === "password"
        ├── if !auth.password → { ok: false, reason: "password_missing_config" }
        ├── if !connectAuth.password → { ok: false, reason: "password_missing" }
        ├── safeEqualSecret(connectAuth.password, auth.password)
        │     → false: recordFailure() + { ok: false, reason: "password_mismatch" }
        │     → true:  reset() +    { ok: true, method: "password" }
```

**Key design:** `token_missing` and `password_missing` do **not** call
`recordFailure()`. Missing credentials (no token provided at all) do not burn
rate-limit slots — only wrong credentials do. This prevents a bare browser open
from consuming the limit before the user types their token.

### Constant-time comparison: `safeEqualSecret` (`src/security/secret-equal.ts`)

```typescript
export function safeEqualSecret(provided, expected): boolean {
  const hash = (s: string) => createHash("sha256").update(s).digest();
  return timingSafeEqual(hash(provided), hash(expected));
}
```

Both inputs are SHA-256 hashed before `timingSafeEqual`. This means:

1. The comparison always takes the same time regardless of how many characters match
2. The hash normalizes lengths so `timingSafeEqual` (which requires equal-length
   buffers) always receives 32-byte buffers
3. Timing attacks on long vs. short tokens are not possible

**Breakpoint at line 448** (`safeEqualSecret(connectAuth.token, auth.token)`).
When this fires:

- `connectAuth.token` — what the client sent
- `auth.token` — the gateway's configured token (handle with care in logs)
- The return value tells you if credentials matched

---

## 5. Handshake auth state: `auth-context.ts`

**File:** `src/gateway/server/ws-connection/auth-context.ts`

`ConnectAuthState` is the intermediate result produced between the two steps of
auth decision:

```typescript
type ConnectAuthState = {
  authResult: GatewayAuthResult; // result of authorizeWsControlUiGatewayConnect
  authOk: boolean; // authResult.ok
  authMethod: GatewayAuthResult["method"];
  sharedAuthOk: boolean; // true if token/password/trusted-proxy verified
  sharedAuthProvided: boolean; // true if client sent any token/password
  bootstrapTokenCandidate?: string; // from connectParams.auth.bootstrapToken
  deviceTokenCandidate?: string; // from connectParams.auth.deviceToken (or .token fallback)
  deviceTokenCandidateSource?: "explicit-device-token" | "shared-token-fallback";
};
```

### Step 1: `resolveConnectAuthState` (line 84)

Called from `message-handler.ts` before device signature verification.

```
resolveConnectAuthState(params)
  │
  ├── resolveSharedConnectAuth(connectAuth)
  │     → extracts { token, password } from connect params
  │
  ├── resolveDeviceTokenCandidate(connectAuth)
  │     → explicit: connectAuth.deviceToken  → source = "explicit-device-token"
  │     → fallback: connectAuth.token        → source = "shared-token-fallback"
  │       (the CLI sends the shared token in both fields initially;
  │        if rejected as shared, it tries as device token)
  │
  ├── resolveBootstrapTokenCandidate(connectAuth)
  │     → connectAuth.bootstrapToken (one-time pairing token)
  │
  ├── authorizeWsControlUiGatewayConnect(...)   ← primary auth
  │     → if hasDeviceTokenCandidate: pass rateLimiter=undefined
  │       (device-token auth uses its own rate-limit scope later)
  │
  ├── [if deviceTokenCandidate + authOk + shared method]:
  │     additional check: is the shared-secret scope itself rate-limited?
  │     → prevents shared token being used to bootstrap unlimited device-token attempts
  │
  └── authorizeHttpGatewayConnect(allowTailscale=false)  ← sharedAuthOk probe
        → this secondary call resolves sharedAuthOk independently of Tailscale auth,
          because sharedAuthOk is used by roleCanSkipDeviceIdentity() (see Section 7)
```

**`sharedAuthOk` vs `authOk`:**

- `authOk` — the primary auth result, can be `true` via Tailscale
- `sharedAuthOk` — true only for token/password/trusted-proxy, never Tailscale
- A Tailscale-authenticated operator has `authOk=true` but `sharedAuthOk=false`
- `roleCanSkipDeviceIdentity` uses `sharedAuthOk` — so Tailscale-only operators
  still need device identity

### Step 2: `resolveConnectAuthDecision` (line 169)

Called from `message-handler.ts` after device signature verification has passed.
Runs only if the primary `authorizeWsControlUiGatewayConnect` returned `authOk=false`
(i.e. the shared secret failed) and a device credential was provided.

```
resolveConnectAuthDecision(params)
  │
  ├── bootstrap token path (authOk=false + bootstrapTokenCandidate present)
  │     → verifyDeviceBootstrapToken({ deviceId, publicKey, token, role, scopes })
  │           ← one-time token, issued during the pairing approval flow
  │     → if ok: authOk=true, authMethod="bootstrap-token"
  │     → if fail: authResult = { ok: false, reason: "bootstrap_token_invalid" }
  │
  └── device token path (authOk still false + deviceTokenCandidate present)
        → rate-limit check on AUTH_RATE_LIMIT_SCOPE_DEVICE_TOKEN
        → verifyDeviceToken({ deviceId, token, role, scopes })
              ← per-device HMAC token, issued by ensureDeviceToken() at end of pairing
        → if ok: authOk=true, authMethod="device-token", reset device-token limiter
        → if fail: recordFailure on device-token scope
              reason = "device_token_mismatch" (explicit) or preserved shared-auth reason
```

**Why the device token exists:** The shared gateway token is operator-global. A
device token is per-device, per-role, per-scopes — it lets mobile devices reconnect
without re-presenting the shared secret each time. After first successful pairing,
the client stores its device token and uses it on future connections.

**Breakpoint at line 204** (the `verifyBootstrapToken` call). Fires only on first
pairing of a new device. Inspect:

- `params.deviceId` — the device being paired
- `bootstrapTokenCandidate` — the one-time token

**Breakpoint at line 239** (the `verifyDeviceToken` call). Fires on every reconnect
of an already-paired device. Inspect:

- `params.deviceId` — the reconnecting device
- `deviceTokenCandidate` — the stored device token
- `params.state.deviceTokenCandidateSource` — `"explicit-device-token"` (client
  sent `auth.deviceToken`) or `"shared-token-fallback"` (client tried shared token
  first, now retrying as device token)

---

## 6. Rate limiter (`auth-rate-limit.ts`)

**File:** `src/gateway/auth-rate-limit.ts`

### Two limiter instances (created in `server.impl.ts` line 191)

```typescript
function createGatewayAuthRateLimiters(rateLimitConfig) {
  const rateLimiter = rateLimitConfig
    ? createAuthRateLimiter(rateLimitConfig) // undefined if no config
    : undefined;
  const browserRateLimiter = createAuthRateLimiter({
    ...rateLimitConfig,
    exemptLoopback: false, // ← loopback is NEVER exempt for browser connections
  });
  return { rateLimiter, browserRateLimiter };
}
```

`rateLimiter` — general limiter; loopback addresses are exempt by default
(`exemptLoopback: true`). Only created when `gateway.auth.rateLimit` is configured.
Can be `undefined` (no rate limiting) in a basic dev setup.

`browserRateLimiter` — always created, always enforces limits even on loopback.
Used when the connection has a browser `Origin` header. This prevents a browser
running on the same machine from bypassing rate limiting just because its TCP
connection comes from 127.0.0.1.

### Which limiter is selected (from `handshake-auth-helpers.ts` line 25)

```typescript
resolveHandshakeBrowserSecurityContext(params)
  → hasBrowserOriginHeader = Boolean(requestOrigin)
  → authRateLimiter =
      hasBrowserOriginHeader && browserRateLimiter
        ? browserRateLimiter      // browser → always enforce
        : rateLimiter             // non-browser → general (may be undefined)
  → rateLimitClientIp =
      hasBrowserOriginHeader && isLoopbackAddress(clientIp)
        ? BROWSER_ORIGIN_LOOPBACK_RATE_LIMIT_IP  // "198.18.0.1" — synthetic IP
        : clientIp
```

When a browser connects from loopback (`127.0.0.1`), `hasBrowserOriginHeader` is
true and `isLoopbackAddress(clientIp)` is true. The real loopback IP would be
exempt from `browserRateLimiter` if `exemptLoopback` were `true`. To prevent this,
the code substitutes `BROWSER_ORIGIN_LOOPBACK_RATE_LIMIT_IP = "198.18.0.1"` — a
non-loopback synthetic IP — so the loopback exemption never fires for browser
connections.

### Rate limiter mechanics

```
State: Map<"scope:ip", { attempts: number[], lockedUntil?: number }>

check(ip, scope):
  → if isExempt(ip): { allowed: true, remaining: maxAttempts }
  → if lockedUntil && now < lockedUntil: { allowed: false, retryAfterMs }
  → slideWindow(): remove attempts older than windowMs
  → remaining = maxAttempts - attempts.length
  → { allowed: remaining > 0 }

recordFailure(ip, scope):
  → slideWindow(), append now to attempts
  → if attempts.length >= maxAttempts: lockedUntil = now + lockoutMs

reset(ip, scope):
  → delete the entry (clears both attempts and lockout)
```

### Three rate-limit scopes

| Scope constant                        | Value             | Used for                    |
| ------------------------------------- | ----------------- | --------------------------- |
| `AUTH_RATE_LIMIT_SCOPE_SHARED_SECRET` | `"shared-secret"` | Token/password failures     |
| `AUTH_RATE_LIMIT_SCOPE_DEVICE_TOKEN`  | `"device-token"`  | Device token failures       |
| `AUTH_RATE_LIMIT_SCOPE_HOOK_AUTH`     | `"hook-auth"`     | Hook endpoint auth failures |

Scopes share one limiter instance but maintain independent counters per IP via
the `"scope:ip"` composite key.

### Defaults

| Parameter         | Default                              | Config key                           |
| ----------------- | ------------------------------------ | ------------------------------------ |
| `maxAttempts`     | 10                                   | `gateway.auth.rateLimit.maxAttempts` |
| `windowMs`        | 60,000ms (1 min)                     | `gateway.auth.rateLimit.windowMs`    |
| `lockoutMs`       | 300,000ms (5 min)                    | `gateway.auth.rateLimit.lockoutMs`   |
| `exemptLoopback`  | `true` (general) / `false` (browser) | —                                    |
| `pruneIntervalMs` | 60,000ms                             | —                                    |

**Breakpoint at `auth-rate-limit.ts` line 196** (`entry.lockedUntil = now + lockoutMs`).
Fires when an IP is locked out. Inspect:

- `key` — the composite `"scope:ip"` key being locked
- `entry.attempts.length` — should equal `maxAttempts`
- `entry.lockedUntil` — the epoch timestamp when the lockout expires

---

## 7. Roles and scopes (`role-policy.ts`, `method-scopes.ts`)

### Roles (`role-policy.ts`)

```typescript
type GatewayRole = "operator" | "node";
```

Only two roles exist. The role is declared by the client in `connectParams.role`
and validated by `parseGatewayRole`. An invalid role string results in immediate
close with `"invalid role"`.

```typescript
function roleCanSkipDeviceIdentity(role, sharedAuthOk): boolean {
  return role === "operator" && sharedAuthOk;
}
```

An `operator` with verified shared auth (token/password/trusted-proxy) can connect
without a device identity. A `node` can never skip device identity — `node` role
always requires a paired device.

```typescript
function isRoleAuthorizedForMethod(role, method): boolean {
  if (isNodeRoleMethod(method)) {
    return role === "node"; // node-only methods blocked for operators
  }
  return role === "operator"; // all other methods require operator
}
```

Node-role methods (the 7 in `NODE_ROLE_METHODS`) are exclusively for nodes. An
operator client calling `node.pending.pull` will fail this check. A node client
calling `agent` will also fail — nodes can only call node-role methods.

### Scopes (`method-scopes.ts`)

Five scopes exist, organized from narrowest to widest effective access:

| Scope                | Constant          | Covers                                                                                              |
| -------------------- | ----------------- | --------------------------------------------------------------------------------------------------- |
| `operator.read`      | `READ_SCOPE`      | Status, session read, health, models, tools                                                         |
| `operator.write`     | `WRITE_SCOPE`     | Send, agent, chat.send, sessions.create, node.invoke                                                |
| `operator.approvals` | `APPROVALS_SCOPE` | exec/plugin approval flow                                                                           |
| `operator.pairing`   | `PAIRING_SCOPE`   | Device/node pairing management                                                                      |
| `operator.admin`     | `ADMIN_SCOPE`     | Everything: channels.logout, agents.create, skills.install, config._, cron._, sessions.delete, etc. |

**`ADMIN_SCOPE` is a superuser bypass:** `authorizeOperatorScopesForMethod` checks
for `ADMIN_SCOPE` first and returns `allowed: true` immediately for any method.

**Read is subsumed by write:** `operator.read` methods are also allowed when the
client has `operator.write`. This matches the common pattern where a write-capable
client needs read access to operate correctly.

**Default-deny for unclassified methods:** Any method not in `METHOD_SCOPE_GROUPS`
and not matching `ADMIN_METHOD_PREFIXES` resolves to `ADMIN_SCOPE` as required
scope — meaning it is admin-only by default:

```typescript
const requiredScope = resolveRequiredOperatorScopeForMethod(method) ?? ADMIN_SCOPE;
```

### `CLI_DEFAULT_OPERATOR_SCOPES`

The CLI client declares all five scopes. This is the "full access" set used by
the `openclaw` CLI and the gateway control UI. A client that declares only
`["operator.read"]` cannot call `agent` or `sessions.reset`.

**Breakpoint at `method-scopes.ts` line 214** (`authorizeOperatorScopesForMethod`).
Inspect:

- `method` — the method being called
- `scopes` — the client's declared scopes from the handshake
- `requiredScope` — what the method requires
- return value — `{ allowed: true }` or `{ allowed: false, missingScope }`

---

## 8. Control UI device-auth policy (`connect-policy.ts`)

**File:** `src/gateway/server/ws-connection/connect-policy.ts`

This file governs when the control UI is allowed to bypass the normal device-identity
requirement. Three functions matter:

### `resolveControlUiAuthPolicy` (line 13)

Reads `gateway.controlUi.allowInsecureAuth` and
`gateway.controlUi.dangerouslyDisableDeviceAuth` from config:

```
allowInsecureAuthConfigured = isControlUi && controlUiConfig.allowInsecureAuth === true
dangerouslyDisableDeviceAuth = isControlUi && controlUiConfig.dangerouslyDisableDeviceAuth === true
allowBypass = dangerouslyDisableDeviceAuth   ← NOT allowInsecureAuth
device = dangerouslyDisableDeviceAuth ? null : deviceRaw
```

`allowInsecureAuth` does NOT set `allowBypass`. It only grants a narrow exception:
local (loopback) Control UI connections without a browser secure context
(HTTP, not HTTPS) may skip device identity. Remote connections are still rejected
even with `allowInsecureAuth=true`.

`dangerouslyDisableDeviceAuth` is the break-glass: sets `allowBypass=true` and
strips the device field entirely (`device = null`), bypassing all device pairing
for operator-role Control UI clients.

### `evaluateMissingDeviceIdentity` (line 84)

The central decision function when the client presents no device identity:

```
evaluateMissingDeviceIdentity(params) → MissingDeviceIdentityDecision
  │
  ├── hasDeviceIdentity → { kind: "allow" }         (device present, not missing)
  ├── isControlUi && trustedProxyAuthOk → { kind: "allow" }
  ├── isControlUi && allowBypass && role=operator → { kind: "allow" }
  │
  ├── isControlUi && !allowBypass:
  │     if !allowInsecureAuthConfigured || !isLocalClient:
  │       → { kind: "reject-control-ui-insecure-auth" }
  │       (missing device in browser = insecure, unless explicitly opted in AND local)
  │
  ├── roleCanSkipDeviceIdentity(role, sharedAuthOk):
  │     → operator with verified shared token/password → { kind: "allow" }
  │     (CLI client: no browser, has shared token → skips device identity)
  │
  ├── !authOk && hasSharedAuth → { kind: "reject-unauthorized" }
  │     (client tried a credential but it was wrong)
  │
  └── default → { kind: "reject-device-required" }
        (no credential, no device, no bypass → must pair a device)
```

**Breakpoint at line 94** (start of `evaluateMissingDeviceIdentity`). Inspect:

- `params.hasDeviceIdentity` — did the client include a device in connect params?
- `params.sharedAuthOk` — was the shared token/password verified?
- `params.isLocalClient` — is the TCP connection from loopback?
- `params.controlUiAuthPolicy.allowBypass` — is `dangerouslyDisableDeviceAuth` on?
- The return value — which decision path was taken

---

## 9. The unauthorized flood guard (`unauthorized-flood-guard.ts`)

**File:** `src/gateway/server/ws-connection/unauthorized-flood-guard.ts`

This guard is per-connection (created once per WS session). It tracks how many
times a connected (post-handshake) client calls methods it is not authorized for:

```typescript
const DEFAULT_CLOSE_AFTER = 10;   // close connection after 10 unauthorized calls
const DEFAULT_LOG_EVERY = 100;    // suppress log noise: log every 100 after that

registerUnauthorized() → UnauthorizedFloodDecision:
  count += 1
  shouldClose = count > 10
  shouldLog   = count === 1 || count % 100 === 0 || shouldClose
  → { shouldClose, shouldLog, count, suppressedSinceLastLog }
```

In `message-handler.ts` (Session 2), the `respond()` closure calls
`unauthorizedFloodGuard.registerUnauthorized()` when an `ErrorCodes.INVALID_REQUEST`
with message starting `"unauthorized role:"` is returned. Once `count > 10`:
`queueMicrotask(() => close(1008, "repeated unauthorized calls"))`.

This closes the connection asynchronously (via `queueMicrotask`) so the error
response is sent before the close frame arrives at the client.

---

## 10. Auth flow for the four connection types

After this session you should be able to trace auth for all four connection types:

### Type 1 — CLI client (loopback, `openclaw` binary)

```
isLocalClient = true (TCP from 127.0.0.1)
hasBrowserOriginHeader = false (no Origin header from CLI)
authRateLimiter = rateLimiter (general, loopback-exempt → effectively unlimited)
resolveConnectAuthState → authorizeWsControlUiGatewayConnect
  → mode=token → safeEqualSecret(connectAuth.token, auth.token)
  → authOk=true, authMethod="token"
sharedAuthOk = true
roleCanSkipDeviceIdentity("operator", true) = true
→ no device identity required
→ setClient() with role=operator, scopes=CLI_DEFAULT_OPERATOR_SCOPES
```

### Type 2 — Web UI / browser (Control UI)

```
isLocalClient = true (opened on same host)
hasBrowserOriginHeader = true (browser sends Origin header)
authRateLimiter = browserRateLimiter (loopback NOT exempt)
rateLimitClientIp = "198.18.0.1" (synthetic, prevents loopback exemption)

origin check → checkBrowserOrigin() must pass
resolveConnectAuthState → authorizeWsControlUiGatewayConnect
  → if token provided: safeEqualSecret → authOk
device block:
  → client.device present (SubtleCrypto available in secure context)
  → nonce + signature verified
  → getPairedDevice: first connect → requirePairing("not-paired")
    → shouldAllowSilentLocalPairing = true (local, no-browser-but-isControlUi)
    → auto-approve, emit device.pair.resolved broadcast
  → ensureDeviceToken → send deviceToken in hello-ok
```

### Type 3 — Mobile node (node role)

```
role = "node"
roleCanSkipDeviceIdentity("node", *) = false  ← always needs device identity
device block: mandatory
  → verifyDeviceToken or verifyBootstrapToken
  → getPairedNode + resolveNodeCommandAllowlist
  → connectParams.commands filtered to allowlist ∩ pairedCommands
→ setClient() + nodeRegistry.register()
```

### Type 4 — HTTP REST (`/tools/invoke`)

```
Not a WS connection — no handshake
authorizeHttpGatewayConnect (authSurface="http")
  → Tailscale header auth disabled for HTTP surface
  → token/password check only
  → no device identity, no pairing
Bearer token or body.auth.token compared via safeEqualSecret
```

---

## 11. Exercises

### Exercise 1 — Trace a CLI connection auth

1. Set a breakpoint at `auth.ts` line 448 (`safeEqualSecret(connectAuth.token, auth.token)`).
2. Set a breakpoint at `auth-context.ts` line 156 (end of `resolveConnectAuthState`, just
   before `return`).
3. Start the gateway in debug mode. In Terminal 2:
   ```bash
   OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js agent --agent dev \
     --message "Say: ping" --thinking low
   ```
4. At line 448: confirm `connectAuth.token` matches `auth.token`.
5. At line 156: inspect `sharedAuthOk=true`, `bootstrapTokenCandidate=undefined`
   (CLI uses shared token, no device block runs).

### Exercise 2 — Observe `resolveConnectAuthDecision`

1. Set a breakpoint at `auth-context.ts` line 239 (`verifyDeviceToken` call).
2. This fires only when a device-token reconnect happens. To trigger it, open
   the Control UI in a browser that has previously paired.
3. Inspect `params.deviceId`, `deviceTokenCandidate`, and
   `params.state.deviceTokenCandidateSource`.

### Exercise 3 — Observe `evaluateMissingDeviceIdentity`

1. Set a breakpoint at `connect-policy.ts` line 94 (function entry).
2. Fire a CLI message (Exercise 1). At the breakpoint:
   - `params.hasDeviceIdentity` = `false` (CLI sends no device)
   - `params.sharedAuthOk` = `true`
   - Return value: `{ kind: "allow" }` via `roleCanSkipDeviceIdentity`
3. Note: this function is called even when `hasDeviceIdentity=true` — the first
   line returns `{ kind: "allow" }` immediately in that case.

### Exercise 4 — Observe scope authorization

1. Set a breakpoint at `method-scopes.ts` line 214 (`authorizeOperatorScopesForMethod`).
2. Fire a CLI message. At the breakpoint, inspect `method` and `scopes`.
3. Try calling a read-only method (`health`) and a write method (`agent`) —
   confirm `requiredScope` differs between them.
4. Check: does the CLI client's `scopes` include `ADMIN_SCOPE`? (It should.)
   Confirm `{ allowed: true }` via the admin bypass path.

---

## 12. Useful breakpoint summary

| What to catch              | File                          | Line | Inspect                                          |
| -------------------------- | ----------------------------- | ---- | ------------------------------------------------ |
| Startup auth resolved      | `auth.ts`                     | 274  | `mode`, `modeSource`, `allowTailscale`           |
| Token comparison           | `auth.ts`                     | 448  | `connectAuth.token` vs `auth.token`              |
| Auth rate-limit lockout    | `auth-rate-limit.ts`          | 196  | `key`, `entry.attempts.length`, `lockedUntil`    |
| Handshake auth state built | `auth-context.ts`             | 156  | `sharedAuthOk`, `authOk`, `deviceTokenCandidate` |
| Bootstrap token verified   | `auth-context.ts`             | 204  | `params.deviceId`, `bootstrapTokenCandidate`     |
| Device token verified      | `auth-context.ts`             | 239  | `params.deviceId`, `deviceTokenCandidateSource`  |
| Browser security context   | `handshake-auth-helpers.ts`   | 34   | `hasBrowserOriginHeader`, `rateLimitClientIp`    |
| Device signature payload   | `handshake-auth-helpers.ts`   | 116  | `payloadVersion` (v2 or v3)                      |
| Missing device decision    | `connect-policy.ts`           | 94   | `params.*`, return value `kind`                  |
| Scope authorization        | `method-scopes.ts`            | 214  | `method`, `scopes`, `requiredScope`              |
| Unauthorized flood trigger | `unauthorized-flood-guard.ts` | 31   | `count`, `shouldClose`                           |

---

## See Also

- [Session 2 — HTTP and WebSocket Transport](session2-http-ws-transport.md)
- [Session 4 — Method Dispatch and Role Authorization](session4-method-dispatch.md)
- [Gateway Source Walkthrough](gateway-source-walkthrough.md)
- [Attack Surface Map](attack-surface-map.md)
- [STRIDE Threat Model](stride-threat-model.md)
