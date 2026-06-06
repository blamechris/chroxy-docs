---
title: Chroxy Documentation
---

Chroxy is a remote terminal and chat front-end for AI coding CLIs. You run a lightweight daemon on your dev machine and connect to it from your phone or desktop over a secure Cloudflare tunnel. Every session gives you two synchronized views of the same agent: a full xterm.js terminal and a clean chat UI that parses the CLI's stream-json output into readable messages. Pluggable session providers let you drive Claude Code (Agent SDK, legacy CLI, interactive TUI), Google Gemini, or OpenAI Codex behind one identical WebSocket protocol, so the mobile app and dashboard work the same regardless of which backend you pick.

## Quick Links

- [[architecture|Architecture]] — server, tunnel, WebSocket protocol, and client topology
- [[providers|Providers]] — Claude SDK / CLI / TUI, Gemini, Codex, and the capability matrix
- [[configuration|Configuration]] — config precedence, keys, cost budgets, and timeouts
- [[security|Security & Encryption]] — auth tokens, end-to-end encryption, permission gating
- [[container-isolation|Container Isolation]] — sandbox, Docker, and worktree isolation
- [[skills|Skills]] — reusable per-session instruction snippets
- [[developer-guide|Developer Guide]] — repo layout, dev commands, and how to add a provider

## Key facts

- **What it is:** a self-hosted remote terminal + chat client for Claude Code, Gemini, and Codex sessions, reachable from a phone or desktop.
- **Two views, one session:** switch between a markdown-rendered chat UI and a full terminal emulator over the same connection.
- **Multi-provider:** `claude-sdk` (default), `claude-cli`, `claude-tui`, `gemini`, `codex`, plus containerized `docker-sdk` / `docker-cli`. A `claude-channel` provider is in research preview.
- **Stack:** Node.js 22 server (ES modules), a React + Vite web dashboard bundled into the server, a Tauri (Rust) desktop tray app, and a React Native / Expo mobile app. Communication is WebSocket over a Cloudflare tunnel with end-to-end encryption (TweetNaCl).
- **Layout:** npm-workspace monorepo under `packages/` — `server`, `dashboard`, `desktop`, `app`, `protocol`, `store-core`.
- **License:** MIT.

## How to run it

```bash
git clone https://github.com/blamechris/chroxy.git
cd chroxy
npm install

# Install and configure (generates an API token)
PATH="/opt/homebrew/opt/node@22/bin:$PATH" npx chroxy init

# Start the server
PATH="/opt/homebrew/opt/node@22/bin:$PATH" npx chroxy start
```

The server prints a QR code and a tunnel URL. Scan the QR with the mobile app or open the dashboard URL in a browser. On the same WiFi you can skip the tunnel entirely with `npx chroxy start --tunnel none`. Run `npx chroxy doctor` to diagnose missing dependencies.

See the [[developer-guide|Developer Guide]] for prerequisites (Node 22, cloudflared) and per-client build steps.

## Source of truth

This wiki mirrors the documentation in the main Chroxy repository. The authoritative source is the code and `docs/` tree at [github.com/blamechris/chroxy](https://github.com/blamechris/chroxy).
