---
title: Architecture
---

Chroxy is a self-hosted daemon that bridges remote clients to a local AI coding CLI. The data path is the same regardless of which AI backend or client you use:

```
Mobile App / Desktop  <->  Cloudflare Tunnel  <->  WebSocket Server  <->  Session Provider  <->  AI CLI (Claude / Gemini / Codex)
```

## Components

- **Server** (`packages/server`) — a Node.js daemon (ES modules, no TypeScript). `server-cli.js` starts a WebSocket server and creates sessions through pluggable providers. It also serves the web dashboard as a static React SPA.
- **WebSocket layer** (`ws-server.js`) — handles authentication, end-to-end encryption, message routing, session management, and permission handling.
- **Tunnel** — a Cloudflare Quick or Named tunnel exposes the server for secure remote access without port forwarding. See [[configuration|Configuration]] for tunnel modes.
- **Supervisor** — when a tunnel is in use, the supervisor owns the tunnel and auto-restarts the server on crash with exponential backoff.
- **Clients** — the React Native mobile app and the Tauri desktop tray app both connect over WebSocket. The web dashboard is served directly by the server and shares the same protocol.

## Session providers

The server never talks to an AI CLI directly. It selects a **provider** — a session class that wraps one backend behind a uniform contract — via the `--provider` flag, the `CHROXY_PROVIDER` env var, the config file, or per-session at creation time. The built-in providers include `claude-sdk` (Agent SDK, in-process), `claude-cli` (legacy `claude -p` subprocess), `claude-tui` (interactive `claude` under a PTY), `gemini`, `codex`, and the containerized `docker-sdk` / `docker-cli`.

The provider registry lives in `packages/server/src/providers.js` as a plain object literal mapping provider names to session classes. Every session class extends `EventEmitter` and exposes `start` / `destroy` / `sendMessage` / `interrupt` / `setModel` / `setPermissionMode`, plus a static `capabilities` getter the registry inspects at runtime. See [[providers|Providers]] for the full capability matrix.

## Data flow

```
[Mobile App / Desktop] <-WebSocket-> [Cloudflare] <-> [WsServer]
                                                          |
                                                 [Session Provider]
                                                /         |         \
                                      [CliSession]  [SdkSession]  [Docker*Session]
                                           |              |              |
                                      [claude -p]   [Agent SDK]   [docker exec -> claude]
                                           |              |              |
                                               [Streaming JSON Events]
```

CLI and SDK events are normalized into a single event format (`event-normalizer.js`) before being broadcast over the WebSocket, which is why the chat and terminal views render identically across providers.

## WebSocket protocol

The protocol is bidirectional and validated with Zod schemas (`ws-schemas.js`). It carries far more than chat text — clients drive the entire session lifecycle over it.

**Client to server** messages include `auth`, `key_exchange`, `create_session`, `switch_session`, `destroy_session`, `rename_session`, `input`, `interrupt`, `set_model`, `set_permission_mode`, `set_permission_rules`, `set_thinking_level`, `permission_response`, file operations (`browse_files`, `read_file`, `write_file`, `list_files`), git operations (`git_status`, `git_diff`, `git_stage`, `git_unstage`, `git_commit`, `git_branches`), conversation history (`list_conversations`, `resume_conversation`, `search_conversations`), checkpoints, environments, cost queries, and push-token registration.

**Server to client** messages include `auth_ok` / `auth_fail`, `key_exchange_ok`, streaming (`stream_start`, `stream_delta`, `stream_end`), `message`, `tool_start` / `tool_result`, `permission_request` / `permission_resolved`, plan-mode events (`plan_started`, `plan_ready`), agent tracking (`agent_spawned`, `agent_completed`, `agent_busy` / `agent_idle`), cost and usage updates (`cost_update`, `session_usage`, `budget_warning` / `budget_exceeded`), session and timeout events, and `server_shutdown` / `server_status` lifecycle signals.

`auth_ok` carries a `protocolVersion` integer (currently `2`), bumped on breaking wire-shape changes so newer servers can degrade gracefully for older clients. The full message catalog and per-message payload details live in the project's [architecture reference](https://github.com/blamechris/chroxy/blob/main/docs/architecture/reference.md).

## Broadcast scoping

Messages are either session-scoped or global. Session-scoped messages (`stream_delta`, `model_changed`, `permission_mode_changed`, `dev_preview`, normalizer events, and others) are delivered only to clients viewing or subscribed to that session. Global messages (`session_list`, `session_activity`, `client_joined` / `client_left`, `token_rotated`, `available_models`, timeout warnings, and server-status events) go to every connected client. Clients subscribe to non-active sessions with `subscribe_sessions` for background monitoring.

## Repository layout

```
chroxy/
├── packages/
│   ├── server/      # Node.js daemon, CLI, and bundled web dashboard server
│   ├── dashboard/   # Web dashboard (React + Vite) — built into the server bundle
│   ├── desktop/     # Tauri tray app (Rust) wrapping the dashboard
│   ├── app/         # React Native mobile app (TypeScript, Expo 54)
│   ├── protocol/    # Shared WebSocket protocol types and Zod schemas
│   └── store-core/  # Shared store logic and crypto for app + dashboard
├── docs/            # Setup guides, architecture, provider reference
└── scripts/         # Install and tooling helpers
```

The mobile app and dashboard both use a Zustand store driven by a shared `ConnectionPhase` state machine (`disconnected -> connecting -> connected / reconnecting / server_restarting`) for resilient auto-reconnect.

## See also

- [[providers|Providers]] — what each backend can and cannot do
- [[security|Security & Encryption]] — the auth and encryption layers on the WebSocket
- [[developer-guide|Developer Guide]] — how to add a provider or run the stack locally
- [Architecture reference (main repo)](https://github.com/blamechris/chroxy/blob/main/docs/architecture/reference.md) — full component and protocol tables
