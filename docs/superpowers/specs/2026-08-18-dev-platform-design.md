# Development Platform — Design Spec

**Date:** 2026-08-18
**Status:** Approved (ready for implementation planning)
**Owner:** akhil

## Purpose

Stand up a reusable development *platform* — source control, CI/CD, code review,
and planning workflow — **before** building any apps, so every future project
inherits a correct, consistent pipeline for free and additional developers can
join with minimal onboarding.

The guiding principle is **minimum viable platform**: build the correct baseline
without ceremony a small/new team won't use yet, structured so heavier pieces
(automated versioning, staging→prod gates) drop in later without rework.

## Decisions (locked)

| Decision | Choice | Rationale |
|---|---|---|
| Hosting target | Mixed / host-agnostic | Will build varied apps, CLIs, tools; deploy is a swappable last step |
| Collaboration hub | GitHub-native (Issues + Projects) | One login, Issues↔PRs↔Projects auto-link, CI in same repo |
| Repo strategy | Polyrepo + template repo | Strong isolation per app; template + central workflows keep config DRY |
| GitHub setup | Personal account (for now) | Migratable to an org later; reusable workflows + templates work today |
| Branching model | Trunk-based + PR previews | Simplest that is still "correct" for a small team |
| Scope tier | Standard + Dependabot/CodeQL | Correct baseline; Full-tier pieces are additive later |

### Known caveat — protection enforcement on private repos
Enforced required-reviews / required-status-checks on **private** repos
historically requires **GitHub Pro (~$4/mo)** or the repos being **public**.
Repository rulesets cover more on free plans now. Decide per-repo at setup;
the ruleset configuration is identical either way. Confirm exact enforcement
when wiring the first repo.

## Architecture — three kinds of repos

```
akhil (personal GitHub account)
├── .github          Platform repo: reusable workflows, shared issue/PR
│                    templates, CODEOWNERS defaults, label definitions,
│                    CONTRIBUTING.md
├── template-app     Scaffold new projects are created FROM (template repo flag)
├── app-foo          Real project, created from template-app
├── tool-bar         Real project, created from template-app
└── ...
```

- **`.github` repo** — single source of truth for pipeline *logic*. Each project's
  workflow file is a thin (~10 line) caller: `uses: akhil/.github/.github/workflows/ci.yml@v1`.
  Fix CI once, every repo benefits.
- **`template-app`** — the *structure* a new project needs: thin caller workflows,
  lint/format/test config, `CODEOWNERS`, PR template, `.gitignore`, README
  skeleton, and a `justfile`/`Makefile` of common commands.
- **Real project repos** — lean; reference the platform rather than copying it.

**Platform versioning:** `.github` reusable workflows are git-tagged (`v1`).
Projects pin to `@v1`; breaking CI changes ship as `@v2` and repos migrate
deliberately.

## CI/CD pipeline (staged, in reusable workflow)

Runs on every PR, fail-fast, cheap-first:

| # | Stage | What runs | Gate |
|---|---|---|---|
| 1 | Lint/format | ESLint/Prettier or ruff/black/gofmt (language-detected) | required |
| 2 | Typecheck | `tsc --noEmit` / mypy / etc. | required |
| 3 | Unit tests | Jest/Vitest/pytest + coverage | required |
| 4 | Integration | tests needing services (DB, etc.) via service containers | required |
| 5 | Build | production build succeeds | required |
| 6 | Preview deploy | per-project deploy step → posts preview URL on PR | swappable |
| 7 | Security | Dependabot (deps) + CodeQL (code scanning) | Dependabot auto-PRs; CodeQL required |

Design choices:
- **Stages 1–5 are language-agnostic** via workflow inputs (e.g. `language: node`),
  so the same pipeline serves a Next.js app and a Python CLI.
- **Stage 6 (deploy) is the single swappable seam** — Vercel today (GitHub
  integration); swap for container/AWS later without touching stages 1–5.
  Preview-on-PR enables "see it before merge."
- Integration/e2e can be **path-filtered** so trivial PRs skip full cost.
- Merge to `main` re-runs the pipeline and triggers production deploy.

## Code review & branch protection

- **`main` protected** via repository ruleset: no direct pushes; PR required;
  required status checks (stages 1–5 + CodeQL green); branches up to date before
  merge; linear history via **squash-merge only**.
- **1 required approving review**; dismiss stale approvals on new commits.
- **`CODEOWNERS`** auto-requests reviewers. Solo today = akhil; adding a teammate
  pulls them into relevant PRs with no process change.
- **PR template**: what/why, linked issue (`Closes #NNN`), testing done,
  screenshots for UI.

## Planning & collaboration (GitHub-native)

- **Issue templates** (in `.github`): 🐞 Bug, ✨ Feature request, 💡 Idea/RFC,
  🔧 Chore — structured forms.
- **GitHub Projects (v2)** — one board across all repos:
  `Backlog → Ready → In progress → In review → Done`. Fields: priority,
  size/estimate, iteration (sprint). Built-in automation: issue opened → Backlog;
  PR opened → In review; PR merged → Done.
- **Sprint planning** via the Projects **Iteration** field (timeboxed cycles,
  burn-down). No extra tool.
- **Ideas not yet ready** = 💡 RFC issues in a `Backlog`/`Icebox` view.
- **Labels** standardized across repos from `.github` (type, priority, status).

## Scaffolding a new project

1. "Use this template" from `template-app` → new repo.
2. Run `scripts/bootstrap.sh` once: rename placeholders, set language input in the
   caller workflow, apply the ruleset, link the repo to the shared Project, apply
   label set — via `gh` CLI.
3. Start coding.

**Success target:** new project → first green PR with preview URL in **under 10
minutes**, zero copy-paste.

## Onboarding new devs

- **`CONTRIBUTING.md`** in `.github` (inherited by every repo): branch naming,
  commit/PR conventions, running tests locally, the review flow.
- New dev added as collaborator, reads one doc; guardrails (protection + CI +
  CODEOWNERS) teach the workflow by enforcing it.

## Explicitly out of scope (deferred to "Full" tier)

- Conventional Commits + automated changelog / release-please / changesets
- Staging→prod manual promotion gate
- Migration from personal account to a GitHub Organization

These are additive and do not require rework of the Standard baseline.

## Success criteria

1. A new repo created from `template-app` has working CI, branch protection,
   preview deploys, and Project-board integration on its first PR.
2. Pipeline logic is defined once (in `.github`) and referenced, not copied.
3. A new developer can go from zero to a merged PR using only `CONTRIBUTING.md`.
4. The deploy step can be swapped (Vercel → container/AWS) without editing
   test/lint/build stages.
