# Development Platform Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a reusable GitHub-based development platform (reusable CI/CD workflows, a project template, branch protection, code review, and planning boards) so every future app inherits a correct pipeline for free.

**Architecture:** A central `.github` repo holds all pipeline *logic* (reusable workflows) and account-wide community-health defaults (issue/PR templates, CONTRIBUTING). A `template-app` repo holds project *structure*: thin caller workflows that pin the reusable workflows at a version tag, plus lint/test/build config and a `bootstrap.sh` that applies branch-protection rulesets, labels, and board linkage via the `gh` CLI. New projects are created from `template-app`. A smoke-test project proves the whole chain end-to-end.

**Tech Stack:** GitHub (personal account), GitHub Actions (reusable workflows via `workflow_call`), repository rulesets, GitHub Projects v2, CodeQL, Dependabot, `gh` CLI, `actionlint`. Default project language: TypeScript/Node (Vitest + ESLint + Prettier + tsc); the reusable CI also supports Python.

## Global Constraints

- GitHub is a **personal account** (owner login referred to below as `$OWNER`). No GitHub-Organization features — use **repository rulesets**, not org rulesets. Determine the value once: `OWNER=$(gh api user -q .login)`.
- **All repos are created `--private`** (user decision). On a free plan, ruleset enforcement on private repos may be limited; Tasks 8/10/12 attempt it and must report honestly whether protection actually took effect. Do not claim enforcement works without evidence from `gh pr checks` / a rejected push.
- All caller workflows **pin reusable workflows to the `v1` tag** (`uses: $OWNER/.github/.github/workflows/ci.yml@v1`), never `@main`.
- Reusable CI must be **language-agnostic via inputs** (`language: node|python`); stages 1–5 must not hardcode a stack.
- The **deploy step is the single swappable seam** — no test/lint/build stage may depend on the deploy provider.
- Merges to `main` are **squash-merge only**; `main` requires PR + passing required checks + 1 approving review.
- **Private-repo caveat:** enforced required-reviews/checks on a *private* repo needs GitHub Pro or a public repo. The ruleset JSON is identical either way; when applying to a private repo on a free plan, verify enforcement and note it.
- Every task ends with a **verification command and expected output**, then a **commit**. Prefer `actionlint` for fast local workflow validation before pushing.

**Prerequisites (verify before Task 1):**
- `gh auth status` shows an authenticated account with `repo`, `workflow`, `read:org`, and `project` scopes. If missing project scope: `gh auth refresh -s project,read:project`.
- `actionlint` installed (`brew install actionlint`).
- Working directory: `~/dev-platform` (already a git repo holding the spec + this plan).

---

### Task 1: Create the `.github` platform repo skeleton

**Files:**
- Create (local clone): `~/dev-platform/repos/.github/README.md`
- Create: `~/dev-platform/repos/.github/.github/workflows/.gitkeep`

**Interfaces:**
- Produces: a pushed GitHub repo `$OWNER/.github` on branch `main`, cloned at `~/dev-platform/repos/.github`. All later `.github` tasks commit into this clone.

- [ ] **Step 1: Capture owner and create the repo**

```bash
OWNER=$(gh api user -q .login)
mkdir -p ~/dev-platform/repos
gh repo create "$OWNER/.github" --private \
  --description "Platform: reusable workflows + account-wide defaults" \
  --clone
mv "$OWNER/.github" ~/dev-platform/repos/.github 2>/dev/null || true
cd ~/dev-platform/repos/.github
```

- [ ] **Step 2: Add skeleton files**

```bash
mkdir -p .github/workflows
printf '# %s/.github\n\nPlatform repo: reusable workflows and account-wide community-health defaults.\n' "$OWNER" > README.md
touch .github/workflows/.gitkeep
```

- [ ] **Step 3: Verify the repo exists and is empty of workflows**

Run: `gh repo view "$OWNER/.github" --json name,visibility -q '.name + " " + .visibility'`
Expected: prints `.github public`

- [ ] **Step 4: Commit and push**

```bash
git add -A
git commit -m "chore: initialize .github platform repo"
git branch -M main
git push -u origin main
```

---

### Task 2: Reusable CI workflow (stages 1–5)

**Files:**
- Create: `~/dev-platform/repos/.github/.github/workflows/ci.yml`

**Interfaces:**
- Produces: reusable workflow callable as `$OWNER/.github/.github/workflows/ci.yml@v1` with inputs `language` (default `node`), `node-version` (default `20`), `python-version` (default `3.12`), `run-integration` (default `true`). Runs jobs: `lint`, `typecheck`, `unit`, `integration`, `build`.

- [ ] **Step 1: Write the reusable workflow**

```yaml
# .github/workflows/ci.yml
name: CI
on:
  workflow_call:
    inputs:
      language:
        description: "node | python"
        type: string
        default: node
      node-version:
        type: string
        default: "20"
      python-version:
        type: string
        default: "3.12"
      run-integration:
        type: boolean
        default: true

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - if: inputs.language == 'node'
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
      - if: inputs.language == 'node'
        run: npm ci
      - if: inputs.language == 'node'
        run: npm run lint
      - if: inputs.language == 'python'
        uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - if: inputs.language == 'python'
        run: pip install ruff && ruff check .

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - if: inputs.language == 'node'
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
      - if: inputs.language == 'node'
        run: npm ci
      - if: inputs.language == 'node'
        run: npm run typecheck
      - if: inputs.language == 'python'
        uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - if: inputs.language == 'python'
        run: pip install mypy && mypy .

  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - if: inputs.language == 'node'
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
      - if: inputs.language == 'node'
        run: npm ci
      - if: inputs.language == 'node'
        run: npm run test:unit -- --coverage
      - if: inputs.language == 'python'
        uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - if: inputs.language == 'python'
        run: pip install -e '.[dev]' && pytest -m 'not integration' --cov

  integration:
    if: inputs.run-integration
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 10s --health-timeout 5s --health-retries 5
    env:
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/postgres
    steps:
      - uses: actions/checkout@v4
      - if: inputs.language == 'node'
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
      - if: inputs.language == 'node'
        run: npm ci
      - if: inputs.language == 'node'
        run: npm run test:integration
      - if: inputs.language == 'python'
        uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - if: inputs.language == 'python'
        run: pip install -e '.[dev]' && pytest -m integration

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - if: inputs.language == 'node'
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          cache: npm
      - if: inputs.language == 'node'
        run: npm ci
      - if: inputs.language == 'node'
        run: npm run build
      - if: inputs.language == 'python'
        uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - if: inputs.language == 'python'
        run: pip install build && python -m build
```

- [ ] **Step 2: Lint the workflow (this is the failing-then-passing check)**

Run: `cd ~/dev-platform/repos/.github && actionlint .github/workflows/ci.yml`
Expected: before writing the file this path did not exist; now it exits `0` with no output. (If actionlint reports real errors, fix them before committing.)

- [ ] **Step 3: Commit and push**

```bash
git add .github/workflows/ci.yml
git commit -m "feat: add reusable CI workflow (lint/typecheck/unit/integration/build)"
git push
```

---

### Task 3: Reusable security workflow (CodeQL) + Dependabot template

**Files:**
- Create: `~/dev-platform/repos/.github/.github/workflows/codeql.yml`
- Create: `~/dev-platform/repos/.github/workflow-templates/dependabot.yml` (template shipped to projects, not run in `.github`)

**Interfaces:**
- Consumes: nothing.
- Produces: reusable workflow `$OWNER/.github/.github/workflows/codeql.yml@v1` with input `languages` (default `javascript-typescript`). Dependabot config template consumed later by `template-app` (Task 8).

- [ ] **Step 1: Write the CodeQL reusable workflow**

```yaml
# .github/workflows/codeql.yml
name: CodeQL
on:
  workflow_call:
    inputs:
      languages:
        description: "Comma-separated CodeQL languages, e.g. javascript-typescript,python"
        type: string
        default: javascript-typescript

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read
      actions: read
    strategy:
      fail-fast: false
      matrix:
        language: ${{ fromJSON(format('["{0}"]', join(fromJSON(format('[\"{0}\"]', replace(inputs.languages, ',', '\",\"'))), '","'))) }}
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: ${{ matrix.language }}
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
```

> Note: if the matrix expression above fails actionlint or CodeQL, replace it with a single-language form: set `matrix.language: [ javascript-typescript ]` literally and expose multi-language later. Simpler is acceptable — the seam is the input.

- [ ] **Step 2: Write the Dependabot template**

```yaml
# workflow-templates/dependabot.yml  (copied into projects as .github/dependabot.yml)
version: 2
updates:
  - package-ecosystem: npm
    directory: "/"
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
  - package-ecosystem: github-actions
    directory: "/"
    schedule:
      interval: weekly
```

- [ ] **Step 3: Verify**

Run: `cd ~/dev-platform/repos/.github && actionlint .github/workflows/codeql.yml`
Expected: exits `0`. (Apply the fallback in the note if the matrix line errors.)

- [ ] **Step 4: Commit and push**

```bash
git add .github/workflows/codeql.yml workflow-templates/dependabot.yml
git commit -m "feat: add reusable CodeQL workflow and dependabot template"
git push
```

---

### Task 4: Account-wide issue & PR templates

**Files:**
- Create: `~/dev-platform/repos/.github/.github/ISSUE_TEMPLATE/bug.yml`
- Create: `~/dev-platform/repos/.github/.github/ISSUE_TEMPLATE/feature.yml`
- Create: `~/dev-platform/repos/.github/.github/ISSUE_TEMPLATE/idea.yml`
- Create: `~/dev-platform/repos/.github/.github/ISSUE_TEMPLATE/chore.yml`
- Create: `~/dev-platform/repos/.github/.github/ISSUE_TEMPLATE/config.yml`
- Create: `~/dev-platform/repos/.github/.github/PULL_REQUEST_TEMPLATE.md`

**Interfaces:**
- Produces: default community-health files inherited by any of `$OWNER`'s repos that lack their own.

- [ ] **Step 1: Write the four issue-form templates**

```yaml
# .github/ISSUE_TEMPLATE/bug.yml
name: "🐞 Bug report"
description: Something is broken
labels: ["type:bug"]
body:
  - type: textarea
    id: what
    attributes: { label: What happened?, description: Include steps to reproduce }
    validations: { required: true }
  - type: textarea
    id: expected
    attributes: { label: Expected behavior }
    validations: { required: true }
  - type: input
    id: env
    attributes: { label: Environment (OS, version, browser) }
```

```yaml
# .github/ISSUE_TEMPLATE/feature.yml
name: "✨ Feature request"
description: Propose new functionality
labels: ["type:feature"]
body:
  - type: textarea
    id: problem
    attributes: { label: Problem, description: What need does this address? }
    validations: { required: true }
  - type: textarea
    id: proposal
    attributes: { label: Proposed solution }
    validations: { required: true }
```

```yaml
# .github/ISSUE_TEMPLATE/idea.yml
name: "💡 Idea / RFC"
description: An early idea not yet ready to build
labels: ["type:idea"]
body:
  - type: textarea
    id: idea
    attributes: { label: The idea }
    validations: { required: true }
  - type: textarea
    id: open-questions
    attributes: { label: Open questions }
```

```yaml
# .github/ISSUE_TEMPLATE/chore.yml
name: "🔧 Chore"
description: Maintenance, refactor, or tooling
labels: ["type:chore"]
body:
  - type: textarea
    id: task
    attributes: { label: What needs doing? }
    validations: { required: true }
```

- [ ] **Step 2: Write the issue-template config**

```yaml
# .github/ISSUE_TEMPLATE/config.yml
blank_issues_enabled: false
```

- [ ] **Step 3: Write the PR template**

```markdown
<!-- .github/PULL_REQUEST_TEMPLATE.md -->
## What & why

<!-- Summary of the change and the motivation -->

Closes #

## Testing

<!-- How did you verify this? Commands run, results. -->

## Screenshots

<!-- For UI changes -->

## Checklist
- [ ] Linked the related issue
- [ ] Added/updated tests
- [ ] CI is green
```

- [ ] **Step 4: Verify YAML parses**

Run: `cd ~/dev-platform/repos/.github && for f in .github/ISSUE_TEMPLATE/*.yml; do python3 -c "import sys,yaml; yaml.safe_load(open('$f'))" && echo "ok $f"; done`
Expected: `ok` printed for each of the 5 files, no traceback.

- [ ] **Step 5: Commit and push**

```bash
git add .github/ISSUE_TEMPLATE .github/PULL_REQUEST_TEMPLATE.md
git commit -m "feat: add account-wide issue and PR templates"
git push
```

---

### Task 5: CONTRIBUTING.md, CODEOWNERS default, and label definitions

**Files:**
- Create: `~/dev-platform/repos/.github/CONTRIBUTING.md`
- Create: `~/dev-platform/repos/.github/.github/CODEOWNERS`
- Create: `~/dev-platform/repos/.github/labels.json`

**Interfaces:**
- Produces: `labels.json` consumed by the label-sync step in `bootstrap.sh` (Task 9). Signature: a JSON array of `{name, color, description}` objects.

- [ ] **Step 1: Write CONTRIBUTING.md**

```markdown
# Contributing

## Branching
- `main` is protected. Never push to it directly.
- Branch names: `feat/<slug>`, `fix/<slug>`, `chore/<slug>`.

## Workflow
1. Create/assign an issue; move it to **In progress** on the board.
2. Branch, commit in small steps.
3. Open a PR; fill the template; link the issue with `Closes #NNN`.
4. CI must be green and one review approved before squash-merge.

## Running tests locally
- `npm ci` then `npm test` (or `just test`). See the project README for stack-specific commands.

## Reviews
- CODEOWNERS auto-requests reviewers. Address comments with follow-up commits (approvals reset on new commits).
```

- [ ] **Step 2: Write the default CODEOWNERS**

```bash
OWNER=$(gh api user -q .login)
printf '* @%s\n' "$OWNER" > .github/CODEOWNERS
```

- [ ] **Step 3: Write labels.json**

```json
[
  { "name": "type:bug",      "color": "d73a4a", "description": "Something is broken" },
  { "name": "type:feature",  "color": "0e8a16", "description": "New functionality" },
  { "name": "type:idea",     "color": "fbca04", "description": "Early idea / RFC" },
  { "name": "type:chore",    "color": "cfd3d7", "description": "Maintenance / tooling" },
  { "name": "priority:high", "color": "b60205", "description": "Do soon" },
  { "name": "priority:med",  "color": "d93f0b", "description": "Normal priority" },
  { "name": "priority:low",  "color": "fef2c0", "description": "Someday" },
  { "name": "status:blocked","color": "5319e7", "description": "Blocked on something" }
]
```

- [ ] **Step 4: Verify**

Run: `cd ~/dev-platform/repos/.github && python3 -c "import json; print(len(json.load(open('labels.json'))), 'labels')" && cat .github/CODEOWNERS`
Expected: prints `8 labels` and a CODEOWNERS line `* @<owner>`.

- [ ] **Step 5: Commit and push**

```bash
git add CONTRIBUTING.md .github/CODEOWNERS labels.json
git commit -m "feat: add CONTRIBUTING, default CODEOWNERS, and label definitions"
git push
```

---

### Task 6: Tag the platform `v1`

**Files:** none (git tag on `$OWNER/.github`).

**Interfaces:**
- Consumes: all reusable workflows from Tasks 2–3.
- Produces: the `v1` tag that every caller workflow pins to.

- [ ] **Step 1: Create and push the tag**

```bash
cd ~/dev-platform/repos/.github
git tag -a v1 -m "Platform v1: reusable CI + CodeQL"
git push origin v1
```

- [ ] **Step 2: Verify the tag resolves**

Run: `gh api "repos/$OWNER/.github/git/refs/tags/v1" -q .ref`
Expected: prints `refs/tags/v1`

---

### Task 7: Create `template-app` with thin caller workflows and project config

**Files:**
- Create (local clone): `~/dev-platform/repos/template-app/` (new repo)
- Create: `template-app/.github/workflows/ci.yml` (caller)
- Create: `template-app/.github/workflows/codeql.yml` (caller)
- Create: `template-app/.github/dependabot.yml`
- Create: `template-app/package.json`, `tsconfig.json`, `eslint.config.js`, `.prettierrc.json`, `vitest.config.ts`, `.gitignore`, `README.md`, `justfile`
- Create: `template-app/src/index.ts`, `template-app/src/index.test.ts`

**Interfaces:**
- Consumes: reusable workflows `@v1` from Task 6.
- Produces: a GitHub **template repository** `$OWNER/template-app` whose first PR exercises the full pipeline. `package.json` must define scripts: `lint`, `typecheck`, `test:unit`, `test:integration`, `build`, `test`.

- [ ] **Step 1: Create the repo and clone**

```bash
OWNER=$(gh api user -q .login)
gh repo create "$OWNER/template-app" --private \
  --description "Template for new projects (CI, tests, review, board wired in)" --clone
mv "$OWNER/template-app" ~/dev-platform/repos/template-app 2>/dev/null || true
cd ~/dev-platform/repos/template-app
```

- [ ] **Step 2: Write the caller workflows**

```yaml
# .github/workflows/ci.yml  (replace OWNER at bootstrap time; template ships a placeholder)
name: CI
on:
  pull_request:
  push:
    branches: [main]
jobs:
  ci:
    uses: __OWNER__/.github/.github/workflows/ci.yml@v1
    with:
      language: node
      node-version: "20"
      run-integration: true
```

```yaml
# .github/workflows/codeql.yml
name: CodeQL
on:
  pull_request:
  push:
    branches: [main]
  schedule:
    - cron: "0 6 * * 1"
jobs:
  codeql:
    uses: __OWNER__/.github/.github/workflows/codeql.yml@v1
    with:
      languages: javascript-typescript
```

- [ ] **Step 3: Copy the Dependabot config**

```bash
cp ~/dev-platform/repos/.github/workflow-templates/dependabot.yml .github/dependabot.yml
```

- [ ] **Step 4: Write project config and a sample tested module**

```json
// package.json
{
  "name": "template-app",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "lint": "eslint .",
    "typecheck": "tsc --noEmit",
    "test:unit": "vitest run --exclude '**/*.integration.test.ts'",
    "test:integration": "vitest run **/*.integration.test.ts || echo 'no integration tests'",
    "test": "vitest run",
    "build": "tsc -p tsconfig.json"
  },
  "devDependencies": {
    "@types/node": "^20",
    "eslint": "^9",
    "prettier": "^3",
    "typescript": "^5",
    "vitest": "^2"
  }
}
```

```json
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noEmit": false,
    "outDir": "dist",
    "skipLibCheck": true
  },
  "include": ["src"]
}
```

```js
// eslint.config.js
export default [
  { files: ["**/*.ts"], rules: {} }
];
```

```json
// .prettierrc.json
{ "singleQuote": false, "semi": true }
```

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
export default defineConfig({ test: { environment: "node" } });
```

```ts
// src/index.ts
export function greet(name: string): string {
  return `Hello, ${name}!`;
}
```

```ts
// src/index.test.ts
import { expect, test } from "vitest";
import { greet } from "./index.js";

test("greet returns a greeting", () => {
  expect(greet("world")).toBe("Hello, world!");
});
```

```
# .gitignore
node_modules/
dist/
coverage/
*.log
.env
```

```
# justfile
install:
    npm ci
test:
    npm test
lint:
    npm run lint
build:
    npm run build
```

```markdown
<!-- README.md -->
# template-app

Project scaffold. After creating a repo from this template, run:

    ./scripts/bootstrap.sh

which wires CI, branch protection, labels, and the shared board.
```

- [ ] **Step 5: Verify locally**

Run: `cd ~/dev-platform/repos/template-app && npm install && npm run typecheck && npm run test:unit && actionlint`
Expected: install succeeds; typecheck exits `0`; the `greet` test passes; actionlint reports no errors (the `__OWNER__` placeholder is a valid string to actionlint).

- [ ] **Step 6: Commit and push**

```bash
git add -A
git commit -m "feat: template-app scaffold with caller workflows and sample tested module"
git branch -M main
git push -u origin main
```

---

### Task 8: Repository ruleset JSON for `main` protection

**Files:**
- Create: `template-app/.platform/ruleset.json`

**Interfaces:**
- Consumes: nothing.
- Produces: `.platform/ruleset.json` applied by `bootstrap.sh` (Task 9) via `gh api repos/$OWNER/$REPO/rulesets`. Required checks named must match job names from the reusable CI (`lint`, `typecheck`, `unit`, `build`) plus `analyze` (CodeQL).

- [ ] **Step 1: Write the ruleset**

```json
{
  "name": "protect-main",
  "target": "branch",
  "enforcement": "active",
  "conditions": {
    "ref_name": { "include": ["refs/heads/main"], "exclude": [] }
  },
  "rules": [
    { "type": "deletion" },
    { "type": "non_fast_forward" },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true,
        "required_review_thread_resolution": true
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "strict_required_status_checks_policy": true,
        "required_status_checks": [
          { "context": "lint" },
          { "context": "typecheck" },
          { "context": "unit" },
          { "context": "build" }
        ]
      }
    }
  ]
}
```

> Note: check contexts for reusable-workflow jobs appear as `ci / lint`, etc., depending on the caller job name. After the smoke test (Task 12) surfaces the exact context strings via `gh pr checks`, update this file to match and re-apply. Start with the short names; correct in Task 12 if needed.

- [ ] **Step 2: Verify JSON is valid**

Run: `cd ~/dev-platform/repos/template-app && python3 -c "import json; json.load(open('.platform/ruleset.json')); print('valid')"`
Expected: prints `valid`

- [ ] **Step 3: Commit and push**

```bash
git add .platform/ruleset.json
git commit -m "feat: add main-branch protection ruleset"
git push
```

---

### Task 9: `bootstrap.sh` — one-shot project setup

**Files:**
- Create: `template-app/scripts/bootstrap.sh` (executable)

**Interfaces:**
- Consumes: `labels.json` (fetched from `$OWNER/.github`), `.platform/ruleset.json` (Task 8), caller workflows with `__OWNER__` placeholder (Task 7).
- Produces: a fully configured repo — owner substituted, labels synced, ruleset applied, Dependabot present. Board linkage is Task 11.

- [ ] **Step 1: Write the script**

```bash
#!/usr/bin/env bash
set -euo pipefail

OWNER=$(gh api user -q .login)
REPO=$(basename "$(git rev-parse --show-toplevel)")
echo "Bootstrapping $OWNER/$REPO"

# 1. Substitute the __OWNER__ placeholder in caller workflows
grep -rl '__OWNER__' .github/workflows | while read -r f; do
  sed -i.bak "s/__OWNER__/$OWNER/g" "$f" && rm -f "$f.bak"
done

# 2. Sync labels from the platform repo
tmp=$(mktemp)
gh api "repos/$OWNER/.github/contents/labels.json" -q .content | base64 -d > "$tmp"
python3 - "$tmp" "$OWNER" "$REPO" <<'PY'
import json, subprocess, sys
labels, owner, repo = json.load(open(sys.argv[1])), sys.argv[2], sys.argv[3]
for l in labels:
    subprocess.run(["gh","label","create",l["name"],"--color",l["color"],
                    "--description",l["description"],"--force",
                    "--repo",f"{owner}/{repo}"], check=True)
PY
rm -f "$tmp"

# 3. Apply the branch-protection ruleset
gh api -X POST "repos/$OWNER/$REPO/rulesets" \
  --input .platform/ruleset.json >/dev/null
echo "Ruleset applied."

# 4. Commit the owner substitution
git add .github/workflows
git commit -m "chore: bootstrap project ($OWNER)" || echo "nothing to commit"
echo "Done. Push and open your first PR."
```

- [ ] **Step 2: Make it executable and syntax-check**

```bash
cd ~/dev-platform/repos/template-app
chmod +x scripts/bootstrap.sh
bash -n scripts/bootstrap.sh && echo "syntax ok"
```
Expected: prints `syntax ok`

- [ ] **Step 3: Commit and push**

```bash
git add scripts/bootstrap.sh
git commit -m "feat: add bootstrap.sh for one-shot project setup"
git push
```

- [ ] **Step 4: Mark the repo as a template**

```bash
gh repo edit "$OWNER/template-app" --template
```
Verify: `gh repo view "$OWNER/template-app" --json isTemplate -q .isTemplate`
Expected: prints `true`

---

### Task 10: Apply protection to `template-app` itself and verify enforcement

**Files:** none (applies existing ruleset to `template-app`).

**Interfaces:**
- Consumes: `.platform/ruleset.json`.
- Produces: proof that the ruleset applies and blocks direct pushes.

- [ ] **Step 1: Apply the ruleset to template-app**

```bash
OWNER=$(gh api user -q .login)
cd ~/dev-platform/repos/template-app
gh api -X POST "repos/$OWNER/template-app/rulesets" --input .platform/ruleset.json
```

- [ ] **Step 2: Verify the ruleset is active**

Run: `gh api "repos/$OWNER/template-app/rulesets" -q '.[].name'`
Expected: includes `protect-main`

- [ ] **Step 3: Verify a direct push to main is rejected**

```bash
git commit --allow-empty -m "test: should be rejected on main"
git push origin main || echo "REJECTED_AS_EXPECTED"
git reset --hard HEAD~1
```
Expected: push fails and prints `REJECTED_AS_EXPECTED` (protection blocks direct pushes). If it succeeds, note the private-repo/free-plan caveat and record whether enforcement is available.

---

### Task 11: Create the shared GitHub Project board

**Files:**
- Create: `~/dev-platform/repos/.github/scripts/link-project.sh`

**Interfaces:**
- Produces: a Projects v2 board with fields Status, Priority, Size, Iteration; and a reusable `link-project.sh` that links a repo to the board. Records the project number in `~/dev-platform/repos/.github/PROJECT.md`.

- [ ] **Step 1: Create the project and capture its number**

```bash
OWNER=$(gh api user -q .login)
PNUM=$(gh project create --owner "$OWNER" --title "Platform Board" --format json -q .number)
echo "Project number: $PNUM"
printf '# Platform Board\n\nProject number: %s\nOwner: %s\n' "$PNUM" "$OWNER" \
  > ~/dev-platform/repos/.github/PROJECT.md
```

- [ ] **Step 2: Add the Priority, Size, and Iteration fields**

```bash
gh project field-create "$PNUM" --owner "$OWNER" --name "Priority" \
  --data-type SINGLE_SELECT --single-select-options "High,Medium,Low"
gh project field-create "$PNUM" --owner "$OWNER" --name "Size" \
  --data-type SINGLE_SELECT --single-select-options "XS,S,M,L,XL"
gh project field-create "$PNUM" --owner "$OWNER" --name "Iteration" \
  --data-type ITERATION
```
> Note: the built-in `Status` field already exists with Todo/In Progress/Done. Rename/extend columns in the UI to `Backlog → Ready → In progress → In review → Done` (the CLI cannot fully script status options today). Workflow automations (auto-add, item-closed → Done) are enabled via the project's **Workflows** UI tab — a one-time manual toggle documented in PROJECT.md.

- [ ] **Step 3: Write link-project.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail
OWNER=$(gh api user -q .login)
REPO=$(basename "$(git rev-parse --show-toplevel)")
PNUM=$(grep 'Project number' ~/dev-platform/repos/.github/PROJECT.md | grep -o '[0-9]\+')
gh project link "$PNUM" --owner "$OWNER" --repo "$OWNER/$REPO"
echo "Linked $OWNER/$REPO to project $PNUM"
```

- [ ] **Step 4: Verify the project and fields exist**

Run: `gh project field-list "$PNUM" --owner "$OWNER" -q '.fields[].name' 2>/dev/null || gh project field-list "$PNUM" --owner "$OWNER"`
Expected: lists Title, Status, Priority, Size, Iteration.

- [ ] **Step 5: Commit link-project.sh and PROJECT.md**

```bash
cd ~/dev-platform/repos/.github
chmod +x scripts/link-project.sh
git add scripts/link-project.sh PROJECT.md
git commit -m "feat: create shared project board and link script"
git push
```

---

### Task 12: End-to-end smoke test (prove the 10-minute target)

**Files:** none (creates and then optionally deletes a throwaway repo `smoke-test`).

**Interfaces:**
- Consumes: everything — template, bootstrap, ruleset, reusable workflows, board.
- Produces: evidence that a new project gets green CI + preview + review gate + board linkage on its first PR.

- [ ] **Step 1: Start a timer and create the project from the template**

```bash
OWNER=$(gh api user -q .login)
date +%s > /tmp/smoke_start
gh repo create "$OWNER/smoke-test" --template "$OWNER/template-app" --private --clone
cd smoke-test
```

- [ ] **Step 2: Bootstrap and link the board**

```bash
./scripts/bootstrap.sh
git push
~/dev-platform/repos/.github/scripts/link-project.sh
```

- [ ] **Step 3: Open a feature branch PR that changes tested code**

```bash
git checkout -b feat/smoke
sed -i.bak 's/Hello, /Hi, /' src/index.ts && rm -f src/index.ts.bak
sed -i.bak 's/Hello, /Hi, /' src/index.test.ts && rm -f src/index.test.ts.bak
git commit -am "feat: change greeting"
git push -u origin feat/smoke
gh pr create --fill
```

- [ ] **Step 4: Verify CI checks run and gate the PR**

Run: `gh pr checks feat/smoke --watch`
Expected: `lint`, `typecheck`, `unit`, `build`, and CodeQL (`analyze`) checks appear and pass. Record the **exact context names** printed. If they differ from `.platform/ruleset.json` (e.g. `ci / lint`), update `template-app/.platform/ruleset.json` and re-apply to `template-app` and `smoke-test`, then re-run this step.

- [ ] **Step 5: Verify the merge is gated on review**

Run: `gh pr merge feat/smoke --squash --auto || echo "BLOCKED_PENDING_REVIEW_AS_EXPECTED"`
Expected: merge does not complete immediately because 1 review is required — prints the blocked message (or queues as auto-merge pending review). This proves the review gate.

- [ ] **Step 6: Measure elapsed time and record results**

```bash
echo "Elapsed: $(( $(date +%s) - $(cat /tmp/smoke_start) )) seconds"
```
Expected: setup-to-first-green-PR well under 600s (the 10-minute target). Note the number.

- [ ] **Step 7: Clean up the smoke repo (optional — MANUAL, do not run in this session)**

The current token lacks `delete_repo` scope, so deletion is left to the human to do later:

```bash
# Run manually when ready:
#   gh auth refresh -s delete_repo
#   gh repo delete "$OWNER/smoke-test" --yes
cd .. && rm -rf smoke-test   # remove local clone only
```

- [ ] **Step 8: Commit any ruleset corrections discovered**

```bash
cd ~/dev-platform/repos/template-app
git add .platform/ruleset.json 2>/dev/null || true
git commit -m "fix: correct required-check context names from smoke test" || echo "no changes"
git push || true
```

---

## Deferred (out of scope — "Full" tier)

- Conventional Commits + automated changelog (release-please / changesets)
- Staging → production manual promotion gate
- Migration from personal account to a GitHub Organization (unlocks org rulesets + reusable-workflow sharing at org scope)
- Vercel preview-deploy wiring is intentionally left to per-project setup via the Vercel GitHub integration (the swappable seam); add a reusable `deploy.yml` when the first deployable app exists.
