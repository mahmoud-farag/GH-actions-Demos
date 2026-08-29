# GitHub Actions - Comprehensive Guide

## Table of Contents
1. [What is GitHub Actions?](#what-is-github-actions)
2. [Workflow Triggers (Events)](#workflow-triggers-events)
3. [Core Concepts](#core-concepts)
4. [Practical Workflows](#practical-workflows)
5. [Important Resources](#important-resources)

---

## What is GitHub Actions?

GitHub Actions is a CI/CD (Continuous Integration/Continuous Deployment) platform that allows you to automate tasks triggered by events in your repository. Workflows are defined in YAML files located in `.github/workflows/` directory.

**Key Components:**
- **Workflow**: An automated process defined in a YAML file
- **Event**: A specific activity that triggers a workflow (e.g., push, pull request)
- **Job**: A set of steps that execute on the same runner
- **Step**: A single task (action or shell command) within a job
- **Action**: A reusable unit of code (can be created or sourced from the marketplace)
- **Runner**: A server that runs your workflows

---

## Workflow Triggers (Events)

Workflows are triggered by specific events in your repository. Below is a comprehensive list:

### Commit & Branch Events
- **`push`** - Triggered when a commit or tag is pushed to the repository (includes branch pushes and tag pushes)
- **`create`** - Triggered when a branch or tag is created in the repository
- **`delete`** - Triggered when a branch or tag is deleted from the repository

### Pull Request Events
- **`pull_request`** - Triggered when a PR is opened, closed, reopened, edited, synchronized (new commits pushed), assigned, labeled, etc.
- **`pull_request_review`** - Triggered when a PR review is submitted, edited, or dismissed (approve/request changes/comment)
- **`pull_request_review_comment`** - Triggered when a comment is made on a PR's diff (inline code review comment) — created, edited, or deleted
- **`pull_request_target`** - Like `pull_request`, but runs in the context of the base repository, giving access to secrets — use carefully for PRs from forks

### Issue Events
- **`issues`** - Triggered when an issue is opened, edited, deleted, closed, reopened, assigned, labeled, pinned, etc.
- **`issue_comment`** - Triggered when a comment is created, edited, or deleted on an issue OR a pull request (since PRs are issues under the hood)

### Release & Deployment Events
- **`release`** - Triggered when a release is published, unpublished, created, edited, deleted, or pre-released
- **`page_build`** - Triggered when someone pushes changes to a GitHub Pages-enabled branch (build attempt for GitHub Pages)

### Repository Events
- **`fork`** - Triggered when someone forks the repository to their own account/org
- **`watch`** - Triggered when someone stars the repository (historically called "watch," now maps to starring)
- **`star`** - Triggered when a repository is starred or unstarred (newer, more explicit alternative/companion to watch)
- **`member`** - Triggered when a collaborator is added, removed, or their permissions change on the repository
- **`public`** - Triggered when a private repository is made public

### Discussion Events
- **`discussion`** - Triggered when a GitHub Discussion is created, edited, answered, deleted, pinned/unpinned, etc.
- **`discussion_comment`** - Triggered when a comment on a GitHub Discussion is created, edited, or deleted

### Code Events
- **`commit_comment`** - Triggered when a comment is made directly on a commit (not on a PR diff, but on the commit itself)
- **`gollum`** - Triggered when a wiki page in the repository is created or updated

### Project Board Events
- **`project`** - Triggered when a classic GitHub Project (project board) is created, updated, closed, reopened, or deleted
- **`project_card`** - Triggered when a card in a classic Project board is created, moved, updated, converted to an issue, or deleted
- **`project_column`** - Triggered when a column in a classic Project board is created, updated, moved, or deleted

### Integration & External Events
- **`repository_dispatch`** - Triggered by a custom external webhook event sent via the GitHub API — used to trigger workflows from outside GitHub (e.g., another CI system)
- **`status`** - Triggered when the status of a Git commit changes (e.g., an external CI reports pending/success/failure)
- **`workflow_dispatch`** - Allows for manually running the workflow from the Actions tab in your repository

---

## Core Concepts

### 1. Job Execution Model

**Parallel Execution (Default)**
- All jobs in a workflow run in parallel by default
- Each job runs on its own isolated runner (virtual machine)

**Sequential Execution**
- Use the `needs:` key to make jobs run sequentially
- A job will only run if all of its dependency jobs succeed

```yaml
jobs:
  test-job:
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  build-job:
    needs: test-job  # This job only runs after test-job succeeds
    runs-on: ubuntu-latest
    steps:
      - run: npm run build
```

### 2. Job Artifacts

**What are Job Artifacts?**
- Files/assets generated during the build phase (e.g., compiled code, dist folders)
- Temporary data that persists between jobs within a workflow

**Usage:**
- Use `actions/upload-artifact@v4` to store artifacts from one job
- Use `actions/download-artifact@v4` to retrieve artifacts in a subsequent job
- Useful for multi-job workflows (test → build → deploy)

```yaml
# Upload artifacts
- name: Upload Build Artifact
  uses: actions/upload-artifact@v4
  with:
    name: build-artifact
    path: dist

# Download artifacts
- name: Get build artifact
  uses: actions/download-artifact@v4
  with:
    name: build-artifact  # Must match the upload name
```

### 3. Job Outputs

- Pass data from one job to another using outputs
- Accessed via `needs.<job-id>.outputs.<output-name>`

```yaml
build-job:
  outputs:
    script-file: ${{ steps.unique-id.outputs.script-file }}
  steps:
    - id: unique-id
      run: echo "script-file=myfile.js" >> $GITHUB_OUTPUT

deploy-job:
  needs: build-job
  steps:
    - run: echo "File is ${{ needs.build-job.outputs.script-file }}"
```

### 4. GitHub Context

The `github` context provides information about the workflow run and the event that triggered it:
- `${{ github }}` - Complete GitHub context as JSON
- `${{ github.event }}` - Event payload details
- `${{ github.actor }}` - Username who triggered the workflow
- `${{ github.ref }}` - Git reference (branch/tag)
- `${{ github.repository }}` - Repository name

Use `${{ toJson(github) }}` or `${{ toJson(github.event) }}` to output context as JSON.

### 5. Actions

Reusable units of code that perform common tasks:
- **`actions/checkout@v3`** - Checks out your repository code to the runner
- **`actions/setup-node@v7`** - Sets up a specific Node.js version
- **`actions/upload-artifact@v4`** - Uploads files as artifacts
- **`actions/download-artifact@v4`** - Downloads artifacts from previous jobs

### 6. Path Filters

Control when workflows run based on file paths that changed:

**`paths`** - Only run if specific paths are modified
```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'package.json'
```

**`paths-ignore`** - Run for all changes EXCEPT specified paths
```yaml
on:
  push:
    paths-ignore:
      - '**/*.md'      # Ignores all markdown files in any directory
      - 'docs/**'      # Ignores entire docs folder
      - '.gitignore'   # Ignores specific files
```

**Common Glob Patterns:**
- `**/*.md` - All markdown files in any directory
- `*.md` - Only markdown files in root directory
- `src/**` - All files in src folder
- `src/**/*.js` - All JS files in src and subdirectories

⚠️ **Important:** Use `**/*.md` NOT `**.md` (the latter is invalid syntax and will be ignored)

### 7. Caching Dependencies

Improve workflow performance by caching dependencies:
- Use `actions/cache` to cache npm packages, dependencies, etc.
- Reduces installation time on subsequent workflow runs

[Learn more about caching](https://github.com/actions/cache)

### 8. GitHub Actions Expressions

Dynamically evaluate values in your workflows:
- Use `${{ expression }}` syntax
- Supports functions like `toJson()`, `contains()`, `startsWith()`, etc.

[GitHub Actions Expressions Documentation](https://docs.github.com/en/actions/reference/workflows-and-actions/expressions)

---

## Practical Workflows

### Workflow 1: Testing Pipeline (`testing-pipeline.yml`)

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
    needs: test-job  # Only runs if test-job succeeds
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
      - name: Run Build
        run: npm run build
      - name: Deploy
        run: echo "Deploying..."
```

**Key Concepts:**
- Triggered on `push` and `workflow_dispatch` (manual trigger)
- `test-job` runs first
- `deploy-job` waits for `test-job` success (using `needs:`)
- Sets up Node.js version 20

---

### Workflow 2: Job Artifact Pipeline (`jobArtifact-pipeline.yml`)

```yaml
name: Job Artifact demo
on:
  push:
    branches:
      - main

jobs:
  test-job:
    runs-on: ubuntu-latest
    steps:
      - name: Get The Code
        uses: actions/checkout@v3
      - name: install dependencies
        run: npm ci
      - name: lint Code
        run: npm run lint
      - name: Test Code
        run: npm run test

  build-job:
    runs-on: ubuntu-latest
    needs: test-job
    outputs:
      script-file: ${{ steps.unique-id.outputs.script-file }}
    steps:
      - name: Get The Code
        uses: actions/checkout@v3
      - name: install dependencies
        run: npm ci
      - name: Build website
        run: npm run build
      - name: Publish JS files to the deploy-job
        id: unique-id
        run: find dist/assets/*.js -type f -execdir echo '::set-output name=script-file::{}' ';'
      - name: Upload Build Artifact
        uses: actions/upload-artifact@v4
        with:
          name: build-artifact
          path: dist

  deploy-job:
    runs-on: ubuntu-latest
    needs: build-job
    steps:
      - name: Get build artifact
        uses: actions/download-artifact@v4
        with:
          name: build-artifact
      - name: List the build artifact
        run: ls
      - name: Display the JS files name
        run: echo "the JS file name is ${{ needs.build-job.outputs.script-file }}"
      - name: Deploy website
        run: echo "Deploying website..."
```

**Key Concepts:**
- Three-stage pipeline: test → build → deploy
- Each stage depends on the previous one
- `build-job` uploads artifacts (the `dist` folder)
- `deploy-job` downloads artifacts from `build-job`
- Job outputs pass data between jobs

---

### Workflow 3: Activity Events Pipeline (`activity-pipeline.yml`)

```yaml
name: check github activities events
on:
  pull_request:
    types: [opened, edited]  # Specific PR actions
  workflow_dispatch

jobs:
  check-activities-job:
    runs-on: ubuntu-latest
    steps:
      - name: Get The Code
        uses: actions/checkout@v3
      - name: Get Github Activities Context
        run: echo "${{toJson(github.event)}}"
```

**Key Concepts:**
- Triggered on `pull_request` with specific types (opened/edited)
- Also allows manual trigger via `workflow_dispatch`
- Outputs the event context as JSON to inspect PR details

---

### Workflow 4: GitHub Info Pipeline (`gitHubInfo-pipeline.yml`)

```yaml
name: gitHub information pipeline
on: workflow_dispatch

jobs:
  github-info-job:
    runs-on: ubuntu-latest
    steps:
      - name: Get Github Information Context
        run: echo "${{toJson(github)}}"
```

**Key Concepts:**
- Only triggered manually via `workflow_dispatch`
- Outputs the complete GitHub context
- Useful for debugging and understanding available context variables

---

## Important Notes

### Security Considerations

⚠️ **Pull Requests from Forks**
- By default, PRs based on forks do NOT trigger any pipelines/workflows
- The owner/admin of the main repository must approve and run the pipeline manually
- This is a security measure to prevent malicious code execution from untrusted forks

### Best Practices

1. **Use `needs:` to control job order** - Makes dependencies explicit
2. **Use artifacts for multi-job workflows** - Avoid rebuilding in each job
3. **Cache dependencies** - Speeds up workflow execution
4. **Use specific action versions** - Avoid unexpected behavior from version changes
5. **Name your steps clearly** - Makes logs easier to debug
6. **Use `npm ci` instead of `npm install`** - More reliable for CI/CD environments

---

## Important Resources

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Context Reference](https://docs.github.com/en/actions/reference/workflows-and-actions/contexts)
- [GitHub Actions Expressions](https://docs.github.com/en/actions/reference/workflows-and-actions/expressions)
- [GitHub Actions Cache](https://github.com/actions/cache)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
