---
name: git-commit
description: Auto-commit after a completed task in a Git repo — commit that task's files, then push the branch. Use when finishing a unit of work, or when the user asks to commit or push.
---

# Git Commit — Auto-commit

Finished a complete, verifiable task → commit its files → push the branch. Nothing else.

Skip when: not a repo, detached HEAD, nothing changed in the task's files, or the work is incomplete or its checks fail.

1. **Scope** — stage explicit paths whose changes are entirely this task's. A file mixed with user edits stays out; report it as blocked. Unrelated dirty files stay untouched.
2. **Secrets** — scan staged names and diff for `.env`, keys, certs, tokens, passwords → stop, report paths only, never values.
3. **Commit**
   ```bash
   git add -- <paths>
   git diff --cached --check -- <paths>           # fail → report, no commit
   git commit -m "<type>: <summary>" -- <paths>
   ```
   Message: `<type>: <summary>`, type ∈ `feat fix docs style refactor perf test chore build ci`, ≤50 chars, no period.
4. **Push** — current branch to the same-named remote branch: `git push -u <remote> HEAD:refs/heads/<branch>` first, then `git push <remote> HEAD:refs/heads/<branch>`. Remote = upstream, else user-specified, else the sole remote. The default branch stays local. Non-fast-forward or any failure → stop and report.
5. **Report** — `✅ Committed <hash>: <message>; pushed to <remote>/<branch> | not pushed: <reason>`

Never without an explicit request: force push, `reset --hard`, `clean`, stash, unstaging, `add -A`, rebase, branch deletion, PR.
