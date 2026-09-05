# GitHub Actions - Comprehensive Guide

> **A hands-on reference** built around the real workflows in this repository.
> Every concept links back to the `.yml` file that demonstrates it.

---

## Table of Contents

1. [What is GitHub Actions?](#what-is-github-actions)
2. [Workflow Triggers (Events)](#workflow-triggers-events)
3. [Core Concepts](#core-concepts)
   - [Job Execution Model](#1-job-execution-model)
   - [Job Artifacts](#2-job-artifacts)
   - [Job Outputs](#3-job-outputs)
   - [GitHub Context](#4-github-context)
   - [Actions (Marketplace & Built-in)](#5-actions-marketplace--built-in)
   - [Path Filters](#6-path-filters)
   - [Caching Dependencies](#7-caching-dependencies)
   - [Conditional Execution (`if`)](#8-conditional-execution-if)
   - [Matrix Strategy](#9-matrix-strategy)
   - [Reusable Workflows (`workflow_call`)](#10-reusable-workflows-workflow_call)
   - [Custom Composite Actions](#11-custom-composite-actions)
   - [GitHub Actions Expressions](#12-github-actions-expressions)
   - [Environment Variables & Secrets](#13-environment-variables--secrets)
   - [Permissions & Security Hardening](#14-permissions--security-hardening)
4. [Practical Workflows (from this project)](#practical-workflows-from-this-project)
   - [Testing Pipeline](#workflow-1-testing-pipeline)
   - [Job Artifact Pipeline](#workflow-2-job-artifact-pipeline)
   - [Custom Actions Pipeline](#workflow-3-custom-actions-pipeline)
   - [Matrix Pipeline](#workflow-4-matrix-pipeline)
   - [Reusable Deploy Workflow](#workflow-5-reusable-deploy-workflow)
   - [Activity Events Pipeline](#workflow-6-activity-events-pipeline)
   - [GitHub Info Pipeline](#workflow-7-github-info-pipeline)
5. [Workflow Visualisation](#workflow-visualisation)
6. [Important Notes & Best Practices](#important-notes--best-practices)
7. [Important Resources](#important-resources)

---

## What is GitHub Actions?

GitHub Actions is a **CI/CD** (Continuous Integration / Continuous Deployment) platform that lets you automate tasks triggered by events in your repository. Workflows are defined in **YAML** files stored in the `.github/workflows/` directory.

### Key Components

| Component    | Description |
|-------------|-------------|
| **Workflow** | An automated process defined in a YAML file |
| **Event**    | A specific activity that triggers a workflow (e.g., `push`, `pull_request`) |
| **Job**      | A set of steps that execute on the same runner |
| **Step**     | A single task — an action or a shell command — within a job |
| **Action**   | A reusable unit of code (custom or from the Marketplace) |
| **Runner**   | The virtual machine that executes your workflow (GitHub-hosted or self-hosted) |

### How It Works (High Level)

```
Event (push / PR / manual)
  └─▶ Workflow (.github/workflows/*.yml)
        ├─▶ Job 1 (Runner A)
        │     ├─ Step 1: Checkout code
        │     ├─ Step 2: Install dependencies
        │     └─ Step 3: Run tests
        └─▶ Job 2 (Runner B)  ← can depend on Job 1
              ├─ Step 1: Download artifact
              └─ Step 2: Deploy
```

---

## Workflow Triggers (Events)

Workflows are triggered by specific events. Below is a comprehensive reference:

### Commit & Branch Events
| Event | Description |
|-------|-------------|
| **`push`** | Triggered when a commit or tag is pushed to the repository |
| **`create`** | Triggered when a branch or tag is created |
| **`delete`** | Triggered when a branch or tag is deleted |

### Pull Request Events
| Event | Description |
|-------|-------------|
| **`pull_request`** | PR is opened, closed, reopened, edited, synchronized, assigned, labeled, etc. |
| **`pull_request_review`** | A PR review is submitted, edited, or dismissed |
| **`pull_request_review_comment`** | A comment on a PR diff is created, edited, or deleted |
| **`pull_request_target`** | Like `pull_request` but runs in the base repo context (has access to secrets — use carefully!) |

### Issue Events
| Event | Description |
|-------|-------------|
| **`issues`** | An issue is opened, edited, deleted, closed, reopened, assigned, labeled, pinned, etc. |
| **`issue_comment`** | A comment on an issue or PR is created, edited, or deleted |

### Release & Deployment Events
| Event | Description |
|-------|-------------|
| **`release`** | A release is published, created, edited, deleted, or pre-released |
| **`page_build`** | A GitHub Pages build is attempted |
| **`deployment`** | A deployment is created (via API) |
| **`deployment_status`** | A deployment status is updated |

### Repository Events
| Event | Description |
|-------|-------------|
| **`fork`** | Someone forks the repository |
| **`watch`** / **`star`** | Someone stars or unstars the repository |
| **`member`** | A collaborator is added, removed, or permissions changed |
| **`public`** | A private repository is made public |

### Discussion Events
| Event | Description |
|-------|-------------|
| **`discussion`** | A GitHub Discussion is created, edited, answered, deleted, pinned/unpinned |
| **`discussion_comment`** | A comment on a Discussion is created, edited, or deleted |

### Code & Wiki Events
| Event | Description |
|-------|-------------|
| **`commit_comment`** | A comment is made directly on a commit |
| **`gollum`** | A wiki page is created or updated |

### Project Board Events
| Event | Description |
|-------|-------------|
| **`project`** | A classic Project board is created, updated, closed, reopened, or deleted |
| **`project_card`** | A card in a Project board is created, moved, updated, or deleted |
| **`project_column`** | A column in a Project board is created, updated, moved, or deleted |

### Scheduled & Manual Events
| Event | Description |
|-------|-------------|
| **`schedule`** | Runs the workflow on a POSIX cron schedule (e.g., `cron: '0 0 * * *'`) |
| **`workflow_dispatch`** | Allows manual trigger from the Actions tab in your repository |
| **`workflow_call`** | Allows this workflow to be called by another workflow (reusable workflow) |

### Integration & External Events
| Event | Description |
|-------|-------------|
| **`repository_dispatch`** | Triggered by a custom external webhook event via the GitHub API |
| **`status`** | Triggered when the status of a Git commit changes (e.g., external CI reporting) |
| **`workflow_run`** | Triggered when another workflow completes, succeeds, or fails |

---

## Core Concepts

### 1. Job Execution Model

**Parallel Execution (Default)**
- All jobs in a workflow run in parallel by default
- Each job runs on its own isolated runner (virtual machine)
- Jobs do **not** share file systems — use artifacts to transfer files

**Sequential Execution**
- Use the `needs:` key to make jobs run sequentially
- A job will only run if **all** of its dependency jobs succeed

```yaml
jobs:
  test-job:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  build-job:
    needs: test-job  # Runs only after test-job succeeds
    runs-on: ubuntu-latest
    steps:
      - run: npm run build

  deploy-job:
    needs: build-job  # Runs only after build-job succeeds
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying..."
```

> 📁 **See it in action:** [`jobArtifact-pipeline.yml`](.github/workflows/jobArtifact-pipeline.yml-old) — three-stage pipeline with `test → build → deploy`

---

### 2. Job Artifacts

**What are Job Artifacts?**
- Files/assets generated during a job (e.g., compiled code, `dist` folder, test reports)
- Temporary data that can be shared between jobs within the same workflow run
- Artifacts are also downloadable from the Actions UI after a run completes

**Upload Artifacts:**
```yaml
- name: Upload Build Artifact
  uses: actions/upload-artifact@v4
  with:
    name: build-artifact
    path: dist
    # You can also upload multiple paths:
    # path: |
    #   dist
    #   package.json
```

**Download Artifacts:**
```yaml
- name: Get build artifact
  uses: actions/download-artifact@v4
  with:
    name: build-artifact  # Must match the upload name exactly
```

**Conditional Artifact Upload (on failure):**
```yaml
- name: Upload Test Report on Failure
  if: failure() && steps.test-step.outcome == 'failure'
  uses: actions/upload-artifact@v4
  with:
    name: test-report-artifact
    path: test-report.json
```

> 📁 **See it in action:** [`jobArtifact-pipeline.yml`](.github/workflows/jobArtifact-pipeline.yml-old) and [`customActions-pipeline.yml`](.github/workflows/customActions-pipeline.yml)

---

### 3. Job Outputs

Pass data from one job to another using **outputs**:

**Step 1 — Write the output inside the producing job:**
```yaml
build-job:
  outputs:
    script-file: ${{ steps.unique-id.outputs.script-file }}
  steps:
    - id: unique-id
      run: echo "script-file=myfile.js" >> $GITHUB_OUTPUT
```

**Step 2 — Read the output in the consuming job:**
```yaml
deploy-job:
  needs: build-job
  steps:
    - run: echo "File is ${{ needs.build-job.outputs.script-file }}"
```

> ⚠️ **Note:** `::set-output` is deprecated. Always use `>> $GITHUB_OUTPUT` instead.

> 📁 **See it in action:** [`jobArtifact-pipeline.yml`](.github/workflows/jobArtifact-pipeline.yml-old) — `build-job` publishes the JS filename and `deploy-job` reads it

---

### 4. GitHub Context

The `github` context provides metadata about the workflow run and the event that triggered it:

| Expression | Description |
|-----------|-------------|
| `${{ github }}` | Complete GitHub context |
| `${{ github.event }}` | Event payload details |
| `${{ github.event_name }}` | Name of the event that triggered the workflow |
| `${{ github.actor }}` | Username who triggered the workflow |
| `${{ github.ref }}` | Git reference (branch/tag) |
| `${{ github.sha }}` | Commit SHA that triggered the workflow |
| `${{ github.repository }}` | Repository name (`owner/repo`) |
| `${{ github.run_id }}` | Unique ID for the workflow run |
| `${{ github.run_number }}` | Incrementing number for each run of this workflow |
| `${{ github.workspace }}` | Default working directory on the runner |

Use `${{ toJson(github) }}` or `${{ toJson(github.event) }}` to dump the full context as JSON for debugging.

> 📁 **See it in action:** [`gitHubInfo-pipeline.yml`](.github/workflows/gitHubInfo-pipeline.yml-old) and [`activity-pipeline.yml`](.github/workflows/activity-pipeline.yml-old)

---

### 5. Actions (Marketplace & Built-in)

Actions are reusable units of code that perform common tasks. They come from three sources:

| Source | Example |
|--------|---------|
| **Official (GitHub)** | `actions/checkout@v3`, `actions/setup-node@v7` |
| **Community (Marketplace)** | `aws-actions/configure-aws-credentials@v4` |
| **Custom (your repo)** | `./.github/custom-actions` |

**Commonly Used Actions:**

| Action | Purpose |
|--------|---------|
| `actions/checkout@v3` | Checks out your repository code to the runner |
| `actions/setup-node@v7` | Sets up a specific Node.js version |
| `actions/upload-artifact@v4` | Uploads files as artifacts |
| `actions/download-artifact@v4` | Downloads artifacts from previous jobs |
| `actions/cache@v3` | Caches dependencies to speed up builds |

---

### 6. Path Filters

Control when workflows run based on file paths that changed:

**`paths`** — Only run if specific paths are modified:
```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
```

**`paths-ignore`** — Run for all changes EXCEPT specified paths:
```yaml
on:
  push:
    paths-ignore:
      - '**/*.md'      # Ignores all markdown files in any directory
      - 'docs/**'      # Ignores entire docs folder
      - '.gitignore'   # Ignores specific files
```

**Common Glob Patterns:**

| Pattern | Matches |
|---------|---------|
| `**/*.md` | All markdown files in any directory |
| `*.md` | Only markdown files in root directory |
| `src/**` | All files in src folder (recursive) |
| `src/**/*.js` | All JS files in src and subdirectories |
| `!src/generated/**` | Exclude generated files |

> ⚠️ **Important:** Use `**/*.md` NOT `**.md` — the latter is invalid syntax and will be silently ignored.

> 📁 **See it in action:** [`customActions-pipeline.yml`](.github/workflows/customActions-pipeline.yml) — ignores markdown file changes

---

### 7. Caching Dependencies

Improve workflow performance by caching dependencies between runs:

**Using `actions/cache` directly:**
```yaml
- name: Cache node dependencies
  id: cache
  uses: actions/cache@v3
  with:
    path: node_modules     # What to cache
    key: node-dependencies-${{ hashFiles('**/package-lock.json') }}
    
- name: Install dependencies
  if: steps.cache.outputs.cache-hit != 'true'  # Skip install if cache hit
  run: npm ci
```

**How it works:**
1. On the first run, dependencies are installed and the `node_modules` folder is cached
2. On subsequent runs, if `package-lock.json` hasn't changed, the cache is restored — skipping `npm ci`
3. If `package-lock.json` changes (new hash), the cache misses and dependencies are reinstalled

**Cache Key Strategy:**
| Key Pattern | When It Invalidates |
|------------|-------------------|
| `node-deps-${{ hashFiles('**/package-lock.json') }}` | When any `package-lock.json` changes |
| `node-deps-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}` | Per-OS cache + lock file changes |
| `node-deps-${{ github.ref }}` | Per-branch (rarely useful for deps) |

> 📁 **See it in action:** [`jobArtifact-pipeline.yml`](.github/workflows/jobArtifact-pipeline.yml-old) and [`custom-actions/action.yml`](.github/custom-actions/action.yml)

---

### 8. Conditional Execution (`if`)

Control whether steps or jobs run using the `if:` keyword:

**Status Check Functions:**
| Function | Description |
|----------|-------------|
| `success()` | Returns `true` when all previous steps succeeded (default behavior) |
| `failure()` | Returns `true` when any previous step failed |
| `always()` | Always returns `true` — the step runs regardless of outcome |
| `cancelled()` | Returns `true` if the workflow was cancelled |

**Step Outcome:**
```yaml
- name: Test Code
  id: test-step
  run: npm run test

- name: Upload Report on Failure
  if: failure() && steps.test-step.outcome == 'failure'
  uses: actions/upload-artifact@v4
  with:
    name: test-report
    path: test-report.json
```

**Conditional on Branch:**
```yaml
- name: Deploy to Production
  if: github.ref == 'refs/heads/main'
  run: echo "Deploying to production..."
```

**Conditional on Event Type:**
```yaml
- name: Comment on PR
  if: github.event_name == 'pull_request'
  run: echo "This is a PR event"
```

> 📁 **See it in action:** [`customActions-pipeline.yml`](.github/workflows/customActions-pipeline.yml) — uploads test report only on test failure

---

### 9. Matrix Strategy

Run the same job across **multiple configurations** (OS, runtime versions, etc.):

```yaml
jobs:
  build-job:
    continue-on-error: true  # Don't fail the entire workflow if one matrix combo fails
    strategy:
      matrix:
        node-version: [16.x, 18.x, 24.x]
        operating-system: [ubuntu-latest, windows-latest]
        include:                              # Add extra specific combos
          - node-version: 16.x
            operating-system: windows-latest
        exclude:                              # Remove specific combos
          - node-version: 24.x
            operating-system: windows-latest
    runs-on: ${{ matrix.operating-system }}
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm run build
```

**Key Matrix Features:**

| Feature | Description |
|---------|-------------|
| `matrix:` | Defines the combinations to test |
| `include:` | Adds extra specific combinations beyond the cross-product |
| `exclude:` | Removes specific combinations from the cross-product |
| `continue-on-error:` | Allows the workflow to continue even if some matrix jobs fail |
| `fail-fast:` (default `true`) | If `true`, cancels all in-progress matrix jobs when one fails; set to `false` to run all |
| `max-parallel:` | Limits concurrent matrix jobs (useful for rate-limited resources) |

**How the Matrix Expands:**

The example above creates this matrix (6 combos - 1 exclusion = 5 jobs):

| Node | OS | Status |
|------|----|--------|
| 16.x | ubuntu-latest | ✅ Runs |
| 16.x | windows-latest | ✅ Runs (also explicitly included) |
| 18.x | ubuntu-latest | ✅ Runs |
| 18.x | windows-latest | ✅ Runs |
| 24.x | ubuntu-latest | ✅ Runs |
| 24.x | windows-latest | ❌ Excluded |

> 📁 **See it in action:** [`matirx-pipeline.yml`](.github/workflows/matirx-pipeline.yml-old)

---

### 10. Reusable Workflows (`workflow_call`)

Create workflows that can be **called by other workflows**, reducing duplication:

**Defining a Reusable Workflow:**
```yaml
# reusable-pipeline.yml
name: reusable deploy workflow
on:
  workflow_call:          # This is the key — makes it callable
    inputs:
      artifact-name:
        description: 'The name of the artifact to be deployed'
        required: false
        default: 'dist'
        type: string
    # You can also define outputs and secrets:
    # outputs:
    #   deploy-url:
    #     description: 'The deployment URL'
    #     value: ${{ jobs.deploy-job.outputs.url }}
    # secrets:
    #   deploy-token:
    #     required: true

jobs:
  deploy-job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: ${{ inputs.artifact-name }}
      - run: ls
      - run: echo "Deploying artifact ${{ inputs.artifact-name }}..."
```

**Calling a Reusable Workflow:**
```yaml
# testing-pipeline.yml
jobs:
  test-job:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy-job:
    needs: test-job
    uses: ./.github/workflows/reusable-pipeline.yml  # Relative path to the reusable workflow
    with:
      artifact-name: dist
    # secrets:
    #   deploy-token: ${{ secrets.DEPLOY_TOKEN }}
```

**Reusable Workflow vs Custom Action:**

| Feature | Reusable Workflow | Custom Action |
|---------|------------------|---------------|
| Defined with | `workflow_call` trigger | `action.yml` |
| Scope | Full jobs with multiple steps | A single step |
| Location | `.github/workflows/` | Anywhere (e.g., `.github/custom-actions/`) |
| Sharing | Cross-repo via `org/repo/.github/workflows/file.yml@ref` | Via Marketplace or local path |
| Access to secrets | Via `secrets:` input | Inherits from workflow |

> 📁 **See it in action:** [`reusable-pipeline.yml`](.github/workflows/reusable-pipeline.yml-old) called by [`testing-pipeline.yml`](.github/workflows/testing-pipeline.yml-old)

---

### 11. Custom Composite Actions

Create **your own reusable actions** with multiple steps bundled together:

**Defining a Composite Action** (`.github/custom-actions/action.yml`):
```yaml
name: Cache & install dependencies
description: This action caches and installs dependencies.
inputs:
  should-cache:
    description: 'Whether to cache dependencies or not'
    required: false
    default: 'true'
outputs:
  used-cache:
    description: 'Whether the cache was used or not'
    value: ${{ steps.install.outputs.used-cache }}
runs:
  using: "composite"    # MANDATORY for composite actions
  steps:
    - name: Cache node dependencies
      if: ${{ inputs.should-cache == 'true' }}
      id: cache
      uses: actions/cache@v3
      with:
        path: node_modules
        key: node-dependencies-${{ hashFiles('**/package-lock.json') }}
    - name: Install dependencies
      id: install
      if: steps.cache.outputs.cache-hit != 'true' || ${{ inputs.should-cache != 'true' }}
      run: |
        npm ci
        echo "used-cache=${{ steps.cache.outputs.cache-hit }}" >> $GITHUB_OUTPUT
      shell: bash    # MANDATORY for composite action run steps
```

**Using a Custom Action:**
```yaml
steps:
  - name: Get The Code
    uses: actions/checkout@v3   # Must checkout first!
  - name: Cache & Install Dependencies
    id: cache-install
    uses: ./.github/custom-actions  # Relative path to the action directory
    with:
      should-cache: 'true'
  - name: Output cache status
    run: 'echo "Cache status: ${{ steps.cache-install.outputs.used-cache }}"'
```

**Composite Action Rules:**
1. The action file **must** be named `action.yml` (or `action.yaml`)
2. `using: "composite"` is **mandatory** in the `runs:` section
3. Every `run:` step **must** specify `shell:` (e.g., `shell: bash`)
4. You **must** `actions/checkout` before using a local action (it needs the files on disk)
5. Inputs are accessed via `${{ inputs.<name> }}`
6. Outputs are set using `>> $GITHUB_OUTPUT`

> 📁 **See it in action:** [`custom-actions/action.yml`](.github/custom-actions/action.yml) used by [`customActions-pipeline.yml`](.github/workflows/customActions-pipeline.yml)

---

### 12. GitHub Actions Expressions

Dynamically evaluate values in your workflows using `${{ expression }}` syntax:

**Operators:**
| Operator | Example |
|----------|---------|
| `==` | `github.ref == 'refs/heads/main'` |
| `!=` | `github.event_name != 'pull_request'` |
| `&&` | `failure() && steps.test.outcome == 'failure'` |
| `\|\|` | `github.ref == 'refs/heads/main' \|\| github.ref == 'refs/heads/develop'` |
| `!` | `!contains(github.event.head_commit.message, 'skip-ci')` |

**Built-in Functions:**

| Function | Description | Example |
|----------|-------------|---------|
| `toJson(value)` | Converts a value to JSON | `echo "${{ toJson(github) }}"` |
| `fromJson(value)` | Parses a JSON string | `fromJson(needs.setup.outputs.matrix)` |
| `contains(search, item)` | Checks if a string/array contains a value | `contains(github.event.labels.*.name, 'bug')` |
| `startsWith(str, prefix)` | Checks string prefix | `startsWith(github.ref, 'refs/tags/')` |
| `endsWith(str, suffix)` | Checks string suffix | `endsWith(github.repository, '-demo')` |
| `format(str, ...)` | String formatting | `format('Hello {0}!', github.actor)` |
| `hashFiles(patterns)` | SHA-256 hash of files | `hashFiles('**/package-lock.json')` |

---

### 13. Environment Variables & Secrets

#### Environment Variables — Three Scopes

Environment variables can be defined at **three levels**, and child scopes **override** parent scopes:

```yaml
# 1️⃣  Workflow-level (global) — accessible by ALL jobs and steps
env:
  NODE_ENV: production

jobs:
  build:
    # 2️⃣  Job-level — accessible only within this job
    env:
      CI: true
      NODE_ENV: staging  # ⬅ OVERRIDES the workflow-level NODE_ENV!
    runs-on: ubuntu-latest
    steps:
      - name: Use env vars
        # 3️⃣  Step-level — accessible only within this step
        env:
          DATABASE_URL: postgres://localhost:5432/mydb
          NODE_ENV: development  # ⬅ OVERRIDES the job-level NODE_ENV!
        run: echo "Environment is $NODE_ENV"  # prints "development"
```

> 💡 **Override Rule:** Step-level `env` overrides Job-level, which overrides Workflow-level. This is possible and intentional in GitHub Actions.

**Default Environment Variables (available in every workflow):**

| Variable | Description |
|----------|-------------|
| `GITHUB_REPOSITORY` | The owner/repo name |
| `GITHUB_REF` | The branch or tag ref |
| `GITHUB_SHA` | The commit SHA |
| `GITHUB_ACTOR` | The user who triggered the workflow |
| `GITHUB_WORKSPACE` | The default working directory |
| `GITHUB_RUN_ID` | Unique identifier for the run |
| `GITHUB_RUN_NUMBER` | Incrementing run number |
| `GITHUB_OUTPUT` | File path for setting step outputs |
| `GITHUB_ENV` | File path for setting environment variables |
| `RUNNER_OS` | The OS of the runner (e.g., `Linux`, `Windows`, `macOS`) |

> 📖 [Full list of default environment variables](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-variables#default-environment-variables)

**Setting Environment Variables Dynamically:**
```yaml
- name: Set dynamic env var
  run: echo "BUILD_VERSION=1.2.3" >> $GITHUB_ENV

- name: Use the env var
  run: echo "Version is $BUILD_VERSION"
```

---

#### Secrets

**How to Add Secrets (via GitHub UI):**
1. Go to your repository on GitHub
2. Navigate to **Settings → Secrets and variables → Actions**
3. Click **New repository secret**
4. Enter the secret name (e.g., `API_KEY`) and its value
5. Click **Add secret**

**Reading Secrets in Your Workflow:**
```yaml
steps:
  - name: Deploy
    env:
      API_KEY: ${{ secrets.API_KEY }}          # Repository secret
      ORG_TOKEN: ${{ secrets.ORG_TOKEN }}       # Organization secret
    run: ./deploy.sh
```

> ⚠️ Secrets are **masked** in logs — GitHub automatically redacts them. Never `echo` secrets for debugging.

---

#### Environment-Specific Secrets

If you need the **same secret names** with **different values** for different environments (e.g., `staging` vs `production`):

**Step 1 — Create environments in GitHub UI:**
1. Go to **Settings → Environments**
2. Click **New environment** and create your environments (e.g., `staging`, `production`)
3. Add secrets to each environment with the same name but different values

**Step 2 — Use the `environment:` key in your workflow:**
```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging           # ⬅ Uses secrets from the "staging" environment
    steps:
      - name: Deploy to Staging
        run: echo "Deploying to staging..."
        env:
          API_KEY: ${{ secrets.API_KEY }}   # ⬅ Gets the STAGING value of API_KEY

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production        # ⬅ Uses secrets from the "production" environment
    steps:
      - name: Deploy to Production
        run: echo "Deploying to production..."
        env:
          API_KEY: ${{ secrets.API_KEY }}   # ⬅ Gets the PRODUCTION value of API_KEY
```

> 💡 **Tip:** Environments can also have **protection rules** — like requiring manual approval before deploying to production, or restricting to specific branches.

---

### 14. Permissions & Security Hardening

**Principle of Least Privilege:**
```yaml
# Restrict permissions at the workflow level
permissions:
  contents: read      # Read-only access to repo contents
  pull-requests: write # Write access to PRs (for comments)
  
jobs:
  deploy:
    # Override permissions at job level
    permissions:
      contents: read
      deployments: write
```

**Security Best Practices:**
1. **Pin action versions to full SHA** (not just tags):
   ```yaml
   # ✅ Good — pinned to SHA
   uses: actions/checkout@8ade135a41bc03ea155e62e844d188df1ea18608
   
   # ⚠️ Acceptable — pinned to major version
   uses: actions/checkout@v3
   
   # ❌ Bad — unpinned
   uses: actions/checkout@main
   ```

2. **Use `pull_request` not `pull_request_target`** for untrusted code
3. **Never use `${{ github.event.* }}` in `run:` directly** — it can be injected:
   ```yaml
   # ❌ Vulnerable to script injection
   - run: echo "${{ github.event.pull_request.title }}"
   
   # ✅ Safe — use an environment variable
   - env:
       PR_TITLE: ${{ github.event.pull_request.title }}
     run: echo "$PR_TITLE"
   ```

4. **Use `GITHUB_TOKEN` instead of personal access tokens** when possible
5. **Set `permissions:` at the top of every workflow** to avoid using the default (which is often too broad)

---

## Practical Workflows (from this project)

### Workflow 1: Testing Pipeline

📄 **File:** [`testing-pipeline.yml`](.github/workflows/testing-pipeline.yml-old)

```yaml
name: for testing pipeline
on: [push, workflow_dispatch]

jobs:
  test-job:
    runs-on: ubuntu-latest
    steps:
      - name: Get The Code
        uses: actions/checkout@v3
      - name: set nodejs version
        uses: actions/setup-node@v7
        with:
          node-version: 20
      - name: Install Dependencies
        run: npm install
      - name: Run Tests
        run: npm test

  deploy-job:
    needs: test-job  # Only runs after test-job succeeds
    uses: ./.github/workflows/reusable-pipeline.yml  # Calls a reusable workflow
    with:
      artifact-name: dist
```

**Concepts Demonstrated:**
- ✅ Multiple trigger events (`push` + `workflow_dispatch`)
- ✅ Sequential job execution with `needs:`
- ✅ Node.js setup with `actions/setup-node`
- ✅ Calling a reusable workflow with `uses:` and `with:`

---

### Workflow 2: Job Artifact Pipeline

📄 **File:** [`jobArtifact-pipeline.yml`](.github/workflows/jobArtifact-pipeline.yml-old)

```yaml
name: Job Artifact demo
on:
  push:
    branches:
      - main
    paths-ignore:
      - '**/*.md'

jobs:
  test-job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Cache node dependencies
        id: cache
        uses: actions/cache@v3
        with:
          path: node_modules
          key: node-dependencies-${{ hashFiles('**/package-lock.json') }}
      - name: Install dependencies
        if: steps.cache.outputs.cache-hit != 'true'
        run: npm ci
      - name: lint Code
        run: npm run lint
      - name: Test Code
        id: test-step
        run: npm run test -- --reporter=json --outputFile=test-report.json
      - name: Upload Test Artifact
        if: failure() && steps.test-step.outcome == 'failure'
        uses: actions/upload-artifact@v4
        with:
          name: test-report-artifact
          path: test-report.json

  build-job:
    needs: test-job
    runs-on: ubuntu-latest
    outputs:
      script-file: ${{ steps.unique-id.outputs.script-file }}
    steps:
      - uses: actions/checkout@v3
      - name: Cache & Install
        id: cache
        uses: actions/cache@v3
        with:
          path: node_modules
          key: node-dependencies-${{ hashFiles('**/package-lock.json') }}
      - name: Install dependencies
        if: steps.cache.outputs.cache-hit != 'true'
        run: npm ci
      - name: Build website
        run: npm run build
      - name: Publish JS files
        id: unique-id
        run: find dist/assets/*.js -type f -execdir echo '::set-output name=script-file::{}' ';'
      - name: Upload Build Artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: dist

  deploy-job:
    needs: build-job
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-artifact
      - run: ls
      - run: echo "JS file is ${{ needs.build-job.outputs.script-file }}"
      - run: echo "Deploying website..."
```

**Concepts Demonstrated:**
- ✅ Branch filtering (`branches: [main]`)
- ✅ Path filtering (`paths-ignore: ['**/*.md']`)
- ✅ Dependency caching with `actions/cache`
- ✅ Conditional caching with `if: steps.cache.outputs.cache-hit != 'true'`
- ✅ Uploading & downloading artifacts between jobs
- ✅ Job outputs for passing data
- ✅ Conditional artifact upload on failure (`if: failure()`)
- ✅ Three-stage pipeline: `test → build → deploy`

---

### Workflow 3: Custom Actions Pipeline

📄 **Files:** [`customActions-pipeline.yml`](.github/workflows/customActions-pipeline.yml) + [`custom-actions/action.yml`](.github/custom-actions/action.yml)

```yaml
name: Job Artifact demo
on:
  push:
    branches: [main]
    paths-ignore: ['**/*.md']

jobs:
  test-job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Cache & Install Dependencies
        id: cache-install
        uses: ./.github/custom-actions       # Local composite action
        with:
          should-cache: 'false'
      - name: Output cache status
        run: 'echo "Cache status: ${{ steps.cache-install.outputs.used-cache }}"'
      - run: npm run lint
      - name: Test Code
        id: test-step
        run: npm run test -- --reporter=json --outputFile=test-report.json
      - name: Upload Test Artifact
        if: failure() && steps.test-step.outcome == 'failure'
        uses: actions/upload-artifact@v4
        with:
          name: test-report-artifact
          path: test-report.json

  build-job:
    needs: test-job
    runs-on: ubuntu-latest
    outputs:
      script-file: ${{ steps.unique-id.outputs.script-file }}
    steps:
      - uses: actions/checkout@v3
      - name: Cache & Install Dependencies
        id: cache-install
        uses: ./.github/custom-actions
        with:
          should-cache: 'true'
      - name: Output cache status
        run: 'echo "Cache status: ${{ steps.cache-install.outputs.used-cache }}"'
      - run: npm run build
      - id: unique-id
        run: find dist/assets/*.js -type f -execdir echo '::set-output name=script-file::{}' ';'
      - uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: dist

  deploy-job:
    needs: build-job
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-artifact
      - run: ls
      - run: echo "JS file is ${{ needs.build-job.outputs.script-file }}"
      - run: echo "Deploying website..."
```

**Concepts Demonstrated:**
- ✅ Custom composite action with inputs and outputs
- ✅ Replacing duplicated cache+install steps with a reusable action
- ✅ Passing `should-cache` input to control caching behavior
- ✅ Reading action outputs (`steps.cache-install.outputs.used-cache`)
- ✅ DRY principle — same action used in both `test-job` and `build-job`

---

### Workflow 4: Matrix Pipeline

📄 **File:** [`matirx-pipeline.yml`](.github/workflows/matirx-pipeline.yml-old)

```yaml
name: matrix demo
on:
  push:
    branches: [main]

jobs:
  build-job:
    continue-on-error: true
    strategy:
      matrix:
        node-version: [16.x, 18.x, 24.x]
        operating-system: [ubuntu-latest, windows-latest]
        include:
          - node-version: 16.x
            operating-system: windows-latest
        exclude:
          - node-version: 24.x
            operating-system: windows-latest
    runs-on: ${{ matrix.operating-system }}
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
      - run: npm ci
      - run: npm run build
```

**Concepts Demonstrated:**
- ✅ Matrix strategy for cross-platform, multi-version testing
- ✅ `include:` to add specific combos
- ✅ `exclude:` to remove specific combos
- ✅ `continue-on-error: true` — workflow doesn't fail if a matrix job fails
- ✅ Dynamic runner and node version via `${{ matrix.* }}`

---

### Workflow 5: Reusable Deploy Workflow

📄 **File:** [`reusable-pipeline.yml`](.github/workflows/reusable-pipeline.yml-old)

```yaml
name: reusable deploy workflow
on:
  workflow_call:
    inputs:
      artifact-name:
        description: 'The name of the artifact to be deployed'
        required: false
        default: 'dist'
        type: string

jobs:
  deploy-job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: ${{ inputs.artifact-name }}
      - run: ls
      - run: echo "Deploying artifact ${{ inputs.artifact-name }}..."
```

**Concepts Demonstrated:**
- ✅ `workflow_call` trigger makes the workflow reusable
- ✅ Typed inputs with defaults
- ✅ Accessing caller inputs via `${{ inputs.<name> }}`

---

### Workflow 6: Activity Events Pipeline

📄 **File:** [`activity-pipeline.yml`](.github/workflows/activity-pipeline.yml-old)

```yaml
name: check github activities events
on:
  pull_request:
    types: [opened, edited]   # Only specific PR event types
  workflow_dispatch

jobs:
  check-activities-job:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - run: echo "${{ toJson(github.event) }}"
```

**Concepts Demonstrated:**
- ✅ Filtering PR events by type (`opened`, `edited`)
- ✅ Combined triggers (`pull_request` + `workflow_dispatch`)
- ✅ Inspecting event payload with `toJson()`

---

### Workflow 7: GitHub Info Pipeline

📄 **File:** [`gitHubInfo-pipeline.yml`](.github/workflows/gitHubInfo-pipeline.yml-old)

```yaml
name: gitHub information pipeline
on: workflow_dispatch

jobs:
  github-info-job:
    runs-on: ubuntu-latest
    steps:
      - run: echo "${{ toJson(github) }}"
```

**Concepts Demonstrated:**
- ✅ Manual-only trigger (`workflow_dispatch`)
- ✅ Dumping the entire `github` context for debugging

---

## Workflow Visualisation

```
┌─────────────────────────────────────────────────────────────────────┐
│                    customActions-pipeline.yml                       │
│                                                                     │
│   ┌──────────┐      ┌───────────┐      ┌────────────┐             │
│   │ test-job  │─────▶│ build-job  │─────▶│ deploy-job  │            │
│   │          │      │           │      │            │             │
│   │ checkout │      │ checkout  │      │ download   │             │
│   │ custom   │      │ custom    │      │  artifact  │             │
│   │  action  │      │  action   │      │ deploy     │             │
│   │ lint     │      │ build     │      │            │             │
│   │ test     │      │ upload    │      │            │             │
│   │ (upload  │      │  artifact │      │            │             │
│   │  on fail)│      │           │      │            │             │
│   └──────────┘      └───────────┘      └────────────┘             │
│                                                                     │
│   ┌─────────────────────────────────────┐                          │
│   │      .github/custom-actions/         │                          │
│   │  ┌─────────────────────────────┐    │                          │
│   │  │  Composite Action           │    │                          │
│   │  │  inputs: should-cache       │    │                          │
│   │  │  outputs: used-cache        │    │                          │
│   │  │  steps: cache → npm ci      │    │                          │
│   │  └─────────────────────────────┘    │                          │
│   └─────────────────────────────────────┘                          │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    testing-pipeline.yml                              │
│                                                                     │
│   ┌──────────┐      ┌────────────────────────────────┐             │
│   │ test-job  │─────▶│ deploy-job (reusable workflow)  │            │
│   │          │      │                                │             │
│   │ checkout │      │  uses: reusable-pipeline.yml   │             │
│   │ node 20  │      │  with: artifact-name: dist     │             │
│   │ install  │      │                                │             │
│   │ test     │      └────────────────────────────────┘             │
│   └──────────┘                                                      │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    matirx-pipeline.yml                               │
│                                                                     │
│   ┌─────────────────────────────────────────────────────┐          │
│   │  build-job (matrix)                                  │          │
│   │                                                      │          │
│   │  Node 16.x + Ubuntu  │  Node 16.x + Windows        │          │
│   │  Node 18.x + Ubuntu  │  Node 18.x + Windows        │          │
│   │  Node 24.x + Ubuntu  │  (24.x + Windows excluded)  │          │
│   └─────────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Important Notes & Best Practices

### Security Considerations

> ⚠️ **Pull Requests from Forks**
> - By default, PRs based on forks do **NOT** trigger any pipelines/workflows
> - The owner/admin of the main repository must approve and run the pipeline manually
> - This is a security measure to prevent malicious code execution from untrusted forks

> ⚠️ **Secrets in Forks**
> - Secrets are **never** passed to workflows triggered by PRs from forks
> - Use `pull_request_target` only when you fully understand the security implications

### Best Practices Checklist

| Practice | Why |
|----------|-----|
| **Use `needs:` to control job order** | Makes dependencies explicit and prevents wasted compute |
| **Use artifacts for multi-job workflows** | Each job runs on a fresh VM — files don't persist between jobs |
| **Cache dependencies** | Speeds up workflow execution dramatically |
| **Use specific action versions** (e.g., `@v3`, or SHA) | Avoids unexpected breaking changes |
| **Name your steps clearly** | Makes logs easier to debug |
| **Use `npm ci` instead of `npm install`** | More reliable for CI/CD — uses exact lock file versions |
| **Use `paths-ignore` or `paths`** | Avoids running workflows on irrelevant changes (e.g., docs) |
| **Use `$GITHUB_OUTPUT`** instead of `::set-output` | `::set-output` is deprecated and has security issues |
| **Use composite actions for repeated steps** | DRY principle — define once, use everywhere |
| **Use reusable workflows for repeated jobs** | Reduces duplication across workflow files |
| **Set `permissions:` explicitly** | Principle of least privilege |
| **Use environment variables for user input** | Prevents script injection attacks |
| **Use `continue-on-error` wisely** | Lets matrix builds complete even if some combos fail |
| **Use `fail-fast: false` in matrix** | Runs all combos to get full picture of failures |

### Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `**.md` in paths filter | Use `**/*.md` — double glob needs `/*.ext` |
| Forgetting `shell: bash` in composite actions | Every `run:` in a composite action requires explicit `shell:` |
| Not checking out code before using local actions | `uses: ./.github/custom-actions` requires `actions/checkout` first |
| Using `::set-output` | Deprecated — use `>> $GITHUB_OUTPUT` |
| Using `::set-env` | Deprecated — use `>> $GITHUB_ENV` |
| Not matching artifact names between upload/download | Names must be **identical** |
| Putting secrets in logs | Use env vars and never `echo` secrets |

---

## Important Resources

| Resource | Link |
|----------|------|
| GitHub Actions Documentation | [docs.github.com/actions](https://docs.github.com/en/actions) |
| Workflow Syntax Reference | [Workflow syntax](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions) |
| GitHub Context Reference | [Contexts](https://docs.github.com/en/actions/learn-github-actions/contexts) |
| GitHub Actions Expressions | [Expressions](https://docs.github.com/en/actions/learn-github-actions/expressions) |
| Events that trigger workflows | [Events](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows) |
| GitHub Actions Cache | [actions/cache](https://github.com/actions/cache) |
| Reusable Workflows | [Reusing workflows](https://docs.github.com/en/actions/using-workflows/reusing-workflows) |
| Creating Composite Actions | [Composite actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) |
| Security Hardening | [Security hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions) |
| GitHub Actions Marketplace | [Marketplace](https://github.com/marketplace?type=actions) |
| Encrypted Secrets | [Secrets](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions) |
| Self-hosted Runners | [Self-hosted runners](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners) |
