# Claude Code Settings for Bigcapital

## Node.js Version

Always use Node.js 18.16.1 for this project. Before running any npm/pnpm/node commands:

```bash
nvm use 18.16.1
```

## Package Manager

Use `pnpm` for this project.

## Fork-local: working efficiently

Rules learned from real sessions. They cut repeated work, not quality.

### Automate what doesn't need judgement
- **No dependency bots on this fork** (they would flood PRs against upstream code). Upgrades are pulled in by syncing with upstream, one at a time.
- **Never run dependency upgrades in parallel.** They all touch the lockfile and re-conflict. One at a time, or one grouped PR.
- **Rules that can be tests, are tests.** Prefer a failing check over a prose rule a reviewer must remember.

### Make "green locally" mean "green in CI"
- Before opening a PR, run the same lint, format, typecheck and test steps the workflows in `.github/workflows/` run.
- A PR is merged only when it is up to date with the default branch and green.

### Keep the repo the source of truth
- A fork-only change that alters a live system (deploy, version, infra setting) updates the deploy notes in the same PR.
- Open questions that need the owner: list them in the PR description (one line each), skip those items and keep going instead of stopping. The owner answers in batches.

### Delegating to subagents
- Only when the work is genuinely parallel and non-overlapping (different files).
- Give a tight brief: the goal, the exact files/dirs, the commands, what "done" looks like. Point at only the docs that task needs.
- Batch small related fixes into one agent; use a smaller model for mechanical work (config, docs, version bumps), the strongest model for domain logic.

### Environment
- Use the Node version and package manager already documented above; do not add session hooks to this fork.

### PRs
- Squash-merge own PRs once CI is green and the branch is up to date; no need to ask again for each one.
- Stop watching a PR as soon as it merges.
