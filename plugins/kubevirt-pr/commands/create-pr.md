---
description: Push current branch and create a GitHub PR using the KubeVirt PR template
argument-hint: [optional title override]
---

## Name
kubevirt-pr:create-pr

## Synopsis
```
/kubevirt-pr:create-pr [optional title override]
```

## Description
Creates a GitHub pull request for the current branch against `upstream/main` using the KubeVirt project's official PR template from `.github/PULL_REQUEST_TEMPLATE.md`.

## Implementation

### Phase 1: Validate State

1. Verify we are in a git repo and NOT on the `main` branch:
   ```bash
   BRANCH=$(git rev-parse --abbrev-ref HEAD)
   ```
   If `$BRANCH` is `main` or `master`, stop with an error.

2. Check for uncommitted changes:
   ```bash
   git status --porcelain
   ```
   If non-empty, warn the user and ask whether to proceed.

3. Detect the GitHub username from CLAUDE.md or git config:
   - Check CLAUDE.md for a `Git Identity` section with a GitHub username
   - Fall back to `gh api user --jq '.login'`

4. Determine the remote to push to. Look for a remote pointing to the user's fork:
   ```bash
   gh repo view --json owner --jq '.owner.login'
   PUSH_REMOTE=$(git remote -v | grep "github.com.*${GITHUB_USER}.*push" | awk '{print $1}' | head -1)
   ```
   If no fork remote found, fall back to `origin`.

### Phase 2: Gather Context

1. Get the diff against the base branch:
   ```bash
   BASE=$(git merge-base HEAD upstream/main 2>/dev/null || git merge-base HEAD origin/main)
   COMMITS=$(git log --oneline ${BASE}..HEAD)
   DIFF=$(git diff ${BASE} --stat)
   FULL_DIFF=$(git diff ${BASE})
   ```

2. Read the PR template from the repo:
   ```bash
   cat .github/PULL_REQUEST_TEMPLATE.md
   ```

3. If an argument was provided, use it as the PR title. Otherwise, derive the title from the branch name or the first commit message:
   ```bash
   TITLE=$(git log --format='%s' ${BASE}..HEAD | tail -1)
   ```

### Phase 3: Generate PR Description

Fill in the KubeVirt PR template by analyzing the commits and diff:

1. **`### What this PR does`**: Summarize what changed and why in 2-3 sentences. Derive from commit messages and diff.

2. **`#### Before this PR:`**: 2-3 short bullet points describing the current behavior/problem.

3. **`#### After this PR:`**: 2-3 short bullet points describing the new behavior.

4. **`### References`**: Include references if:
   - Commit messages mention issue numbers (`#1234`, `Fixes #1234`)
   - The branch name contains an issue number
   - Otherwise leave the section with just the HTML comment from the template

5. **`### Why we need it and why it was done in this way`**: For small fixes, keep this brief — one sentence about the tradeoff or "Straightforward fix, no alternatives considered." Remove the placeholder lines about tradeoffs/alternatives if they don't apply.

6. **`### Special notes for your reviewer`**: Leave empty unless the diff is large or touches sensitive areas.

7. **`### Checklist`**: Include as-is from the template. Do not check any boxes.

8. **`### Release note`**: Default to `NONE` unless the change is a user-facing bug fix or feature. If the commits mention a bug fix that affects users, write a one-line release note.

### Phase 4: Push and Create PR

1. Push the branch to the user's fork:
   ```bash
   git push -u ${PUSH_REMOTE} HEAD
   ```

2. Create the PR using `gh`:
   ```bash
   gh pr create --repo kubevirt/kubevirt --base main --head ${GITHUB_USER}:${BRANCH} --title "${TITLE}" --body "${BODY}"
   ```
   Use a HEREDOC for the body to preserve formatting.

3. Print the PR URL.

## Examples

1. **Create PR with auto-generated title**:
   ```
   /kubevirt-pr:create-pr
   ```

2. **Create PR with custom title**:
   ```
   /kubevirt-pr:create-pr Fix node-labeller tests for multi-arch clusters
   ```

## Prerequisites
- `gh` CLI authenticated with GitHub
- Current branch is not `main`/`master`
- A remote pointing to the user's fork of kubevirt/kubevirt
- `.github/PULL_REQUEST_TEMPLATE.md` exists in the repo
