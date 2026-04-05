---
title: "Gateway Security — Study Materials"
summary: "Reading plan, session guides, synthesis, tool inventory, and analysis documents produced during the OpenClaw gateway security study"
read_when:
  - Orienting to the gateway security study materials
  - Looking for a specific session guide, analysis document, or tool reference
---

# Gateway Security — Study Materials

These documents were produced during a structured security study of the OpenClaw
gateway source code. They are not part of the official product documentation.

---

## Start Here

| Document                                                       | Purpose                                                                                                                     |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| [architecture-synthesis.md](architecture-synthesis.md)         | **Best first read.** Cross-source synthesis — full component map, 7-layer defense, 4-layer memory, command queue, MCP gaps. |
| [gateway-source-walkthrough.md](gateway-source-walkthrough.md) | The session reading plan. Describes which files to read and how layers connect.                                             |
| [attack-surface-map.md](attack-surface-map.md)                 | Entry-point inventory — all surfaces where external input reaches the gateway.                                              |
| [stride-threat-model.md](stride-threat-model.md)               | STRIDE analysis across all gateway components, with priority matrix and enhancement proposals.                              |
| [tool-security-inventory.md](tool-security-inventory.md)       | Security-classified inventory of all agent tools — reachability, approval requirements, risk class, high-risk chains.       |

---

## Session Guides (`sessions/`)

Seven structured study guides, one per architectural layer. Read in order.

| Session | Topic                                  | File                                                                                     |
| ------- | -------------------------------------- | ---------------------------------------------------------------------------------------- |
| 1       | Startup and Wiring                     | [sessions/session1-server-impl.md](sessions/session1-server-impl.md)                     |
| 2       | HTTP and WebSocket Transport           | [sessions/session2-http-ws-transport.md](sessions/session2-http-ws-transport.md)         |
| 3       | Authentication                         | [sessions/session3-authentication.md](sessions/session3-authentication.md)               |
| 4       | Method Dispatch and Role Authorization | [sessions/session4-method-dispatch.md](sessions/session4-method-dispatch.md)             |
| 5       | Agent Loop and Chat                    | [sessions/session5-agent-loop-and-chat.md](sessions/session5-agent-loop-and-chat.md)     |
| 6       | Tool Dispatch                          | [sessions/session6-tool-dispatch.md](sessions/session6-tool-dispatch.md)                 |
| 7       | Sessions and Channels                  | [sessions/session7-sessions-and-channels.md](sessions/session7-sessions-and-channels.md) |

Each guide includes: key concepts, file-and-line breakpoints, exercises, and security observations.

---

## Development Setup

For building and running the gateway locally during study:
[docs/help/study/dev-local-build.md](../../help/study/dev-local-build.md)

---

## See Also

- [Official security hardening guide](../index.md)
- [Plugin security study](../../security/study/README.md)
