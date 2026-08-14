---
name: git-commit
description: 在个人 feature 分支上安全创建 Git commit、推送远程同名分支，并在开发完成时以远程 main（或明确指定的 base 分支）为基准 rebase 后创建 GitHub PR。用户提到提交代码、commit、推送 feature 分支、完成 feature、同步 main、rebase main、创建 PR 或提交 PR 时使用。仅要求 commit 时不要推送或创建 PR。
---

# Feature Branch Git 工作流

此 skill 适用于：从非默认的个人 feature 分支持续开发、反复 commit 和普通 push；开发完成后才同步远程 `main` 并创建 PR。

## 操作与授权边界

- **commit**：仅创建当前 feature 分支的本地提交。
- **commit + push**：创建提交后，普通推送到远程同名 feature 分支。
- **完成 feature / 同步 main / 创建 PR**：在 feature 完成时 fetch 并 rebase 到远程 base 分支；仅在用户明确要求时创建 PR。

push 授权不等同于 rebase 或创建 PR 授权；创建 PR 也不等同于可 force push。非预期失败立即停止并报告。禁止：`git push --force`、`git push --force-with-lease`、`git reset --hard`、`git clean`、自动 stash/unstash、自动取消暂存。

## 0. 建立分支角色与安全门禁

```bash
git rev-parse --is-inside-work-tree
git branch --show-current
git status --short
git diff --cached --name-only
git remote -v
git rev-parse --abbrev-ref --symbolic-full-name @{upstream} 2>/dev/null || true
```

定义：

- `<feature-branch>`：当前本地分支。
- `<feature-remote>`：feature 分支的推送远程；优先从 upstream 得出。没有 upstream 时使用用户指定的 remote；未指定且仅有 `origin` 时使用 `origin`。
- `<feature-ref>`：远程同名 feature 分支，例如 `origin/feature/login`；只在该分支存在时定义。
- `<base-remote>`：PR 的目标 remote。用户未指定时，默认与 `<feature-remote>` 相同。
- `<base-branch>`：用户指定的 PR base 分支；未指定时，在 fetch `<base-remote>` 后通过 `<base-remote>/HEAD` 获取默认分支（通常为 `main`）。
- `<base-ref>`：`<base-remote>/<base-branch>`，例如 `origin/main`。

门禁：

- 不在 Git 仓库、处于 detached HEAD、没有当前分支时停止。
- 在 `<base-branch>`、默认分支（例如 `main` / `master`）或其他共享保护分支上时，停止；要求用户先创建或切换到个人 feature 分支。不要自动创建或切换分支。
- 若启动时已有 staged 内容，停止并列出路径。不得自行纳入 commit、取消暂存或重写用户的 index，除非用户明确授权该 staged 内容属于本次提交。
- 有多个 remote 且无法确定 feature 或 base remote 时，先询问；不能假定所有 remote 都是同一仓库。

## 1. 确定本次提交范围

`<scope>` 是本轮任务中 AI 实际修改且用户允许提交的**具体文件路径集合**。不能因文件出现在工作区就自动纳入。

```bash
git status --short
git diff --name-only
git ls-files --others --exclude-standard
```

- 若有 `<scope>` 外的已修改、删除或未跟踪文件，停止并报告；由用户决定扩大范围或保留其未提交。
- 若同一文件混入无法安全分离的本任务外修改，停止并询问；不要猜测性使用交互式暂存。
- 用户明确要求提交工作区全部改动，且启动时 staged 区为空时，才可将全部当前改动作为 `<scope>`。
- 仅暂存明确路径：

```bash
git add -- <scope 中的具体文件路径>
git diff --cached --name-only
git diff --cached --stat
git diff --cached --check
git diff --cached
```

- 禁止 `git add -A`、`git add .` 及未带路径的 `git add`。
- 暂存后新增的文件必须都属于 `<scope>`；不符时停止报告，绝不自动 unstage。
- `git diff --cached --quiet` 表示没有新增内容：不创建 commit；若未要求 push，报告“无新变更”并结束。
- `git diff --cached --check` 失败时停止；不得 commit、push 或创建 PR。

### 敏感信息门禁

审查 staged 文件名和完整 staged diff。发现 `.env`、私钥/证书、密钥文件、token、密码或生产凭据等疑似敏感信息时停止，并只报告路径和原因，不回显敏感值。

- 项目已有 `gitleaks`、`detect-secrets`、pre-commit 或同类检查时，优先按现有方式执行。
- 无扫描器时人工检查常见私钥头、访问令牌和硬编码密码；无法判断时停止询问。

## 2. 验证并创建本地 commit

运行用户指定的验证命令。未指定时，查看项目已有脚本、CI 配置或贡献文档，选择与变更最相关且成本合理的 lint、typecheck、test 或 build。

- 验证失败时停止并报告；不得 commit、push 或创建 PR。
- 没有可用项目检查时，记录“未发现项目验证命令”；`git diff --cached --check` 仍为最低必做检查。
- 基于最终 `git diff --cached` 实际内容生成 Conventional Commit message：`<type>: <摘要>`；type 仅限 `feat / fix / docs / style / refactor / perf / test / chore / build / ci`，摘要单行、不超过 50 个字符、不加句号。

```bash
git commit -m "<message>"
git rev-parse --short HEAD
```

记录 `<commit-hash>` 与 `<commit-message>`。commit 失败时停止。

## 3. 日常推送 feature 分支

仅当用户要求 push 时执行。日常迭代的目标是远程同名 feature 分支，**不在每次 push 前 rebase `main`**。

```bash
git fetch <feature-remote> --prune
git show-ref --verify --quiet refs/remotes/<feature-remote>/<feature-branch>
```

- 若同名远程分支存在，设 `<feature-ref>`，并检查：

```bash
git log --oneline <feature-ref>..HEAD
git diff --name-status <feature-ref>..HEAD
```

- 待推送提交必须属于当前 feature 的已授权历史；存在无关本地提交或文件时停止，绝不顺带 push。
- 若不存在 `<feature-ref>`，这是首次推送。确认本地历史均属于当前 feature 后使用：

```bash
git push -u <feature-remote> <feature-branch>
```

- 若 `<feature-ref>` 存在，使用普通 push：

```bash
git push <feature-remote> <feature-branch>
```

- 若普通 push 被 non-fast-forward 拒绝，fetch 后重新检查 feature 分支范围；不要自动 rebase、merge 或 force push。停止并让用户决定后续同步策略。

## 4. 完成 feature 后同步远程 main

只有用户明确表示 feature 已完成、要求“同步/rebase main”，或要求创建 PR 时才执行本节。日常 commit/push 不执行本节。

```bash
git fetch <base-remote> --prune
git symbolic-ref --quiet refs/remotes/<base-remote>/HEAD 2>/dev/null || true
git show-ref --verify --quiet refs/remotes/<base-remote>/<base-branch>
git status --short
git rebase <base-ref>
```

- 用户未指定 `<base-branch>` 时，从 `<base-remote>/HEAD` 得到默认分支；无法获得时停止询问，不能武断使用 `main`。
- rebase 前工作区必须干净，且没有进行中的 merge/rebase/cherry-pick；否则停止。
- `<base-ref>` 是唯一的 rebase 基准，通常为 `origin/main`；**绝不 rebase 到远程 feature 分支 `<feature-ref>`。**
- rebase 发生任何冲突时，停止并报告冲突路径和 Git 状态。不要自动编辑冲突、`rebase --continue`、`rebase --abort` 或创建额外 commit；用户需要解决时移交 `resolving-merge-conflicts` skill。
- rebase 成功后重新运行项目验证，并检查 feature 相对 base 的历史和 diff：

```bash
git log --oneline <base-ref>..HEAD
git diff --name-status <base-ref>..HEAD
```

- 如果本次 rebase 改写了已推送 feature 分支的历史，普通 push 将被拒绝。由于本 skill 禁止 force push，**停止且不要创建 PR**，报告需由用户另行选择：明确授权受保护的 `git push --force-with-lease`、改用 merge `<base-ref>`，或自行处理历史。不得自行选择其中任一方式。
- 若 feature 尚未推送，或 rebase 后仍能安全进行普通 fast-forward push，则按第 3 节推送 feature 分支。

## 5. 创建或确认 GitHub PR

仅当用户明确要求创建 PR，且 feature 已包含最终 rebase 结果并已成功推送时执行。PR 的 head 是 `<feature-branch>`，base 是 `<base-branch>`；普通 push 不创建 PR。

先检查是否已有同一 head/base 的开放 PR：

```bash
gh pr list --head <feature-branch> --base <base-branch> --state open --json number,url,headRefName,baseRefName
```

- 已有 PR 时，验证 `headRefName` 与 `baseRefName`，报告 URL 和编号；不要重复创建。
- 没有 PR 时，使用实际 commit 信息自动填充标题和正文：

```bash
gh pr create --base <base-branch> --head <feature-branch> --fill
```

- 创建后读取 PR 的 number、URL、head 和 base 并核验目标。若 GitHub CLI 未认证、仓库非 GitHub 或创建失败，停止并报告；不要改用网页自动化或其他 provider。

## 6. 输出报告

按实际结果输出，不能将本地 commit、feature push、同步 main 和 PR 创建混为一谈：

```markdown
## Feature 提交报告

**Feature 分支**: <feature-branch>
**Feature 远程**: <feature-remote>/<feature-branch> / 未请求推送
**PR 基准**: <base-ref> / 未执行最终同步
**本次提交**: <commit-hash> <commit-message> / 无新增提交

### 变更与验证
- <n> 个文件变更，+x / -y 行
- <命令>: 通过 / 未运行（原因）

### 同步状态
- 日常迭代，未 rebase main / 已 rebase <base-ref> / rebase 冲突中止

### PR 状态
- 未请求 / 已存在：#<number> <url> / 已创建：#<number> <url>

### 最终状态
- ✅ 已提交并推送 feature 分支
- ✅ 已同步 main、推送 feature 分支并创建/更新 PR
- ⚠️ 已创建本地提交，尚未推送（原因）
- ⚠️ rebase 后需要用户决定历史同步方式，未推送且未创建 PR
- ❌ 验证失败，未提交
- ⚠️ 冲突或远程同步中止（附下一步建议）
```
