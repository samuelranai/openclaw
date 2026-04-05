---
title: "Session 2 — HTTP and WebSocket Transport"
summary: "Step-by-step walkthrough of the gateway HTTP and WebSocket transport layer with VS Code debugger attached — covering port bind, upgrade handshake, preauth flood guard, connection lifecycle, and WS message dispatch"
read_when:
  - Following the Gateway Source Walkthrough, Session 2
  - Learning how HTTP upgrade and WebSocket connection lifecycle work
  - Setting up debugger breakpoints for transport-layer debugging
---

# Session 2 — HTTP and WebSocket Transport

This session covers the four files that implement the gateway's network surface:
how the HTTP server binds to a port, how an incoming HTTP connection is upgraded
to WebSocket, how each WS connection is managed before authentication, and how raw
WS messages are routed after authentication.

**Prerequisite:** Complete Session 1 (`server.impl.ts`) before starting this session.
You should already have the debugger running against a `pnpm gateway:dev` process.

---

## 1. What this session covers

| File                                                  | Lines | Role                                             |
| ----------------------------------------------------- | ----- | ------------------------------------------------ |
| `src/gateway/server/http-listen.ts`                   | 61    | HTTP `listen()` with EADDRINUSE retry            |
| `src/gateway/server-http.ts`                          | ~1120 | HTTP route handlers + WS upgrade gating          |
| `src/gateway/server/ws-connection.ts`                 | 344   | Per-connection lifecycle (before and after auth) |
| `src/gateway/server/ws-connection/message-handler.ts` | 1248  | Full WS message state machine                    |

These four files sit between the OS network stack and the method dispatch table.
Nothing in Sessions 3–7 runs until a connection clears the gates described here.

---

## 2. How the files connect to `server.impl.ts`

From Session 1, Phase 7 (`createGatewayRuntimeState`, line 712) creates the HTTP
server and WebSocket server objects in memory but does not bind them yet:

```
server.impl.ts Phase 7
  └── createGatewayRuntimeState()  (src/gateway/server-runtime-state.ts)
        ├── createGatewayHttpServer()       ← builds http.Server with all HTTP routes
        ├── listenGatewayHttpServer()       ← binds the port (http-listen.ts)
        ├── new WebSocketServer({ noServer: true })   ← wss object, not yet connected
        ├── createPreauthConnectionBudget() ← per-IP flood guard
        └── attachGatewayUpgradeHandler()   ← wires http.Server "upgrade" → wss
```

Then in Phase 10 (line 1270):

```
server.impl.ts Phase 10
  └── attachGatewayWsHandlers()  (src/gateway/server-ws-runtime.ts)
        └── attachGatewayWsConnectionHandler()  ← wires wss "connection" → per-conn lifecycle
```

So the two key event wires are:

- `http.Server "upgrade"` → `attachGatewayUpgradeHandler` → `wss.handleUpgrade` → `wss "connection"`
- `wss "connection"` → `attachGatewayWsConnectionHandler` → `attachGatewayWsMessageHandler`

---

## 3. Phase A — HTTP port bind (`http-listen.ts`)

**File:** `src/gateway/server/http-listen.ts` (61 lines)

```typescript
export async function listenGatewayHttpServer(params: {
  httpServer: HttpServer;
  bindHost: string; // "127.0.0.1" for loopback, "0.0.0.0" for LAN
  port: number; // 19001 in dev
});
```

This function does exactly one thing: call `httpServer.listen(port, bindHost)` and
wait for the `"listening"` event. The only complication is a retry loop:

```
for (attempt = 0; ; attempt++) {
  try {
    httpServer.listen(port, bindHost)
    return  // success
  } catch (EADDRINUSE) {
    if (attempt < 4:
      closeServerQuietly()
      sleep(500ms)
      continue          // port may be in TCP TIME_WAIT from previous process exit
    throw GatewayLockError("another gateway instance is already listening on ...")
  }
  throw GatewayLockError("failed to bind gateway socket ...")
}
```

Constants:

- `EADDRINUSE_MAX_RETRIES = 4` — up to 4 retries
- `EADDRINUSE_RETRY_INTERVAL_MS = 500` — 500ms between retries
- Total worst-case wait: ~2 seconds

After `EADDRINUSE_MAX_RETRIES` exhausted, the error is wrapped in `GatewayLockError`
which propagates to `runGatewayLoop` and surfaces as:

```
Gateway failed to start: another gateway instance is already listening on ws://127.0.0.1:19001
```

**Breakpoint at line 38** (`httpServer.listen(port, bindHost)`). When this fires:

- `port` — 19001 in dev
- `bindHost` — "127.0.0.1" (loopback bind)
- Inspect `attempt` — should be 0 on a clean start

---

## 4. Phase B — HTTP upgrade gating (`server-http.ts`)

**File:** `src/gateway/server-http.ts`, function `attachGatewayUpgradeHandler` (line 1007)

The WebSocket server is created with `{ noServer: true }` — it never listens on a
port itself. Instead, every incoming WebSocket upgrade request flows through the
HTTP server's `"upgrade"` event:

```
http.Server "upgrade" event
  └── attachGatewayUpgradeHandler handler
        ├── loadConfig()                      ← fresh config snapshot per upgrade
        ├── Canvas path check
        │     └── if URL matches CANVAS_WS_PATH → authorizeCanvasRequest()
        │           → if fail: writeUpgradeAuthFailure() + socket.destroy()
        ├── if wss.listenerCount("connection") === 0
        │     → 503 "Gateway websocket handlers unavailable" + socket.destroy()
        ├── preauthConnectionBudget.acquire(clientIp)
        │     → if fail: 503 "Too many unauthenticated sockets" + socket.destroy()
        ├── Transfer budget key to ws object via __openclawPreauthBudgetKey
        └── wss.handleUpgrade(req, socket, head, (ws) => wss.emit("connection", ws, req))
```

**Key security detail — preauth budget (line 1077):**
`preauthConnectionBudget.acquire()` enforces a per-IP cap of 32 simultaneous
pre-authenticated connections (default `MAX_PREAUTH_CONNECTIONS_PER_IP = 32`,
overridable via `OPENCLAW_MAX_PREAUTH_CONNECTIONS_PER_IP`). This is the primary
DoS guard — it fires before any auth logic and rejects floods at the HTTP layer
with a plain TCP-level 503, not inside the WebSocket protocol.

**Budget transfer handoff:**
The upgrade handler stamps `ws.__openclawPreauthBudgetKey` on the new WS socket
before emitting `"connection"`. When `ws-connection.ts` handles the connection
event, it reads this key and takes ownership of the budget slot. Once auth
completes (`setClient()` is called), `releasePreauthBudget()` frees the slot.
If auth never completes (handshake timeout, close), the slot is also released.

**Breakpoint at line 1077** (`preauthConnectionBudget.acquire(preauthBudgetKey)`).
Inspect:

- `preauthBudgetKey` — the client IP string used for per-IP counting
- `preauthConnectionBudget` — call `.size()` or inspect internals to see how many
  slots are in use per IP

---

## 5. Phase C — Per-connection lifecycle (`ws-connection.ts`)

**File:** `src/gateway/server/ws-connection.ts`, function `attachGatewayWsConnectionHandler`

This function handles the `wss "connection"` event. It sets up all the per-connection
state and then delegates to the message handler.

### What is set up per connection

```
wss "connection" event fires (socket, upgradeReq)
  │
  ├── connId = randomUUID()       ← unique ID for this connection's lifetime
  ├── openedAt = Date.now()       ← for duration calculation on close
  ├── handshakeState = "pending"  ← tracks: pending | connected | failed
  ├── holdsPreauthBudget = true   ← true until auth completes or connection closes
  │
  ├── Extract headers for logging (sanitized):
  │     requestHost, requestOrigin, requestUserAgent, forwardedFor, realIp
  │
  ├── Resolve canvasHostUrl       ← per-connection canvas base URL
  │
  ├── logWs("in", "open", ...)    ← structured WS log entry
  │
  ├── send connect.challenge      ← nonce sent before first message
  │     { type: "event", event: "connect.challenge", payload: { nonce, ts } }
  │
  ├── Attach socket event handlers:
  │     socket "error" → close()
  │     socket "close" → cleanup (presence, session subscriptions, node registry)
  │
  ├── handshakeTimer = setTimeout(10000ms)
  │     if fires before client = set → handshakeState="failed", close()
  │
  └── attachGatewayWsMessageHandler(...)   ← hands off all message handling
```

### Header sanitization

`sanitizeLogValue()` is applied to all headers before logging:

1. `replaceControlChars` — replaces C0 (0x00–0x1f) and C1 (0x7f–0x9f) with spaces
2. `.replace(LOG_HEADER_FORMAT_REGEX, " ")` — strips Unicode format characters (`\p{Cf}`)
3. `.replace(/\s+/g, " ").trim()` — normalize whitespace
4. `truncateUtf16Safe(300)` — cap at 300 characters

**Important:** This sanitization is **logging-only**. The raw headers are still
passed into `attachGatewayWsMessageHandler` and used for origin and IP resolution.
The gateway does not strip or modify request headers in transit.

### connect.challenge nonce

Immediately on connection, before any message is received, the gateway sends:

```json
{
  "type": "event",
  "event": "connect.challenge",
  "payload": { "nonce": "<uuid>", "ts": 1234567890 }
}
```

This nonce is used in device signature verification (Session 3). The client must
include this nonce in its device identity signature when sending the `connect` RPC.
Without the nonce the device signature is invalid.

### Handshake timeout

`getPreauthHandshakeTimeoutMsFromEnv()` returns 10,000ms (10 seconds) by default.
If the client does not send a valid `connect` request within this window, the
connection is closed with cause `"handshake-timeout"`.

**Breakpoint at line 197** (the `send(...)` for `connect.challenge`). When this
fires, inspect:

- `connId` — the UUID for this connection
- `connectNonce` — the nonce that will be sent to the client
- `handshakeState` — should be `"pending"` at this point

**Breakpoint at line 293** (the `handshakeTimer = setTimeout`). Inspect
`handshakeTimeoutMs` — confirm it is 10000 in dev.

### Connection close cleanup

On `socket "close"`, the handler runs:

```
socket "close" event
  ├── sanitize headers for log
  ├── if !client (never authenticated):
  │     warn log "closed before connect"
  ├── if client was a webchat client:
  │     info log "webchat disconnected"
  ├── if client had presenceKey:
  │     upsertPresence(presenceKey, { reason: "disconnect" })
  │     broadcastPresenceSnapshot()
  ├── context.unsubscribeAllSessionEvents(connId)
  ├── if client.connect.role === "node":
  │     nodeRegistry.unregister(connId)
  │     removeRemoteNodeInfo(nodeId)
  │     context.nodeUnsubscribeAll(nodeId)
  └── logWs("out", "close", { connId, code, durationMs, cause, ... })
```

**Breakpoint at line 231** (the `socket.once("close", ...)` callback entry). This
fires whenever a connection closes, whether gracefully or not. Inspect `closeCause`
to see why it closed (set earlier in the lifecycle by `setCloseCause`).

---

## 6. Phase D — WS message state machine (`message-handler.ts`)

**File:** `src/gateway/server/ws-connection/message-handler.ts` (1248 lines)

This is the most complex file in this session. It implements the complete WS
message lifecycle in a single `socket "message"` handler, branching on whether
the client has completed the handshake (`getClient()` returns a value).

### The two states

```
socket "message" received
  ├── [State: !client — pre-handshake]  → CONNECT path (lines ~262–1144)
  └── [State: client — post-handshake]  → DISPATCH path (lines ~1147–1227)
```

---

### 6a. Pre-handshake: CONNECT path

**Step 1 — Payload size guard (line 267):**

```typescript
if (preauthPayloadBytes > MAX_PREAUTH_PAYLOAD_BYTES) {
  close(1009, "preauth payload too large");
}
```

`MAX_PREAUTH_PAYLOAD_BYTES = 64 * 1024` (64 KB). Any pre-auth message larger than
64 KB is rejected immediately. This is a second size gate; the first is at the
WebSocket server level (`new WebSocketServer({ maxPayload: MAX_PREAUTH_PAYLOAD_BYTES })`).

**Step 2 — Frame validation (line 307):**

The first message must be exactly:

```json
{ "type": "req", "id": "<string>", "method": "connect", "params": { ... } }
```

Any other shape (wrong type, wrong method, missing params) → close 1008.

**Step 3 — Protocol version check (line 375):**

```typescript
if (maxProtocol < PROTOCOL_VERSION || minProtocol > PROTOCOL_VERSION) {
  close(1002, "protocol mismatch");
}
```

The client sends `{ minProtocol, maxProtocol }` in connect params. The gateway
checks its own `PROTOCOL_VERSION` falls within that range.

**Step 4 — Role and scope (line 391):**

```typescript
const role = parseGatewayRole(connectParams.role ?? "operator");
let scopes = Array.isArray(connectParams.scopes) ? connectParams.scopes : [];
```

Scopes are client-declared at this point. The key comment: **"Default-deny: scopes
must be explicit. Empty/missing scopes means no permissions."** Unbound scopes are
cleared (`clearUnboundScopes()`) when no paired device identity is present.

**Step 5 — Origin check (line 411):**

Runs for: browser operator UI clients, webchat clients, or any client when
`enforceOriginCheckForAnyClient` is true (browser origin detected).

```typescript
const originCheck = checkBrowserOrigin({
  requestHost, origin: requestOrigin,
  allowedOrigins: configSnapshot.gateway?.controlUi?.allowedOrigins,
  allowHostHeaderOriginFallback: ...,
  isLocalClient,
})
if (!originCheck.ok) { close(1008, "origin not allowed") }
```

If the origin was accepted via Host-header fallback (the
`dangerouslyAllowHostHeaderOriginFallback` path), a warning is logged and the
`originCheckMetrics.hostHeaderFallbackAccepted` counter is incremented.

**Step 6 — Shared auth (`resolveConnectAuthState`, line 472):**

Resolves the token/password auth result against `resolvedAuth` (the gateway's
startup-time auth configuration). Sets `authOk`, `authMethod`, `sharedAuthOk`.

**Step 7 — Device identity (lines 588–961):**

If the client presents a `device` in connect params:

```
device block
  ├── deriveDeviceIdFromPublicKey(device.publicKey) must match device.id
  ├── device.signedAt must be within ±2 minutes of now (DEVICE_SIGNATURE_SKEW_MS)
  ├── device.nonce must match connectNonce (the challenge sent at connection open)
  ├── Verify Ed25519 device signature via resolveDeviceSignaturePayloadVersion()
  └── Resolve final auth via resolveConnectAuthDecision()
        ├── if bootstrap token: verifyDeviceBootstrapToken() → new pairing path
        └── if device token:    verifyDeviceToken() → existing pairing path
```

If the device is not yet paired (`getPairedDevice` returns null):

- `requirePairing("not-paired")` is called
- For local CLI clients: silent auto-approval (`shouldAllowSilentLocalPairing`)
- For web/mobile: emits `device.pair.requested` broadcast and closes with
  `ErrorCodes.NOT_PAIRED` until the user approves in the UI

If the device is already paired but requests a role/scope upgrade:

- `requirePairing("role-upgrade")` or `requirePairing("scope-upgrade")` called
- A `security audit` line is logged with the current vs. requested access
- Connection is rejected until the user approves the upgrade

**Step 8 — Success: `hello-ok` and `setClient` (line 1043):**

When all checks pass:

```typescript
send({ type: "res", id: frame.id, ok: true, payload: {
  type: "hello-ok",
  protocol: PROTOCOL_VERSION,
  server: { version, connId },
  features: { methods: gatewayMethods, events },
  snapshot,      // gateway health + presence snapshot
  auth: { deviceToken, role, scopes, issuedAtMs },
  policy: { maxPayload: 25MB, maxBufferedBytes: 50MB, tickIntervalMs: 30s }
}})
setClient(nextClient)        // moves client from pre-auth to authenticated set
setHandshakeState("connected")
```

`setClient()` (defined in `ws-connection.ts`) calls `releasePreauthBudget()` —
this frees the preauth slot acquired in the upgrade handler.

---

### 6b. Post-handshake: DISPATCH path

After `setClient()`, all subsequent messages flow to the dispatch path:

```
socket "message" received (post-handshake)
  ├── validateRequestFrame(parsed)    ← must be { type:"req", id, method, params }
  │     if fail → send error response, return
  ├── logWs("in", "req", { connId, id, method })
  ├── Build respond() closure:
  │     ├── send { type:"res", id, ok, payload, error }
  │     ├── if unauthorized role error:
  │     │     unauthorizedFloodGuard.registerUnauthorized()
  │     │     if shouldClose: close(1008, "repeated unauthorized calls")
  │     └── logWs("out", "res", { connId, id, ok, method, ... })
  └── handleGatewayRequest({ req, respond, client, extraHandlers, context })
        └── src/gateway/server-methods.ts (Session 4)
```

**`UnauthorizedFloodGuard`:** If a connected client repeatedly calls methods it
is not authorized for, the guard tracks the count and eventually closes the
connection. This prevents a low-privilege connection from probing the method
surface with brute-force calls.

**Payload limits post-handshake:**

- `MAX_PAYLOAD_BYTES = 25 * 1024 * 1024` (25 MB) — set via `setSocketMaxPayload()`
  in the `setClient` path. This replaces the 64 KB preauth limit.
- `MAX_BUFFERED_BYTES = 50 * 1024 * 1024` (50 MB) — per-connection send buffer

---

## 7. Complete transport flow diagram

```
TCP connection arrives
  │
  ▼
http.Server "upgrade" event
  └── attachGatewayUpgradeHandler  (server-http.ts:1007)
        ├── [canvas path check]    → separate canvas WS path
        ├── wss.listenerCount check → 503 if no handlers yet
        ├── preauthConnectionBudget.acquire(clientIp)
        │     → 503 "Too many unauthenticated sockets" if at limit (32/IP)
        └── wss.handleUpgrade(req, socket, head, ...)
              └── wss.emit("connection", ws, req)
                    │
                    ▼
              attachGatewayWsConnectionHandler  (ws-connection.ts:95)
                    ├── connId = randomUUID()
                    ├── send connect.challenge { nonce, ts }
                    ├── handshakeTimer = setTimeout(10s)
                    └── attachGatewayWsMessageHandler  (message-handler.ts:136)
                          │
                          ▼ socket "message"
                          ├── [pre-auth state]
                          │     ├── preauth size check (64 KB limit)
                          │     ├── validateRequestFrame + method === "connect"
                          │     ├── protocol version check
                          │     ├── role + scope parse
                          │     ├── origin check (browser/webchat)
                          │     ├── shared auth (resolveConnectAuthState)
                          │     ├── device identity + nonce + signature
                          │     ├── device pairing check / requirePairing
                          │     ├── send hello-ok
                          │     └── setClient() → releasePreauthBudget()
                          │
                          └── [post-auth state]
                                ├── validateRequestFrame
                                ├── build respond() closure
                                │     └── UnauthorizedFloodGuard
                                └── handleGatewayRequest()  ← Session 4
```

---

## 8. Exercises

### Exercise 1 — Trace a fresh connection

1. Set a breakpoint at [server-http.ts:1077](../../../src/gateway/server-http.ts#L1077)
   (`preauthConnectionBudget.acquire`).
2. Set a breakpoint at [ws-connection.ts:197](../../../src/gateway/server/ws-connection.ts#L197)
   (the `send(...)` for `connect.challenge`).
3. Set a breakpoint at [message-handler.ts:262](../../../src/gateway/server/ws-connection/message-handler.ts#L262)
   (entry to the `socket "message"` handler).
4. Start the debugger (gateway running). In Terminal 2, send a message:
   ```bash
   OPENCLAW_STATE_DIR=~/.openclaw-dev node dist/index.js agent --agent dev \
     --message "Say: ping" --thinking low
   ```
5. The three breakpoints fire in sequence. At each:
   - **1077:** `preauthBudgetKey` (client IP), `preauthConnectionBudget`
   - **197:** `connectNonce` (the nonce the client must echo back)
   - **262:** `data` (the raw message bytes), `getClient()` → null (pre-auth)

### Exercise 2 — Observe the connect handshake

1. Set a breakpoint at [message-handler.ts:391](../../../src/gateway/server/ws-connection/message-handler.ts#L391)
   (role/scope parse).
2. Set a breakpoint at [message-handler.ts:1043](../../../src/gateway/server/ws-connection/message-handler.ts#L1043)
   (building `helloOk`).
3. Send a message as in Exercise 1.
4. At line 391: inspect `role` ("operator"), `scopes` (array of scope strings).
5. At line 1043: inspect `snapshot` — this is the full gateway state snapshot
   delivered to the client in the `hello-ok` response.

### Exercise 3 — Observe post-auth dispatch

1. Set a breakpoint at [message-handler.ts:1161](../../../src/gateway/server/ws-connection/message-handler.ts#L1161)
   (`logWs("in", "req", ...)`).
2. This fires for every RPC method call after the handshake. Inspect `req.method`
   to see which gateway method is being invoked.
3. Compare with the `gatewayMethods` array from Session 1 Exercise 3 to confirm
   the method is registered.

### Exercise 4 — Trigger the preauth budget

The default limit is 32 simultaneous pre-auth connections per IP. In normal dev
use this is never hit. To observe the rejection path without hitting the limit:

1. Set `OPENCLAW_MAX_PREAUTH_CONNECTIONS_PER_IP=1` in the launch config env and
   restart the gateway.
2. Open two browser tabs pointing at the Control UI simultaneously.
3. The second connection's upgrade request will be rejected with 503
   "Too many unauthenticated sockets".
4. Set a breakpoint at [server-http.ts:1078](../../../src/gateway/server-http.ts#L1078)
   (inside the `if (!preauthConnectionBudget.acquire...)` block) to catch the
   rejection.

---

## 9. Key constants

| Constant                               | Value     | File                             | What it limits                           |
| -------------------------------------- | --------- | -------------------------------- | ---------------------------------------- |
| `EADDRINUSE_MAX_RETRIES`               | 4         | `http-listen.ts:5`               | HTTP bind retries on port conflict       |
| `EADDRINUSE_RETRY_INTERVAL_MS`         | 500ms     | `http-listen.ts:6`               | Sleep between bind retries               |
| `DEFAULT_PREAUTH_HANDSHAKE_TIMEOUT_MS` | 10,000ms  | `handshake-timeouts.ts:1`        | Pre-auth connection lifetime             |
| `MAX_PREAUTH_CONNECTIONS_PER_IP`       | 32        | `preauth-connection-budget.ts:1` | Simultaneous unauth sockets per IP       |
| `MAX_PREAUTH_PAYLOAD_BYTES`            | 64 KB     | `server-constants.ts:5`          | Max message size before auth             |
| `MAX_PAYLOAD_BYTES`                    | 25 MB     | `server-constants.ts:3`          | Max message size after auth              |
| `MAX_BUFFERED_BYTES`                   | 50 MB     | `server-constants.ts:4`          | Per-connection send buffer               |
| `TICK_INTERVAL_MS`                     | 30,000ms  | `server-constants.ts:24`         | Heartbeat tick interval (sent to client) |
| `DEVICE_SIGNATURE_SKEW_MS`             | 120,000ms | `message-handler.ts:103`         | Max device signature age                 |

---

## 10. Useful breakpoint summary

| What to catch             | File                               | Line | Inspect                                                 |
| ------------------------- | ---------------------------------- | ---- | ------------------------------------------------------- |
| HTTP port bind            | `server/http-listen.ts`            | 38   | `port`, `bindHost`, `attempt`                           |
| Preauth budget gate       | `server-http.ts`                   | 1077 | `preauthBudgetKey`, budget state                        |
| connect.challenge sent    | `server/ws-connection.ts`          | 197  | `connId`, `connectNonce`                                |
| Handshake timeout armed   | `server/ws-connection.ts`          | 293  | `handshakeTimeoutMs`                                    |
| Connection close cleanup  | `server/ws-connection.ts`          | 231  | `closeCause`, `durationMs`                              |
| message handler entry     | `ws-connection/message-handler.ts` | 262  | `data`, `getClient()` null = pre-auth                   |
| Preauth size check        | `ws-connection/message-handler.ts` | 267  | `preauthPayloadBytes`                                   |
| Protocol version check    | `ws-connection/message-handler.ts` | 375  | `minProtocol`, `maxProtocol`                            |
| Origin check              | `ws-connection/message-handler.ts` | 411  | `enforceOriginCheckForAnyClient`, `isBrowserOperatorUi` |
| Shared auth result        | `ws-connection/message-handler.ts` | 472  | `authOk`, `authMethod`, `sharedAuthOk`                  |
| Device nonce check        | `ws-connection/message-handler.ts` | 627  | `providedNonce` vs `connectNonce`                       |
| hello-ok built            | `ws-connection/message-handler.ts` | 1043 | `snapshot`, `helloOk.features.methods`                  |
| setClient (auth complete) | `ws-connection/message-handler.ts` | 1080 | `nextClient.connect.role`, `nextClient.connId`          |
| Post-auth dispatch        | `ws-connection/message-handler.ts` | 1161 | `req.method`, `client.connect.role`                     |

---

## See Also

- [Session 1 — server.impl.ts](session1-server-impl.md)
- [Gateway Source Walkthrough](gateway-source-walkthrough.md)
- [Attack Surface Map](attack-surface-map.md)
- [STRIDE Threat Model](stride-threat-model.md)
