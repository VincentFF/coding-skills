---
name: git-commit
description: Two commit modes. Mode 1 (auto-commit) commits a complete unit of AI changes locally after relevant checks pass, preserving pre-existing user changes; confirm in one line, never push. Mode 2 (manual commit) runs on explicit user request — commit the files changed in this task by default, or all worktree changes when the user says "commit all"; remind the user that push/PR are available but never do them unprompted. Use before AI file edits to record existing changes and after a complete unit is verified, and when the user mentions committing code, pushing a feature branch, finishing a feature, syncing/rebasing main, or creating a PR.
---

# Git Commit — Two Modes

- **Mode 1 — auto-commit**: after a complete unit of AI changes and its relevant checks pass, commit that unit locally. One-line confirmation, never push.
- **Mode 2 — manual commit**: only on explicit user request. Default scope = files changed in this task; "commit all" (提交全部) = whole worktree.

Push, rebase, and PR run only on explicit user request (Section 3) — a commit never implies them.

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

Report — one line only, no detailed report:

```
✅ 已提交 <short-hash>: <message>
```

## Mode 2 — Manual commit

Scope:

- **Default**: AI changes made during this task, plus changes the user explicitly requests. Naming a path does not authorize unrelated pre-existing edits within it; clarify inclusion when changes are mixed. Dirty/untracked files outside scope → report them, let the user decide.
- **"commit all"**: everything in `git status --short`, still staged as explicit paths.

Procedure:

1. Stop if not a git work tree or detached HEAD. Content the user pre-staged → list it and ask before including.
2. Stage scope paths; check `git diff --cached --stat` and `--check`. Nothing staged → report "no new changes" and end. Staged content outside scope → stop, never auto-unstage.
3. Verify: the user's command if given, else the cheapest relevant project check (lint/typecheck/test/build); `git diff --cached --check` is the minimum. Fail → stop, no commit.
4. `git commit -m "<message>"`, then record `git rev-parse --short HEAD`.

Report: a short block — commit hash + message, files changed (+/-), verification result — clearly stating what was *not* done (push/rebase/PR). Always end with the reminder:

> 已提交到本地。如需 push、rebase main 或创建 PR，请明确告诉我。

## 3. Push / Rebase / PR (explicit request only)

`<feature-branch>` = current branch. Push remote: upstream if set, else user-specified, else `origin` if it's the only remote; ambiguous → ask.

**Push** — plain push to the same-named remote branch; do not rebase main first:

```bash
git fetch <remote> --prune
git log --oneline <remote>/<feature-branch>..HEAD   # if remote branch exists
```

Unrelated outgoing commits → stop; never push them along. First push: `git push -u <remote> <feature-branch>`; later: plain `git push`. Non-fast-forward rejection → stop; the user picks the sync strategy.

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
