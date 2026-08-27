---
name: git-commit
description: Create Git commits on a personal feature branch, push to the same-named remote branch, and — only when the feature is finished — rebase onto the remote base branch (e.g. main) and create a GitHub PR. Use when the user mentions committing code, pushing a feature branch, finishing a feature, syncing/rebasing main, or creating a PR. Commit-only requests must not push or create a PR.
---

# Feature Branch Git Workflow

Three operations with separate authorization. Never let one imply the next:

- **commit**: local commit only.
- **commit + push**: commit, then plain push to the same-named remote branch.
- **finish feature**: fetch and rebase onto the remote base branch; create a PR only on explicit request.

Forbidden at all times: `git push --force`, `git push --force-with-lease`, `git reset --hard`, `git clean`, automatic stash/unstash, automatic unstaging. Stop and report on any unexpected failure.

## 0. Branch Roles and Safety Gates

```bash
git rev-parse --is-inside-work-tree
git branch --show-current
git status --short
git diff --cached --name-only
git remote -v
git rev-parse --abbrev-ref --symbolic-full-name @{upstream} 2>/dev/null || true
```

- `<feature-branch>`: current local branch.
- `<feature-remote>`: push remote; from upstream if set, else the user-specified remote, else `origin` if it is the only remote.
- `<feature-ref>`: same-named remote branch (e.g. `origin/feature/login`); exists only after first push.
- `<base-remote>`: PR target remote; defaults to `<feature-remote>`.
- `<base-branch>`: user-specified PR base; else the default branch from `<base-remote>/HEAD` after fetch (usually `main`).
- `<base-ref>`: `<base-remote>/<base-branch>`, e.g. `origin/main`.

Stop when:

- Not in a Git work tree, detached HEAD, or no current branch.
- Staged content exists at startup: list the paths and wait; do not touch the user's index without explicit authorization.
- Multiple remotes make `<feature-remote>` or `<base-remote>` ambiguous: ask.

## 1. Commit Scope

Commit only the paths this task actually changed and the user approved — that set is `<scope>`. A file being dirty in the worktree is not approval.

```bash
git status --short
git diff --name-only
git ls-files --others --exclude-standard
```

Stop and report when:

- Dirty or untracked files exist outside `<scope>` — the user decides whether to widen scope or leave them.
- One file mixes in-task and out-of-task changes that cannot be staged separately — do not guess with interactive staging.

Exception: if the user explicitly asks to commit all worktree changes and the index was empty at startup, `<scope>` is all current changes.

Stage explicit paths only — never `git add -A`, `git add .`, or path-less `git add`:

```bash
git add -- <specific paths in scope>
git diff --cached --name-only
git diff --cached --stat
git diff --cached --check
git diff --cached
```

Then:

- Newly staged content must all be inside `<scope>`; otherwise stop and report — never auto-unstage.
- `git diff --cached --quiet` → nothing to commit. Report "no new changes" and end (unless push was requested).
- `git diff --cached --check` fails → stop; no commit, push, or PR.

### Sensitive Data Gate

Scan staged file names and the full staged diff. On suspected secrets — `.env`, private keys/certificates, tokens, passwords, production credentials — stop; report paths and reasons only, never the values.

- If the project has `gitleaks`, `detect-secrets`, pre-commit, or similar, run it.
- Otherwise manually check for private-key headers, access tokens, hardcoded passwords. Unsure → stop and ask.

## 2. Verify and Commit

Run the user's verification command if given. Otherwise pick the most relevant cheap check from project scripts or CI config (lint, typecheck, test, build). If none exists, note it; `git diff --cached --check` is the mandatory minimum.

Verification fails → stop and report; no commit, push, or PR.

Build a Conventional Commit message from the final staged diff: `<type>: <summary>` where type is one of `feat / fix / docs / style / refactor / perf / test / chore / build / ci`; summary is one line, ≤50 chars, no trailing period.

```bash
git commit -m "<message>"
git rev-parse --short HEAD
```

Record `<commit-hash>` and `<commit-message>`. Stop if commit fails.

## 3. Push Feature Branch

Run only when the user asks to push. Target is the same-named remote branch; do **not** rebase `main` first.

```bash
git fetch <feature-remote> --prune
git show-ref --verify --quiet refs/remotes/<feature-remote>/<feature-branch>
```

If `<feature-ref>` exists, inspect what would be pushed:

```bash
git log --oneline <feature-ref>..HEAD
git diff --name-status <feature-ref>..HEAD
```

Every outgoing commit must belong to this feature's authorized history. Unrelated local commits or files → stop; never push them along.

Push:

```bash
# first push (no <feature-ref>)
git push -u <feature-remote> <feature-branch>

# subsequent pushes
git push <feature-remote> <feature-branch>
```

Non-fast-forward rejection → fetch and re-check scope, then stop. No auto-rebase, merge, or force push; the user picks the sync strategy.

## 4. Rebase onto Base After Finishing

Run only when the user says the feature is finished, asks to sync/rebase main, or asks for a PR. Never during routine commit/push.

```bash
git fetch <base-remote> --prune
git symbolic-ref --quiet refs/remotes/<base-remote>/HEAD 2>/dev/null || true
git show-ref --verify --quiet refs/remotes/<base-remote>/<base-branch>
git status --short
git rebase <base-ref>
```

Rules:

- `<base-branch>` unresolved from `<base-remote>/HEAD` → stop and ask; do not assume `main`.
- Dirty worktree or merge/rebase/cherry-pick in progress → stop.
- Rebase only onto `<base-ref>` — never onto `<feature-ref>`.
- Any conflict → stop and report conflicted paths and status. Do not edit conflicts, run `rebase --continue`/`--abort`, or add commits. Hand off to the `resolving-merge-conflicts` skill if the user wants resolution.

After a clean rebase, re-run verification (Section 2) and inspect:

```bash
git log --oneline <base-ref>..HEAD
git diff --name-status <base-ref>..HEAD
```

Then:

- If the branch was already pushed and the rebase rewrote its history, plain push will fail and force push is forbidden → **stop, no PR**. Report the user's options: explicitly authorize a guarded `git push --force-with-lease`, merge `<base-ref>` instead, or handle it themselves. Never choose for them.
- Otherwise push per Section 3.

## 5. Create or Confirm the GitHub PR

Run only on explicit PR request, after the final rebase result is pushed. Head = `<feature-branch>`, base = `<base-branch>`.

Check for an existing open PR first:

```bash
gh pr list --head <feature-branch> --base <base-branch> --state open --json number,url,headRefName,baseRefName
```

- Exists → verify `headRefName`/`baseRefName`, report URL and number; do not duplicate.
- Missing → create with title/body auto-filled from commits:

```bash
gh pr create --base <base-branch> --head <feature-branch> --fill
```

Verify the created PR's number, URL, head, and base. `gh` unauthenticated, non-GitHub repo, or creation failure → stop and report; never fall back to browser automation or another provider.

## 6. Report

Report what actually happened; never conflate commit, push, rebase, and PR:

```markdown
## Feature Commit Report

**Feature branch**: <feature-branch>
**Feature remote**: <feature-remote>/<feature-branch> / push not requested
**PR base**: <base-ref> / final sync not performed
**This commit**: <commit-hash> <commit-message> / no new commit

### Changes & Verification
- <n> files changed, +x / -y lines
- <command>: passed / not run (reason)

### Sync Status
- Routine iteration, no rebase / rebased onto <base-ref> / rebase stopped on conflict

### PR Status
- Not requested / existing: #<number> <url> / created: #<number> <url>

### Final Status
- ✅ Committed and pushed feature branch
- ✅ Synced main, pushed, PR created/updated
- ⚠️ Committed locally, not pushed (reason)
- ⚠️ Rebase rewrote pushed history — awaiting user's sync decision; not pushed, no PR
- ❌ Verification failed, nothing committed
- ⚠️ Conflict or sync aborted (next-step suggestion)
```
