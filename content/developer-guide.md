---
title: Developer Guide
---

This page covers running Chroxy locally, the repo layout, and how to extend it. For the system overview see [[architecture|Architecture]].

## Prerequisites

- **Node.js 22+** — required for the server. macOS: `brew install node@22`; Windows: `winget install OpenJS.NodeJS.LTS`; Linux: use nvm/fnm.
- **cloudflared** — Cloudflare's tunnel client (no account needed for Quick Tunnels). macOS: `brew install cloudflared`.
- **Claude Code CLI** — needed by the Claude providers; `claude --version` should resolve. Authenticate with `claude login` or set `ANTHROPIC_API_KEY`.

Run `npx chroxy doctor` to verify Node, `claude`, and `cloudflared` are present and configured.

## Repository layout

This is an npm-workspace monorepo. One `npm install` at the root covers all packages.

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

The server is plain JavaScript ES modules (no TypeScript, no semicolons, single quotes, EventEmitter-based). The app and dashboard are strict TypeScript with functional components, hooks, and a Zustand store.

## Running the stack

```bash
git clone https://github.com/blamechris/chroxy.git
cd chroxy
npm install

# Terminal 1 — server (Node 22)
PATH="/opt/homebrew/opt/node@22/bin:$PATH" npx chroxy start

# Terminal 2 — mobile app hot-reload (from packages/app/)
npx expo start

# Terminal 3 (optional) — desktop dev (from packages/desktop/)
cargo tauri dev
```

The mobile app requires a **custom dev build** (not Expo Go) because it bundles native modules — `npx expo run:ios` / `npx expo run:android`, or an EAS cloud build. The desktop app is a Tauri tray app wrapping the dashboard; it needs the Rust toolchain plus Tauri's system libraries.

Use `npx chroxy dev` (instead of `start`) when iterating on Chroxy itself — it forces supervisor mode (auto-restart on crash) and requires a tunnel.

## Tests

```bash
# Server tests (Node 22)
cd packages/server && PATH="/opt/homebrew/opt/node@22/bin:$PATH" npm test

# Dashboard tests (Vitest)
cd packages/dashboard && npm test

# App type check
cd packages/app && npx tsc --noEmit

# Lint
cd packages/server && npm run lint
```

Server tests must never touch real user state — every test constructing a `SessionManager` must pass a temp `stateFilePath`, enforced by both a runtime sandbox guard and a CI lint. Provider session classes that extend `BaseSession` must forward every base constructor opt through `super({ ... })`, also enforced by a CI lint.

## Adding a provider

The provider registry (`packages/server/src/providers.js`) is a plain object literal mapping provider names to session classes. To add one:

1. Write a session class that extends `EventEmitter` (or `BaseSession` / `JsonlSubprocessSession`) and exposes `start` / `destroy` / `sendMessage` / `interrupt` / `setModel` / `setPermissionMode`, plus a static `capabilities` getter.
2. Register it in the registry literal.

`sdk-session.js` and `cli-session.js` are the worked examples. The static `capabilities` object is what the registry and clients inspect at runtime to decide which UI affordances (model switch, plan mode, resume, etc.) to show. See [[providers|Providers]] for the capability contract.

## Extension points

| You want to add… | Where it goes |
|------------------|---------------|
| A new session provider | `packages/server/src/providers.js` + a new session class |
| A new tunnel adapter | `packages/server/src/tunnel/` + `registerTunnel` in the registry |
| A new WebSocket message | A Zod schema in `ws-schemas.js` + a handler in `ws-message-handlers.js` |
| A new skill | A Markdown file in `~/.chroxy/skills/` or `<repo>/.chroxy/skills/` — see [[skills|Skills]] |

## See also

- [[architecture|Architecture]] — components, data flow, and the WebSocket protocol
- [[providers|Providers]] — the provider capability matrix
- [[configuration|Configuration]] — config keys and precedence
- [Main repository](https://github.com/blamechris/chroxy) — source of truth, including `CONTRIBUTING.md`
