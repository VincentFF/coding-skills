---
name: git-commit
description: Two commit modes. Mode 1 (auto-commit) runs automatically every time the AI finishes modifying files — commit just those files locally and confirm in one line, never push. Mode 2 (manual commit) runs on explicit user request — commit the files changed in this task by default, or all worktree changes when the user says "commit all"; remind the user that push/PR are available but never do them unprompted. Use after any AI file edit, and when the user mentions committing code, pushing a feature branch, finishing a feature, syncing/rebasing main, or creating a PR.
---

# Git Commit — Two Modes

- **Mode 1 — auto-commit**: after each round of AI file modifications, commit those files locally. One-line confirmation, never push.
- **Mode 2 — manual commit**: only on explicit user request. Default scope = files changed in this task; "commit all" (提交全部) = whole worktree.

Push, rebase, and PR run only on explicit user request (Section 3) — a commit never implies them.

Forbidden always: `git push --force` / `--force-with-lease`, `git reset --hard`, `git clean`, auto stash/unstash, auto unstaging, `git add -A` / `git add .` (stage explicit paths only). On any unexpected failure: stop and report.

**Secrets gate (both modes)**: scan staged file names and diff for `.env`, private keys/certs, tokens, passwords → stop, report paths only, never values. Run the project's scanner (gitleaks, pre-commit, etc.) if one exists.

Commit message (both modes): Conventional Commit `<type>: <summary>`; type ∈ `feat / fix / docs / style / refactor / perf / test / chore / build / ci`; summary one line, ≤50 chars, no trailing period.

## Mode 1 — Auto-commit after file edits

Trigger: after each completed round of AI edits/writes. Skip silently if not a git work tree, detached HEAD, or no changes in the touched files.

Scope = exactly the files just modified. The user's own dirty or staged files are left alone — never block on them, never include them.

```bash
git add -- <files modified this round>
git diff --cached --quiet -- <files modified this round>   # nothing → skip
git diff --cached --check                                  # fail → one-line warning, no commit
git commit -m "<message>" -- <files modified this round>   # pathspec keeps user pre-staged content out
```

Report — one line only, no detailed report:

```
✅ 已提交 <short-hash>: <message>
```

## Mode 2 — Manual commit

Scope:

- **Default**: files changed during this task, plus paths the user names. Dirty/untracked files outside scope → report them, let the user decide.
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
