---
name: test-commit-push
description: Run the project's full test suite, and if all tests pass, create a signed git commit following the repo's CONTRIBUTING.md conventions and push to the current branch. Use this skill whenever the user wants to test-and-commit, test-and-push, or says things like "run the tests and if they pass commit and push", "ship it", "test commit push", "run tests then push my changes", or asks to commit and push after finishing a task. Also trigger when the user types "/test-commit-push". Do not trigger for test-only requests (just "run the tests") or commit-only requests (just "commit this") — this skill is specifically for the combined test+commit+push workflow.
---

# Test, Commit & Push

This skill automates the test-then-commit-then-push workflow. The idea is that you should never push code that doesn't pass tests, so this skill enforces that gate: tests must pass before any commit happens.

## Step-by-step workflow

### 1. Understand what changed

Before doing anything, get a clear picture of the current state:

- Run `git status` to see what files are modified, staged, or untracked.
- Run `git diff` (staged + unstaged) to understand the changes.
- Identify which crate(s) were modified — this matters for the commit message scope.

If there are no changes to commit, tell the user and stop. No point running tests if there's nothing to ship.

### 2. Run quality checks

All three checks below must pass before committing. If any check fails, stop immediately — show the failures clearly and do not proceed to commit or push. The whole point is that broken or messy code doesn't get pushed.

Look at the project to determine the right commands. If the project has a `CLAUDE.md` file, check it first — it often specifies the exact commands.

#### a) Formatting check

Verify code is properly formatted. Run the check in non-modifying mode — the goal is to detect issues, not silently fix them (the user should see what's wrong).

- **Rust/Cargo**: `cargo fmt --check`
- **Node/npm**: `npx prettier --check .` or check `package.json` scripts
- **Python**: `ruff format --check .` or `black --check .`
- **Go**: `gofmt -l .` (list files that differ)

If formatting issues are found, show which files need formatting and stop. Tell the user to run the formatter first.

#### b) Linting

Run the project's linter to catch warnings and code quality issues.

- **Rust/Cargo**: `cargo clippy`
- **Node/npm**: `npm run lint` or `npx eslint .`
- **Python**: `ruff check .` or `flake8`
- **Go**: `go vet ./...` and/or `golangci-lint run`

If there are lint warnings or errors, show them and stop.

#### c) Tests

Run the full test suite.

- **Rust/Cargo**: `cargo test`
- **Node/npm**: `npm test` or check `package.json` scripts
- **Python**: `pytest`, `python -m pytest`, or check for a test script
- **Go**: `go test ./...`
- **General**: check `Makefile`, `justfile`, or CI config for test commands

Show the user the test output so they can see what happened.

### 3. Read commit conventions

Check for a `CONTRIBUTING.md` file in the repo root. If it exists, read it and extract the commit message format, any signing requirements, and other conventions.

If there's no `CONTRIBUTING.md`, fall back to the `CLAUDE.md` commit format section if it exists. If neither exists, use [Conventional Commits](https://www.conventionalcommits.org/) — e.g. `feat(scope): description`, `fix(scope): description`, with types like `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`.

### 4. Create the commit

Based on the changes and the commit conventions you found:

1. **Stage the changed files.** Prefer staging specific files by name rather than `git add -A` — this avoids accidentally including sensitive files (`.env`, credentials) or build artifacts. If unsure which files to stage, ask the user.

2. **Write the commit message.** Follow the format from CONTRIBUTING.md. Typical elements to get right:
   - Commit type (`feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `style`)
   - Scope — usually the crate or module that was changed
   - Description — imperative mood, concise, focused on *what* and *why*
   - Line length limits (often 80 chars)
   - Sign-off flag (`-s`) if required
   - Co-Authored-By trailer for LLM-generated commits

3. **Present the commit message to the user for approval before committing.** This is important — the user should see exactly what will be committed and have a chance to adjust the message. Don't just commit silently.

4. **Create the commit.** Always use both `-s` (sign-off) and `-S` (GPG/SSH sign). For multi-line commits, use separate `-m` flags for each line — never use `\n` within a message string:
   ```bash
   git commit -s -S -m "[type](scope): description" -m "Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
   ```
   If the commit needs a body paragraph, add it as another `-m` between the title and the trailer:
   ```bash
   git commit -s -S -m "[type](scope): description" -m "Body text here." -m "Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>"
   ```

### 5. Push to the current branch

After committing:

1. Check what branch you're on with `git branch --show-current`.
2. Check if the branch has an upstream tracking branch.
3. Push: `git push` (or `git push -u origin <branch>` if no upstream is set).
4. Confirm success to the user, showing the branch name and remote.

Do not create a pull request — just push. The user explicitly wants a direct push workflow.

### 6. Report the result

Give the user a brief summary:
- Tests: passed
- Committed: `[the commit message]`
- Pushed to: `origin/<branch-name>`

## Edge cases

- **Multiple crates changed**: If changes span multiple crates in a workspace, consider whether to make one commit with `(trx-rs)` scope (for repo-wide changes) or suggest separate commits per crate. Ask the user if it's ambiguous.
- **Untracked files**: If there are new untracked files that look like they should be part of the change, mention them to the user and ask if they should be included.
- **Dirty working tree after staging**: If some changes are intentionally left unstaged, that's fine — just commit what's staged.
- **Protected branches**: If pushing to `main` or `master`, warn the user that this is a protected branch push and confirm they want to proceed.
