---
title: Providers
---

Chroxy runs AI coding sessions through pluggable **providers**. Each provider wraps a different AI backend behind the same WebSocket / event contract, so the mobile app and dashboard behave identically regardless of which one you pick. See [[architecture|Architecture]] for how providers fit into the session pipeline.

## Built-in providers

- **`claude-sdk`** *(default)* — Claude Code via the `@anthropic-ai/claude-agent-sdk` (in-process). Fastest startup, live model/mode switching, resume, and thinking-level control. The right choice for most users.
- **`claude-cli`** — Legacy `claude -p` subprocess. Pick this when you need **plan mode**. Same billing as the SDK; permission handling routes through an HTTP hook.
- **`claude-tui`** — Interactive `claude` TUI driven under a PTY. Bills against your Claude subscription's **interactive allowance** instead of the programmatic credit pool. Trade-offs: no live streaming (one burst at turn end), no live model switch, no plan mode, no resume, no attachments, no cost reporting.
- **`claude-channel`** *(research preview, scaffold only)* — will drive Claude through Anthropic's first-party channels MCP protocol (`claude --channels`): subscription billing like `claude-tui`, but a documented protocol plus live streaming and a first-party permission relay. Not yet runnable — `start()` throws until the live bridge lands.
- **`gemini`** — Google Gemini CLI (`gemini -p`). No permissions, no plan mode, no resume, no attachments.
- **`codex`** — OpenAI Codex CLI (`codex exec`). No permissions, no plan mode, no resume, no attachments, no conversation continuity.
- **`docker-cli` / `docker-sdk`** — containerized wrappers of `claude-cli` / `claude-sdk`. Register automatically only when `environments.enabled=true` and Docker is reachable. See [[container-isolation|Container Isolation]].

## Credentials

The default Claude provider uses your existing `claude` CLI login, so no extra setup is needed. Gemini and Codex require explicit keys:

| Provider | Env var | Where to get a key |
|----------|---------|--------------------|
| Claude *(default)* | `ANTHROPIC_API_KEY` *(optional)* | https://console.anthropic.com/settings/keys |
| Gemini | `GEMINI_API_KEY` | https://aistudio.google.com/apikey |
| Codex (OpenAI) | `OPENAI_API_KEY` | https://platform.openai.com/api-keys |

Keys can come from the shell environment, an inline prefix when starting the server, or the **Settings → Provider Credentials** pane in the dashboard. Dashboard-saved keys are written to `~/.chroxy/credentials.json` (mode `0600`, owner-only) and never shown again after saving. Resolution order is **env > store > unset** — an exported shell variable always wins.

Setting `ANTHROPIC_API_KEY` bypasses your Claude subscription's programmatic credit pool and bills the raw API account directly. The Claude CLI/TUI providers deliberately strip `ANTHROPIC_API_KEY` from their spawn env so they keep using subscription / OAuth auth.

## Choosing a Claude backend

| Feature | `claude-sdk` | `claude-cli` | `claude-tui` |
|---------|:------------:|:------------:|:------------:|
| In-process permissions | Yes | No (HTTP hook) | No (HTTP hook) |
| Live model switch | Yes | Yes (restart) | — |
| Live permission-mode switch | Yes | Yes (restart) | — |
| Plan mode | No | **Yes** | No |
| Resume | Yes | No | No |
| Thinking-level control | Yes | No | No |
| Live streaming | Yes | Yes | No (deliver-on-complete) |
| Auth | API key or `claude login` | API key or `claude login` | `claude login` only |
| Billing | Programmatic credits / API | Programmatic credits / API | **Subscription interactive allowance** |
| Startup overhead | None (in-process) | One subprocess spawn / session | One PTY warmup (~3.5s) / session |

- **`claude-sdk`** — default, most-featured, programmatic billing.
- **`claude-cli`** — pick only when you need plan mode.
- **`claude-tui`** — pick when you want sessions billed against your Claude.ai Pro / Max / Team subscription rather than the programmatic credit pool.

## Capability matrix

| Capability | `claude-sdk` | `claude-cli` | `claude-tui` | `codex` | `gemini` |
|------------|:-:|:-:|:-:|:-:|:-:|
| Permission handling | Yes (in-process) | Yes (HTTP hook) | Yes (HTTP hook) | — | — |
| Live model switch | Yes | Yes | — | Yes | Yes |
| Live permission-mode switch | Yes | Yes | — | — | — |
| Plan mode | — | **Yes** | — | — | — |
| Resume | Yes | — | — | — | — |
| Thinking-level control | Yes | — | — | — | — |
| Live streaming | Yes | Yes | No | Yes | Yes |
| Attachments | Yes | Yes | — | — | — |
| Agent tracking | Yes | Yes | — | — | — |
| Cost reporting | Yes | Yes | — | — | — |
| Conversation continuity | Yes | Yes | Yes | **No** | **No** |

A `—` in a capability row means the provider reports `false`; in a behavioural row it means the feature is unimplemented. Most provider-agnostic UI (session tabs, dual chat/terminal view, push notifications, conversation search, dashboard) works across all providers.

## Selecting a provider

Precedence, highest first: **CLI flag > environment variable > config file > default (`claude-sdk`)**.

```bash
npx chroxy start --provider claude-cli
CHROXY_PROVIDER=gemini npx chroxy start
OPENAI_API_KEY=sk-... npx chroxy start --provider codex
```

Clients can also override the default per session by passing `provider` in a `create_session` WebSocket message. If a session is created for a provider whose key isn't set, the server returns a clear error (e.g. *"Codex: required credential not set — OPENAI_API_KEY"*).

## Billing note

The Claude SDK and `claude -p` paths count as **programmatic usage**, which draws from a separate monthly credit pool on Claude subscriptions rather than the interactive Claude Code allowance. Set `ANTHROPIC_API_KEY` to bill the raw API account directly, or use `claude-tui` to bill against the interactive subscription allowance. Prompt caching is on by default and typically cuts credit burn substantially on long sessions. See [[configuration|Configuration]] for `CHROXY_COST_BUDGET` and `CHROXY_SESSION_TIMEOUT` cost controls.

## See also

- [Providers reference (main repo)](https://github.com/blamechris/chroxy/blob/main/docs/providers.md) — per-provider install, sandbox surfaces, and known limits
- [[configuration|Configuration]] — provider selection keys and cost controls
- [[container-isolation|Container Isolation]] — the Docker provider variants
