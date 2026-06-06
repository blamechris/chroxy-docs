---
title: Configuration
---

Chroxy resolves configuration from multiple sources with a clear precedence order. Run `npx chroxy init` to generate `~/.chroxy/config.json` with an API token and default settings, then override per-deployment or per-run as needed.

## Precedence

Highest priority first:

1. **CLI flags** — passed to `npx chroxy start`
2. **Environment variables** — system env
3. **Config file** — `~/.chroxy/config.json`
4. **Defaults** — built-in values

Use `--verbose` to see exactly where each resolved value came from. Unknown keys and type mismatches log non-fatal warnings at startup.

## Core keys

| Key | CLI flag | Env var | Description |
|-----|----------|---------|-------------|
| `apiToken` | — | `API_TOKEN` | Client authentication token |
| `port` | — | `PORT` | Local WebSocket port (default `8765`) |
| `provider` | `--provider <name>` | `CHROXY_PROVIDER` | Default session backend (default `claude-sdk`) — see [[providers|Providers]] |
| `model` | `--model <name>` | `CHROXY_MODEL` | Model to use; provider-specific (aliases like `sonnet`/`opus`/`haiku` resolve to full IDs) |
| `cwd` | `--cwd <path>` | `CHROXY_CWD` | Working directory (CLI mode) |
| `resume` | `--resume` / `-r` | `CHROXY_RESUME` | Resume existing session |
| `noAuth` | `--no-auth` | `CHROXY_NO_AUTH` | Disable authentication (loopback dev only) |
| `costBudget` | `--cost-budget <dollars>` | `CHROXY_COST_BUDGET` | Per-session cost budget; warns at 80%, pauses at 100% |

## Tunnel modes

| Mode | Flag | Description |
|------|------|-------------|
| Quick Tunnel | *(default)* | Random URL, no account needed. URL changes on restart. |
| Named Tunnel | `--tunnel named` | Stable URL that survives restarts. Requires a Cloudflare account + domain. |
| No Tunnel | `--tunnel none` | Local only. Use with `--no-auth` for development. |

Set up a Named Tunnel interactively with `npx chroxy tunnel setup`. The tunnel URL is randomized but the **API token is the real secret** — protect it and rotate it (`npx chroxy init` regenerates) if leaked.

## Cost budget

`costBudget` is applied independently to each session, not as a shared pool. The server emits `budget_warning` at 80% of the budget and `budget_exceeded` at 100%, which **pauses** the session. A client sends `resume_budget` to unpause. Separately, a one-time soft toast fires when a session's cumulative cost crosses a configurable threshold (default $5).

## Inactivity safety net

Sessions are protected by a two-stage timer pair that fires only when the server has heard nothing from the AI CLI for the configured window:

| Stage | Key | Env var | Default | Behaviour |
|-------|-----|---------|---------|-----------|
| Soft | `resultTimeoutMs` | `CHROXY_RESULT_TIMEOUT_MS` | 30 min | Emits `inactivity_warning` + push notification; session stays alive |
| Hard | `hardTimeoutMs` | `CHROXY_HARD_TIMEOUT_MS` | 2 h | Expires outstanding permission prompts, clears busy state, aborts the in-flight query, and emits a timeout error |

Both share the range 30s–24h. The split lets operators keep the warning noisy while leaving the kill generous so a long-running Bash build is not murdered mid-flight. While a permission prompt is outstanding, both timers pause and re-arm on resolution.

## Skills configuration

Several keys govern the [[skills|Skills]] system: `maxSkillBytes` (per-skill cap, default 32KB), `maxTotalSkillBytes` (global context budget, default 256KB, drops lower-priority skills first), `providerSkillAllowlist` (restrict which skills load for non-Claude providers), and `trustMismatchMode` (`warn` / `block` — opts into a per-skill SHA-256 tamper ledger).

## Skip permissions (TUI only)

`dangerouslySkipPermissions` (CLI `--dangerously-skip-permissions`, env `CHROXY_DANGEROUSLY_SKIP_PERMISSIONS`) is a **TUI-only** opt-out from Chroxy's permission gate. When enabled, `claude-tui` sessions spawn with `--dangerously-skip-permissions` and the permission hook is elided. A loud `[security]` warning is logged at boot. Other providers ignore the flag.

## `--no-auth` trust model

`--no-auth` is a **dev-only** mode. It binds to `127.0.0.1` only, skips tunnel startup, disables mDNS, and auto-authenticates loopback clients. It must never be paired with a tunnel or a non-loopback bind, and `chroxy dev` refuses to start with it.

## Kubernetes (advanced)

When the K8s environment backend is active, `environments.k8s` supports a pre-provisioned workspace PVC (`workspace.claimName`), per-Pod CPU/memory requests and limits, and namespace-level `ResourceQuota` / `LimitRange` guardrails. These are opt-in operator-side concerns — see the [full configuration guide](https://github.com/blamechris/chroxy/blob/main/packages/server/CONFIG.md) for the block shapes.

## See also

- [Configuration guide (main repo)](https://github.com/blamechris/chroxy/blob/main/packages/server/CONFIG.md) — every key, validation rules, and worked examples
- [[providers|Providers]] — provider selection and credentials
- [[security|Security & Encryption]] — the auth and encryption model
