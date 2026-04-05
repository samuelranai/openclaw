# Plugin Architecture (today)

This note is a maintainer-oriented map of how OpenClaw plugins are packaged, discovered, loaded, and integrated into the Gateway. It is intentionally structured for “dig in by file” reading.

## 1) What OpenClaw calls a plugin

At runtime, a plugin is a module (or set of modules) that is imported by the Gateway and executes in-process. The module registers capabilities through a typed API surface (tools, channels, providers, gateway handlers, hooks, routes, etc.).

Key characteristic: plugins run with the same OS privileges as the Gateway process.

## 2) Packaging model

### 2.1 Plugin manifest: `openclaw.plugin.json`

Native plugins typically ship a manifest file in the plugin root:

- `openclaw.plugin.json` (examples in `extensions/*/openclaw.plugin.json`)
- Manifest types/schema helpers: `src/plugins/manifest.ts`

The manifest is used to describe plugin identity and capabilities (and, for some plugin kinds, configuration schema and integration metadata) without requiring the plugin runtime to execute.

### 2.2 Package metadata: `package.json` `openclaw` block (common)

Many plugins are also normal Node packages and include a `package.json` with an `openclaw` block that can declare runtime entrypoints.

The discovery layer treats this as “how to find entry modules” and, in some cases, a way to advertise optional “setup” entrypoints.

## 3) Discovery: where plugins come from

Discovery is responsible for turning “paths on disk” into “plugin candidates” (without necessarily executing them).

Primary implementation:

- `src/plugins/discovery.ts`
- Roots and configured search paths: `src/plugins/roots.ts`

Discovery can accept plugins as either:

- A single entry file (`.ts/.js/.mjs/.cjs/...`)
- A directory package that provides entrypoints via `package.json` (or conventional `index.*` fallbacks)

### 3.1 Manifest registry: read capabilities without running code

To decide which plugin IDs exist and what they claim to provide (channels, providers, etc.) without importing plugin code, OpenClaw builds a catalog:

- `src/plugins/manifest-registry.ts`

This registry is used by gateway startup selection logic to resolve “which plugin IDs should load” based on config + available manifests.

## 4) Loading: importing and executing plugins

Loading is the transition from “candidate found on disk” to “module imported and registered”.

Core loader:

- `src/plugins/loader.ts`

The loader:

- Resolves discovery inputs and caches results
- Imports plugin entry modules (TypeScript/JS) at runtime
- Provides SDK aliasing so plugin code can import `openclaw/plugin-sdk/*` as the supported boundary
- Instantiates a registry and runs each plugin’s registration function to populate capabilities

Key related modules:

- Registry + API wiring: `src/plugins/registry.ts`, `src/plugins/types.ts`, `src/plugins/api-builder.ts`
- SDK alias support: `src/plugins/sdk-alias.ts`

## 5) Plugin entry shape: what the module exports

Plugin authors typically implement an entry module using SDK helpers that produce a consistent shape for the loader.

Primary entry helper:

- `src/plugin-sdk/plugin-entry.ts` (`definePluginEntry`)

Channel plugin helper:

- `src/plugin-sdk/core.ts` (`defineChannelPluginEntry`)

The channel helper typically supports “setup vs full” registration modes so a channel can do minimal startup wiring early and complete heavier registration later.

## 6) Gateway integration points (startup flow)

The gateway owns “when plugin loading happens” and which plugin outputs are hooked into the request handling surfaces.

Key integration points:

- Gateway startup and orchestration: `src/gateway/server.impl.ts`
- Gateway plugin bootstrap wrapper: `src/gateway/server-plugin-bootstrap.ts`
- Gateway plugin loading entrypoint: `src/gateway/server-plugins.ts`

Common patterns you’ll see in the gateway:

- Loading a minimal set of plugins early so gateway methods/handlers exist before networking starts
- Merging plugin-provided gateway handlers into the gateway’s handler map
- Loading some plugin sets later (deferred) once the gateway is listening

## 7) Capability registration surface (what plugins can do)

Plugins register capabilities through the `OpenClawPluginApi` surface.

Where to study the API surface and how it is constructed:

- API types and surface: `src/plugins/types.ts`
- API builder: `src/plugins/api-builder.ts`
- Registry implementation and “what registration actually does”: `src/plugins/registry.ts`

Typical capability categories:

- Gateway methods / handlers (RPC-like server surfaces)
- Tools and tool policies
- Channels (inbound/outbound messaging transports)
- Providers (models, web search, speech, media understanding, etc.)
- Hooks (plugin hook handlers)
- HTTP routes (plugin-owned HTTP endpoints)
- Background services / sidecars (where applicable)

## 8) Suggested reading path (fastest to deep)

If you want to build an accurate mental model quickly:

1. Gateway startup integration:
   - `src/gateway/server.impl.ts`
   - `src/gateway/server-plugins.ts`
2. Discovery and manifest catalog:
   - `src/plugins/discovery.ts`
   - `src/plugins/manifest-registry.ts`
   - `src/plugins/manifest.ts`
3. Loader and runtime boundaries:
   - `src/plugins/loader.ts`
   - `src/plugins/registry.ts`
4. Plugin author-facing SDK:
   - `src/plugin-sdk/plugin-entry.ts`
   - `src/plugin-sdk/core.ts`
