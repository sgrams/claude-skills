---
applyTo: "**"
description: "Repository-specific overrides for the /prrr skill. Provides this repo's test command, commit format, and push remote conventions so /prrr does not need to guess."
---

# PRRR — Repo-Specific Overrides

Place this file at `.github/instructions/prrr.instructions.md` in any repository
to customize how `/prrr` behaves for that project. The `applyTo: "**"` frontmatter
ensures these overrides are loaded automatically whenever `/prrr` runs.

## Test Command

Specify the command(s) to run before committing:

```bash
# Examples:
# npm test
# pytest
# cargo test --workspace
# make test
```

If certain changes require additional test suites, note them here:

```bash
# Example: if the change touches module X, also run:
# npm run test:integration
```

## Commit Format

Define the commit message conventions for this repository:

```
<type>(<scope>): <subject>

<body — required, references #<ghid>>

Signed-off-by: <name> <email>   # if DCO is required
```

- Scope: the affected module, package, or component.
- Types: `fix`, `feat`, `docs`, `refactor`, `test`, `chore`
- Sign-off: include `git commit -s` if the project uses DCO.

## Push Remote

Describe which remote to push feature branches to:

- If contributors use forks, the push remote is the user's fork (not `origin`).
- Detect the fork remote by matching the GitHub username against `git remote -v`.
- PR base is always `origin/<default-branch>`.

## Build Verification

List any build or lint checks to run before pushing:

```bash
# Examples:
# npm run build
# cargo build --release
# make lint
```
