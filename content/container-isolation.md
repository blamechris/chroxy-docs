---
title: Container Isolation
---

Chroxy supports three levels of session isolation, from lightweight sandboxing to full Docker containers, so you can constrain what an AI session can touch on the host.

## Isolation modes

- **Sandbox (no Docker)** — uses the Agent SDK's built-in isolation to restrict file system access and network operations without requiring Docker. The simplest option for basic isolation.
- **Container (full Docker)** — runs the session inside a Docker container with the project directory bind-mounted at `/workspace`. Each session gets its own container with configurable resource limits. Use this for untrusted code, multi-tenant environments, or strict resource caps.
- **Combined** — enable both; the SDK sandbox settings apply *inside* the container for defense-in-depth.

## Docker providers

Two containerized providers register automatically when `environments.enabled=true` and Docker is reachable. They wrap the standard Claude providers — see [[providers|Providers]].

| Feature | `docker-cli` (DockerSession) | `docker-sdk` (DockerSdkSession) |
|---|---|---|
| Base class | CliSession | SdkSession |
| Claude invocation | `docker exec -i <id> claude -p` | SDK `spawnClaudeCodeProcess` callback |
| Permission handling | HTTP hook (routed to host) | In-process via SDK `canUseTool` |
| Live model / permission-mode switch | Requires respawn | In-place |
| Conversation resume | No | Yes |
| Container user | root | Non-root (`chroxy`) |
| Claude Code install | Must exist in image | Auto-installed on container start |
| Plan mode | Yes | No |

Both share the same security defaults: image `node:22-slim`, 2 GB memory limit, 2-core CPU limit, 512 PID limit, all capabilities dropped (`--cap-drop ALL`), privilege escalation blocked (`--security-opt no-new-privileges`), and the host project directory mounted at `/workspace`.

## Env var allowlists

Both Docker providers forward only an explicitly allowlisted set of environment variables into the container — never the full host environment. The two allowlists differ because the providers handle permissions differently: `DockerSession` (CLI) needs `CHROXY_PORT`, `CHROXY_HOOK_SECRET`, `CHROXY_PERMISSION_MODE`, and `CLAUDE_HEADLESS` for its external HTTP permission hook, while `DockerSdkSession` (SDK) manages permissions in-process and omits them. Both forward `ANTHROPIC_API_KEY`. See the [architecture reference](https://github.com/blamechris/chroxy/blob/main/docs/architecture/reference.md) for the full allowlist table.

## Git worktree isolation

Independent of Docker, sessions can run in an isolated **git worktree** so concurrent sessions on the same repo don't step on each other's working tree. This is available from session creation on both the mobile app and dashboard.

## Persistent environments

Beyond per-session isolation, Chroxy manages **persistent container environments** — Docker Compose stacks, DevContainers, and plain containers — with snapshot and restore. These are created, listed, and destroyed over the WebSocket protocol and surfaced in the desktop dashboard's environment panel.

## See also

- [Container isolation guide (main repo)](https://github.com/blamechris/chroxy/blob/main/docs/guides/container-isolation.md) — enabling Docker, configuration, and troubleshooting
- [[configuration|Configuration]] — Kubernetes backend resource quotas and workspace PVC
- [[providers|Providers]] — the underlying Claude providers the Docker variants wrap
