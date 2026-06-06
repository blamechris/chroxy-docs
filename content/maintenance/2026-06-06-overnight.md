---
title: "Overnight maintenance session — 2026-06-06"
date: 2026-06-06
tags: [maintenance, autonomous, session-log]
---

# Overnight maintenance session — 2026-06-06

Autonomous, self-paced run kicked off before bed. Mode: **aggressive**. Authorized
outward actions: PR the cleanup branch ✓, file issues from audits, open fix PRs (never
merged), push updates to this vault. **Nothing gets merged** — every PR is left for the
morning review.

This page is the live journal. The loop appends to the **Activity log** each iteration.

## Guardrails (self-imposed)

- Never commit to `main`; branch from an up-to-date `main` per task. Never merge any PR.
- Tests + lints must pass before opening a PR. New server tests **must** pass a temp
  `stateFilePath` (project rule #4633). Forward every `BaseSession` opt (#4797).
- Zero attribution in commits / PRs / issues.
- Skip anything needing live claude-TUI recording, human judgement, or an irreversible
  action — log it under *Deferred* instead.
- Soft cap ~6 fix PRs overnight to keep the morning review surface sane; unlimited
  issue-filing. Favour quality (self-review each diff) over volume.

## Starting state

- Branch `main` @ baseline; PR **#5237** open (coverage-audit cleanup: smoke tooling fix,
  Control Room Playwright smoke coverage, skill refresh).
- 16 open issues. Good first targets: **#5218** (Investigate-seed leak test — matches the
  audit's App.tsx gap), **#5230** (keychain-unavailable reconciliation), the audit's P1/P2
  (3 cheap Maestro flows: child-agent #5135, resume-unknown #4971, session-stopped #4879;
  fix 2 stale Maestro flows).
- Full coverage-audit report (local, gitignored): `chroxy/docs/audit-results/coverage-audit-2026-06-06.html`.

## Plan per iteration

1. `git checkout main && git pull`. Pick the next item: open issue (highest value/safest
   first under aggressive scope) or audit backlog item.
2. Branch, implement, test, self-review the diff, open a PR (never merge).
3. When the queue thins or I hit a blocker, run a `bug-hunt` / `project-audit` swarm and
   file the findings as issues.
4. Append to the Activity log below; push this vault.
5. Schedule the next wake-up.

## Activity log

- **00:00** — Session bootstrapped. PR #5237 opened and verified (live smoke 17/17,
  Control Room table rendered, session-state intact). Journal created. Selecting first
  work item.
- **03:21** — **#5218 done → PR #5238.** Added the deferred Investigate-seed no-leak
  regression test (App.test.tsx). Gave the `CreateSessionModal` mock an `onCreate` seam;
  positive control proves the seed lands, then header-overflow + command-palette plain
  openers must leave the composer empty. **Mutation-verified**: reverting the `seed || null`
  clear in `openCreateSession` fails both leak tests while the positive control still passes.
  Full App.test.tsx 97/97 green. Test-only (App.tsx untouched). Next: pick another item.
- **03:34** — **#5229 done → PR #5239.** Implemented credential data-key rotation. Added
  `rotateMasterKey`/`setMasterKey` (cipher), `rekeyCredentialStore()` (store: decrypt →
  rotate → atomic re-encrypt, with keychain rollback on write failure), and a
  `chroxy credentials rekey [--json]` CLI mirroring `worktree gc`. 13 tests incl. the
  write-failure rollback + read-error-leaves-key-untouched paths; both CI lints + eslint
  green. CLI verified via `--help` only (did NOT run the real rotation — it would touch the
  real keychain). Crash-window across file+keychain documented in the PR for review.
- **03:46** — Safe auto-fixable issue queue is dry (rest need judgement/admin/live-TUI/epics —
  see Deferred). Ran a **bug-hunt swarm** (15 agents: 6 finders over recently-changed modules →
  adversarial verify each candidate, refute-by-default). **9 real bugs confirmed** (reproduced
  where possible), filed as #5240–#5248. Headline: **#5244 (HIGH, data loss)** — `worktree gc`'s
  clean check ignores gitignored files, so `worktree remove` silently deletes node_modules/.env;
  auto-triggers on startup with `autoReap`. **Top fix candidate for the next iteration.** Other
  notables: #5247 activity-reducer stack-overflow DoS of the Control Room render; #5248 background-
  shell mtime false-reap re-opens the #4307 idle-timeout bug; #5242 encrypted creds silently
  resolve null on a recoverable keychain error (spawns unauthenticated).
- **03:52** — **#5244 fixed → PR #5249** (the HIGH data-loss bug). `isClean()` now runs
  `git status --porcelain --ignored` so a worktree holding only gitignored content
  (node_modules/.env) is skipped, not removed. 2 new real-temp-repo tests; **mutation-verified**
  (reverting `--ignored` deletes the worktree + its secret → both tests fail). worktree-gc 24/24,
  both lints green. No real `worktree gc --apply` run — all exercised in /tmp repos. **3 of ~6
  fix PRs.** Next: #5247 (activity-reducer recursion) or #5240 (gh --limit).

## Deferred / needs human

- **#5230** (keychain-unavailable: plaintext fallback vs refuse-to-store) — security-posture decision; could break no-keychain hosts. Needs your call.
- **#5155** (should pairing-bound tokens overwrite credentials?) — security policy question, not a mechanical fix.
- **#3816** (tighten branch protection / Actions fork-PR) — repo-admin + irreversible settings; do interactively.
- **#3808** (sign Windows MSI) — needs signing certs/secrets/infra.
- **#4880 / #4882** (empirical TUI byte-sequence re-recording) — require a live claude TUI; can't drive headless.
- **#3954 / #3955 / #3956** (claude-channel bridge chain) — #3954 needs a live claude session (per project memory); #3955/#3956 are downstream of it.
- **#2661 / #5159 / #3699** — large epics, not a single safe PR.
- **#3840** (revisit gating *if* a directory-trust UI is added) — conditional/future; not actionable yet.
- Audit-backlog Maestro flows (child-agent #5135, resume-unknown #4971, session-stopped #4879) — writable but E2E-verification needs a booted simulator + Metro; not reliably runnable headless. Left for a hands-on pass.

## PRs opened tonight

- [#5237](https://github.com/blamechris/chroxy/pull/5237) — chore: coverage-audit cleanup (foundation, opened pre-loop)
- [#5238](https://github.com/blamechris/chroxy/pull/5238) — test(dashboard): Investigate-seed no-leak regression (closes #5218), mutation-verified
- [#5239](https://github.com/blamechris/chroxy/pull/5239) — feat(server): `chroxy credentials rekey` — rotate the at-rest data key (closes #5229)
- [#5249](https://github.com/blamechris/chroxy/pull/5249) — fix(server): worktree gc must not delete gitignored content (closes #5244, HIGH data-loss), mutation-verified

## Issues filed tonight

From the bug-hunt swarm (all adversarially verified, `audit-finding` label):

- **[#5244](https://github.com/blamechris/chroxy/issues/5244) — HIGH / data-loss**: worktree gc clean-check ignores gitignored files → `worktree remove` deletes node_modules/.env (auto on startup). **Fix next.**
- [#5247](https://github.com/blamechris/chroxy/issues/5247) — activity-reducer unbounded recursion → RangeError crashes Control Room render (DoS).
- [#5248](https://github.com/blamechris/chroxy/issues/5248) — background-shell mtime reap false-positives on output-then-idle procs (re-opens #4307).
- [#5242](https://github.com/blamechris/chroxy/issues/5242) — encrypted creds silently resolve null on recoverable keychain error → unauthenticated spawn (related to #5230).
- [#5240](https://github.com/blamechris/chroxy/issues/5240) — `gh pr list` lacks `--limit` → Control Room PR rollup capped at 30.
- [#5245](https://github.com/blamechris/chroxy/issues/5245) — worktree gc failed-remove leaves worktree permanently unlocked.
- [#5241](https://github.com/blamechris/chroxy/issues/5241) — survey missing maxBuffer mislabels a large dirty repo as "not a git repo".
- [#5243](https://github.com/blamechris/chroxy/issues/5243) — win32 credential write unlinks before rename (crash-window data loss).
- [#5246](https://github.com/blamechris/chroxy/issues/5246) — worktree gc transient stat failure inflates reclaimed count.
