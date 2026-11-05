---
name: git-commit
description: Two commit modes. Mode 1 (auto-commit) commits a complete unit of AI changes after relevant checks pass, preserving pre-existing user changes, then backs up eligible feature branches by pushing. Mode 2 (manual commit) runs on explicit user request — commit the files changed in this task by default, or all worktree changes when the user says "commit all"; apply the same feature-branch backup rules; create PRs only on explicit request. Use before AI file edits to record existing changes and after a complete unit is verified, and when the user mentions committing code, pushing a feature branch, finishing a feature, syncing/rebasing main, or creating a PR.
---

# Git Commit — Two Modes

- **Mode 1 — auto-commit**: after a complete unit of AI changes and its relevant checks pass, commit that unit locally, then apply the feature-branch backup rules in Section 3. Confirm the commit and push outcome in one line.
- **Mode 2 — manual commit**: only on explicit user request. Default scope = files changed in this task; "commit all" (提交全部) = whole worktree.

After each successful commit in either mode, automatically push eligible feature branches under Section 3 unless the user requests local-only work. This is standing authorization for feature-branch backup, not for pushing the default branch, rebasing, or creating PRs; those still require an explicit request. More restrictive repository rules take precedence.

Forbidden always: `git push --force` / `--force-with-lease`, `git reset --hard`, `git clean`, auto stash/unstash, auto unstaging, `git add -A` / `git add .` (stage explicit paths only). On any unexpected failure: stop and report.

**Secrets gate (both modes)**: scan staged file names and diff for `.env`, private keys/certs, tokens, passwords → stop, report paths only, never values. Run the project's scanner (gitleaks, pre-commit, etc.) if one exists.

Commit message (both modes): Conventional Commit `<type>: <summary>`; type ∈ `feat / fix / docs / style / refactor / perf / test / chore / build / ci`; summary one line, ≤50 chars, no trailing period.

## Protect Existing Changes (both modes)

Before editing, inspect the worktree and index and record existing staged, unstaged, and untracked changes in files the task will touch. Before staging or committing, compare the final diff with that baseline; file ownership alone does not establish change ownership. Preserve the user's edits and index state.

For auto-commit, stage and commit only files whose complete changes belong to the completed unit. If a touched file contains pre-existing or concurrent user changes, leave it untouched by staging/commit and report that auto-commit is blocked; do not split a required unit merely to commit its other files. Unrelated dirty or staged files do not block auto-commit and must remain outside it. If no reliable baseline was recorded for a mixed file, stop auto-commit rather than infer ownership. Manual requests may include user changes only within the explicitly authorized scope below.

## Mode 1 — Auto-commit after a verified unit

Trigger: a coherent unit is complete, including its required implementation, tests, fixtures, callers, and documentation, and the relevant project checks pass. A conversation turn or an individual file write is not a commit boundary. Use the user's requested checks or the cheapest relevant project validation; `git diff --check` is the minimum for documentation-only changes. If work is incomplete, checks fail or cannot run, preserve the worktree and report without committing.

Skip silently if not a git work tree, detached HEAD, or no changes in the unit's files.

After applying the existing-change protection and secrets gate, stage explicit paths, inspect their staged diff and run the whitespace check for those paths. Commit only that unit:

```bash
git add -- <unit files>
git diff --cached --quiet -- <unit files>   # nothing → skip
git diff --cached -- <unit files>
git diff --cached --check -- <unit files>   # failure → report, no commit
git commit -m "<message>" -- <unit files>
```

After committing, apply Section 3 and report the commit and push outcome in one line. If push stops, state that the commit remains local and why:

```
✅ 已提交 <short-hash>: <message>；<已推送到 remote/branch，或未推送及原因>
```

## Mode 2 — Manual commit

Scope:

- **Default**: AI changes made during this task, plus changes the user explicitly requests. Naming a path does not authorize unrelated pre-existing edits within it; clarify inclusion when changes are mixed. Dirty/untracked files outside scope → report them, let the user decide.
- **"commit all"**: everything in `git status --short`, still staged as explicit paths.

Procedure:

1. Stop if not a git work tree or detached HEAD. Content the user pre-staged → list it and ask before including.
2. Stage scope paths; check `git diff --cached --stat` and `--check`. Nothing staged → report "no new changes" and end. Staged content outside scope → stop, never auto-unstage.
3. Verify: the user's command if given, else the cheapest relevant project check (lint/typecheck/test/build); `git diff --cached --check` is the minimum. Fail → stop, no commit.
4. `git commit -m "<message>"`, then record `git rev-parse --short HEAD` and apply Section 3 for feature-branch backup.

Report a short block with the commit hash and message, files changed (+/-), verification result, and actual push outcome. If push stops, state that the commit remains local and why. Rebase and PR creation still require an explicit request.

## 3. Feature-branch Backup / Rebase / PR

**Automatic push** — after a successful commit, push the current feature branch only when all conditions below hold. A local-only user instruction skips this step. Never create a branch merely to push a commit already made on the default branch.

1. Resolve the remote from the branch's upstream, else the user-specified remote, else `origin` only when it is the sole remote. An ambiguous or missing remote stops the push. Require a same-named remote branch; an upstream pointing elsewhere stops automatic push.
2. Fetch the chosen remote with `git fetch <remote> --prune`. Establish its actual default branch from remote metadata (`refs/remotes/<remote>/HEAD`, verifying with `git ls-remote --symref <remote> HEAD` when missing or uncertain). If the current branch is the default branch or the default cannot be established, keep the commit local and report. Authentication or network failure stops the push without retrying through another identity or remote.
3. Inspect every outgoing commit: compare against the same-named remote branch if it exists, otherwise the fetched remote default branch. Confirm the entire outgoing set belongs to the authorized task and inspect its changes for secrets under the secrets gate. Unrelated commits, uncertain ownership, or sensitive content stop the push; never publish them along with the task commit.
4. Push only the current branch to the same-named remote branch. First push: `git push -u <remote> HEAD:refs/heads/<feature-branch>`; subsequent pushes: `git push <remote> HEAD:refs/heads/<feature-branch>`. Non-fast-forward rejection or any other failure stops the push; preserve the local commit and report. Do not rebase, merge, force-push, or open a PR as an automatic recovery.

An explicit request to push the default branch is handled separately, with the same remote, outgoing-commit, secrets, and fast-forward checks. Automatic backup does not grant that authorization.

**Rebase** — only when the user says the feature is finished or asks to sync/rebase main. Base: user-specified, else the default branch from `<base-remote>/HEAD` after fetch; unresolved → ask, never assume `main`.

```bash
git fetch <base-remote> --prune
git rebase <base-remote>/<base-branch>
```

Dirty worktree or conflict → stop, report conflicted paths; do not touch conflicts or run `--continue`/`--abort` (hand off to the `resolving-merge-conflicts` skill if the user wants). After a clean rebase, re-verify per Mode 2 step 3. If the branch was already pushed and the rebase rewrote history → **stop, no push, no PR**; present the options (explicitly authorized guarded `--force-with-lease`, merge the base instead, or user handles it) — never choose.

**PR** — only on explicit request, after the final state is pushed. Check for an existing open PR first:

```bash
gh pr list --head <feature-branch> --state open --json number,url
gh pr create --base <base-branch> --head <feature-branch> --fill   # if none
```

Existing → report URL, don't duplicate. `gh` unauthenticated or creation failure → stop and report; no browser fallback.
