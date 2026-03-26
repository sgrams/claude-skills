---
name: prrr
description: "Automate the full lifecycle of resolving a GitHub issue and opening a pull request. Use when: the user wants to fix a GitHub issue end-to-end, create a branch from an issue, implement a fix/feature from issue description, open a PR that closes an issue, or run /prrr <ghid>. Triggers on keywords: prrr, resolve issue, fix issue, implement issue, issue to PR, close issue."
argument-hint: "<ghid> — GitHub issue number to resolve (e.g., 42)"
---

# PRRR — Issue-to-PR Automation

Resolves a GitHub issue end-to-end: reads contributing guidelines, creates a branch, implements the fix/feature, runs tests, pushes, and opens a pull request.

## Input

- `<ghid>` — A GitHub issue number (required).

## Procedure

### Step 0: Parse Input

Extract the issue number from the user's invocation. If `<ghid>` is missing or not a valid number, stop immediately:
> **Error:** No valid issue ID provided. Usage: `/prrr <ghid>` (e.g., `/prrr 42`).

### Step 1: Read Contributing Guidelines

1. Check if `CONTRIBUTING.md` exists at the repository root.
2. If it exists, read and parse it in full. Extract and remember:
   - **Branch naming** conventions (if any).
   - **Commit message format** (type, scope, subject, body, sign-off requirements).
   - **PR title/description** conventions or templates.
   - **Code standards**, test requirements, documentation expectations.
3. If `CONTRIBUTING.md` does not exist, fall back to these defaults:
   - Branch name: `prrr/<ghid>`
   - Commit message: `fix(project): <issue title>` with a descriptive body
   - PR title: the commit title
   - PR description: summary of changes + `Closes #<ghid>`
   - Sign-off: use `git commit -s` if the repo has any DCO/sign-off mentions

Also check for:
- `.github/PULL_REQUEST_TEMPLATE.md` or `.github/PULL_REQUEST_TEMPLATE/` — use the template for PR body if present.
- `.github/ISSUE_TEMPLATE/` — for understanding issue structure.

### Step 2: Detect Repository Remotes

1. Run `git remote -v` to list all configured remotes.
2. Identify:
   - **origin** — the main upstream remote (PR target).
   - **Feature remote** — the remote to push the branch to. This may be a fork remote (not origin). Heuristics:
     - If only `origin` exists, use `origin` for both push and PR target.
     - If multiple remotes exist, prefer the one matching the current user's GitHub username for pushing, and `origin` as the PR base.
3. Determine the **default branch** by running `git remote show origin | grep 'HEAD branch'` or inspecting `git symbolic-ref refs/remotes/origin/HEAD`.

### Step 3: Create Branch

1. Fetch the latest from origin: `git fetch origin`.
2. Check if `prrr/<ghid>` already exists locally or remotely:
   - `git branch --list prrr/<ghid>`
   - `git ls-remote --heads <push-remote> prrr/<ghid>`
3. If the branch already exists, stop:
   > **Error:** Branch `prrr/<ghid>` already exists. Delete it first or use a different approach.
4. Create and check out: `git checkout -b prrr/<ghid> origin/<default-branch>`.

### Step 4: Fetch and Analyze the Issue

1. Use the GitHub tool to fetch issue `#<ghid>`:
   - Retrieve title, body, labels, comments, and any linked PRs.
2. If the issue does not exist or is inaccessible, stop:
   > **Error:** GitHub issue #<ghid> not found or inaccessible. Verify the issue number and repository permissions.
3. Analyze the issue to understand:
   - **What** needs to change (bug description, feature request, task).
   - **Where** in the codebase the change applies (search for relevant files, modules, tests).
   - **Acceptance criteria** from the issue body or comments.
4. Explore the codebase as needed using search, file reads, and subagents to build full context before implementing.

### Step 5: Implement the Resolution

Implement a **complete, production-quality** resolution:

1. **Code changes**: Write the fix or feature. Follow code standards from CONTRIBUTING.md and existing codebase patterns.
2. **Tests**: Add or update tests to cover the change. If the repo has an existing test framework, follow its conventions.
3. **Documentation**: Update any relevant docs, README sections, or changelogs if required by CONTRIBUTING.md.
4. **Verify**: Ensure the code compiles/builds and passes lint checks where applicable.

Do NOT cut corners — the implementation should be ready for review.

### Step 6: Test, Commit, and Push

Invoke `/test-commit-push` to handle this step. If `/test-commit-push` is not available, perform these steps manually:

1. **Run tests**: Execute the project's test suite. If tests fail, diagnose and fix before proceeding.
   - If tests fail and the fix is unclear, stop and report:
     > **Error:** Tests failed after implementation. See output above. The branch `prrr/<ghid>` has been created with your changes but has not been pushed.
2. **Stage changes**: `git add -A` (or selectively stage relevant files).
3. **Commit**: Create a commit following CONTRIBUTING.md conventions. Example:
   ```
   git commit -s -m "<type>(<scope>): <subject>" -m "<body referencing #ghid>"
   ```
   - The commit body should explain what changed and why.
   - Include `Signed-off-by` if required.
4. **Push**: `git push <push-remote> prrr/<ghid>`.
   - If the push is rejected (e.g., permissions, protected branch), stop:
     > **Error:** Push to `<push-remote>` rejected. Check remote permissions and branch protection rules.

### Step 7: Open the Pull Request

1. Determine PR parameters:
   - **Base**: `<default-branch>` on `origin`.
   - **Head**: `prrr/<ghid>` on the push remote.
   - **Title**: Follow CONTRIBUTING.md conventions. Default: the commit title.
   - **Body**: Follow PR template if one exists. Must include:
     - Summary of the changes.
     - `Closes #<ghid>` to auto-link the issue.
     - Any testing notes or reviewer guidance.
2. Open the PR using the GitHub PR tool.
3. Report the PR URL to the user.

### Step 8: Summary

Print a summary:
```
✅ Issue #<ghid> resolved
   Branch: prrr/<ghid>
   Commit: <short-sha> <commit-title>
   PR: <pr-url>
```

## Error Handling

All errors must be **clear and actionable**. The skill stops on:

| Condition | Message |
|-----------|---------|
| Missing `<ghid>` | No valid issue ID provided. Usage: `/prrr <ghid>` |
| Issue not found | GitHub issue #<ghid> not found or inaccessible |
| Branch exists | Branch `prrr/<ghid>` already exists |
| Tests fail | Tests failed — see output. Branch created but not pushed |
| Push rejected | Push rejected — check remote permissions |
| PR creation fails | PR creation failed — see error details. Branch pushed successfully |

## Constraints

- **No hardcoded values**: All repo names, remotes, branch names, and conventions are discovered at runtime.
- **Generic**: Works across any repository with any language, build system, or test framework.
- **CONTRIBUTING.md is authoritative**: If it exists, its conventions override all defaults.
- **Atomic steps**: Each step validates its preconditions before executing.
