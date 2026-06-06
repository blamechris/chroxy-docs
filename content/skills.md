---
title: Skills
---

Skills are reusable instruction snippets that Chroxy injects into every session you start. They let you encode personal conventions, project standards, or recurring guidance once and have every provider (Claude SDK, Claude CLI, Codex, Gemini) automatically apply it.

## Where skill files live

Chroxy looks in two places and merges them at session start:

1. **Global** — `~/.chroxy/skills/*.md` — applied to every session on the machine.
2. **Repo overlay** — `<repo>/.chroxy/skills/*.md` — discovered by walking up from the session's working directory (same pattern as `.git`). Applied only to sessions started inside that repo.

When a global file and a repo file share the same filename, the **repo file wins** — treat it as an override, not an addition.

## Writing a skill

A skill file is just Markdown. The filename (without `.md`) becomes the skill name, and the first non-empty line is used as a short description in the `list_skills` response. No frontmatter is required.

Disable a skill by renaming it to end in `.disabled.md`; rename it back to re-enable. Skills are loaded fresh at every session start, so no restart is needed.

## How injection works per provider

| Provider | Injection method |
|----------|-----------------|
| Claude SDK (`sdk` / Docker) | Appended to the system prompt via `systemPrompt.append` |
| Claude CLI (`cli`) | Passed as `--append-system-prompt` at process start |
| Codex | Prepended to the first user message of the session |
| Gemini | Prepended to the first user message of the session |

Skills are combined under a `# User skills` header, separated by `---` dividers, and sent once per session.

## Trust and safety

Repo-overlay skills are auto-loaded whenever you start a session inside that repo's tree and are injected directly into the model's system prompt. That means opening an **untrusted** repo can shape model behaviour, including potential prompt injection. Treat `<repo>/.chroxy/skills/` like any other code in the repo and review it before working in an unfamiliar checkout.

Several [[configuration|Configuration]] keys harden this: `maxSkillBytes` and `maxTotalSkillBytes` cap individual and total skill size; `providerSkillAllowlist` restricts which skills load for non-Claude providers (which lack Claude's tool-gating layer); and `trustMismatchMode` (`warn` / `block`) opts into a per-skill SHA-256 ledger that flags silent post-review tampering. Third-party skills live under a `community/<author>/` subdirectory and are subject to a first-activation trust prompt.

## Listing active skills

Send a `list_skills` WebSocket message from any connected client to receive the current skill list, including each skill's name, description, and `source` (`global` or `repo`).

## See also

- [Skills guide (main repo)](https://github.com/blamechris/chroxy/blob/main/docs/skills.md) — community namespace, trust file migration, and platform notes
- [[configuration|Configuration]] — skill byte budgets, allowlists, and trust modes
