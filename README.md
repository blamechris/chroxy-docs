# chroxy-docs

Living documentation for [Chroxy](https://github.com/blamechris/chroxy) — a self-hosted remote terminal and chat client for AI coding CLIs (Claude Code, Gemini, Codex), reachable from your phone or desktop over a secure Cloudflare tunnel.

Built with [Quartz 4](https://quartz.jzhao.xyz/) over an Obsidian vault (`content/`). Deployed to GitHub Pages and served at **https://www.blamechris.com/chroxy-docs/**.

## Local development

```bash
npm ci
npx quartz build --serve   # preview at http://localhost:8080
```

## Deploy

Push to `main` — the Quartz GitHub Pages workflow builds and publishes to the `gh-pages` branch automatically.

## Source of truth

The canonical documentation lives in the main chroxy repo at [github.com/blamechris/chroxy](https://github.com/blamechris/chroxy) (README, `CONFIG.md`, and the `docs/` tree). This wiki mirrors it; see `CLAUDE.md` for the sync protocol.
