---
on:
  schedule:
    - cron: "*/15 * * * *"
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: pipeline-orchestrator
  cancel-in-progress: false

permissions:
  contents: read
  issues: read
  pull-requests: read
  actions: read

engine:
  id: copilot
  model: claude-opus-4.6

tools:
  github:
    toolsets: [default, actions]
  bash:
    - "gh:api:graphql"

network:
  allowed:
    - defaults

safe-outputs:
  noop:
    report-as-issue: false
  dispatch-workflow:
    github-token: ${{ secrets.GH_AW_WRITE_TOKEN }}
    workflows: [issue-implementer, ci-fixer]
    max: 1
  add-reviewer:
    github-token: ${{ secrets.GH_AW_WRITE_TOKEN }}
    max: 3
  resolve-pull-request-review-thread:
    github-token: ${{ secrets.GH_AW_WRITE_TOKEN }}
    max: 10
  add-labels:
    github-token: ${{ secrets.GH_AW_WRITE_TOKEN }}
    max: 10
  remove-labels:
    github-token: ${{ secrets.GH_AW_WRITE_TOKEN }}
    max: 10
  add-comment:
    github-token: ${{ secrets.GH_AW_WRITE_TOKEN }}
    max: 5

---

# Pipeline Orchestrator

Own the full lifecycle of agent work: from issue to merged PR. Detect what needs attention and push it forward one step at a time.

## Context

This repository has an automated pipeline:
1. code-health or test-analysis creates issues (labeled `code-health` or `test-audit`)
2. issue-implementer creates a PR from the issue (labeled `aw`, auto-merge enabled)
3. Copilot auto-reviews the PR
4. review-responder addresses review comments and resolves threads
5. quality-gate approves if code quality is good and impact is low/medium
6. auto-merge fires when CI passes + approved + threads resolved

This orchestrator owns steps 2-6. It detects stalls and fixes them.

## Instructions

### Step 1: Find issues that need implementation

First, check if there are any open PRs with the `aw` label. If there are, skip issue dispatch entirely — only one agent PR should be in flight at a time to avoid merge conflicts.

Also check if any `issue-implementer` workflow runs are currently in progress (queued or running). If so, skip issue dispatch — an implementer is already working on something and will create a PR soon.

If there are NO open `aw`-labeled PRs AND no in-progress implementer runs, list open issues with the `code-health` or `test-audit` label. For each issue, check if there is already an open or recently merged PR that references it (look for PRs whose body contains "Closes #N" or "#N" where N is the issue number).

Dispatch the `issue-implementer` workflow for the **first** eligible issue only (one at a time). Add a comment on the issue: "Pipeline Orchestrator: dispatching issue-implementer."

If no issues need implementation, move on to Step 2.

### Step 2: Gather all PR state in one query

Run a single GraphQL query to get everything about open `aw`-labeled PRs. Use `$GITHUB_MCP_SERVER_TOKEN` for authentication:
```
GH_TOKEN="$GITHUB_MCP_SERVER_TOKEN" gh api graphql -f query='query($owner: String!, $name: String!) {
  repository(owner: $owner, name: $name) {
    pullRequests(first: 10, states: OPEN, labels: ["aw"]) {
      nodes {
        number
        headRefName
        mergeStateStatus
        reviewDecision
        autoMergeRequest { enabledAt }
        labels(first: 10) { nodes { name } }
        reviews(first: 10) { nodes { author { login } state } }
        reviewThreads(first: 100) {
          nodes {
            id
            isResolved
            comments(last: 1) {
              nodes { author { login } }
            }
          }
        }
        commits(last: 1) {
          nodes {
            commit {
              statusCheckRollup {
                contexts(first: 20) {
                  ... on CheckRun { name conclusion }
                }
              }
            }
          }
        }
      }
    }
  }
}' -f owner="$GITHUB_REPOSITORY_OWNER" -f name="${GITHUB_REPOSITORY#*/}"
```

From this single response, extract for each PR:
- Whether it has auto-merge enabled (skip if not)
- Whether it has the `aw-conflict` label (skip if yes)
- `mergeStateStatus`
- `reviewDecision`
- Whether any review is from `copilot-pull-request-reviewer`
- All unresolved thread IDs (`PRRT_` format) and the last comment author on each
- Whether the `check` CI job has conclusion `failure`
- Whether it has the `ci-fix-attempted` label

Sort PRs by progress: approved first, then unapproved.

### Step 3: Process each PR

For each PR (in sorted order), apply the actions below **in this exact order**. Apply only the **first matching** action, then move to the next PR:

#### Action 1: Request Copilot review

If no review from `copilot-pull-request-reviewer` exists in the reviews list, request a review from `@copilot` using the add-reviewer safe-output. Stop processing this PR.

#### Action 2: Resolve unresolved threads

If any review threads have `isResolved: false`, check the last comment's author on each. The review-responder posts replies as the value of `$GITHUB_REPOSITORY_OWNER` (the repository owner).

For each unresolved thread where the last comment author matches `$GITHUB_REPOSITORY_OWNER` or `github-actions[bot]`, resolve it using the resolve-pull-request-review-thread safe-output with the thread's `id` field.

If any unresolved threads remain where the last commenter is someone else (Copilot reviewer, human), stop processing this PR — it needs attention.

**This action takes priority over "behind main" — always resolve threads first, even if the PR is behind main.**

#### Action 3: CI failure

If the `check` job has conclusion `failure` and the PR does NOT have the `ci-fix-attempted` label, dispatch the `ci-fixer` workflow with the PR number as input. Stop processing this PR.

If CI failed but the PR already has `ci-fix-attempted`, skip — manual intervention needed.

#### Action 4: Behind main

If the PR is approved, all threads resolved, but `mergeStateStatus` is `BEHIND`:
- Log that the PR is approved and ready but needs a rebase to proceed.
- Do NOT attempt to rebase and do NOT comment on the PR — this requires manual intervention.
- Move to the next PR.

#### Action 5: All clear

If the PR is approved, threads resolved, and not behind main — auto-merge should handle it. Log this and move on.

### Step 4: Summary

After processing all PRs, output a brief summary of actions taken.

## Important rules

- Process PRs one at a time, in sorted order
- Apply only the FIRST matching action per PR, then move to the next
- If any API call fails for a PR, skip it and continue — one failure must not stop the entire run
- NEVER fabricate thread IDs — always use real IDs from the GraphQL response
- The `aw-conflict` label means merge conflicts exist — skip these PRs entirely
