---
name: git-commit
description: 一键完成 git 工作流：拉取远程最新代码、自动 add 并生成简短精炼的 commit message、智能评估并处理冲突、推送到远程同名分支，最后输出 commit 报告。当用户要求提交代码、commit、同步并推送变更时使用。
---

# Git Commit 工作流

按以下步骤严格执行。任何一步失败都要立即停止并报告，不要盲目继续。

## 0. 采集上下文

```bash
git branch --show-current
git status -sb
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null || true
```

- 记录当前分支名 `<branch>`。
- 若当前分支没有 upstream，后续先检查 `origin/<branch>` 是否已存在；若不存在，推送时使用 `git push -u origin <branch>`。

## 1. 自动 add 并生成 commit

```bash
git status --porcelain
git diff --stat
git add -A
git diff --cached --stat
```

**生成 commit message 的规则（简短精炼）：**

- 使用 Conventional Commits 格式：`<type>: <摘要>`
- type 从 `feat / fix / docs / style / refactor / perf / test / chore / build / ci` 中选择
- 摘要不超过 50 个字符，一行写完，不加句号
- 基于 `git diff --cached` 的实际内容总结，不要臆测
- 示例：`fix: 修复登录接口超时重试逻辑`、`chore: 更新依赖版本`

- 如果 `git diff --cached --quiet` 表示没有暂存变更，跳过 commit，但**不要结束流程**，继续执行第 2 步和第 4 步。
- 如果有暂存变更，基于 `git diff --cached` 生成 message，然后执行：

```bash
git commit -m "<message>"
```

## 2. 拉取远程最新内容

先执行：

```bash
git fetch --all --prune
```

然后按分支情况处理：

- **已有 upstream：**

```bash
git pull --rebase
```

- **没有 upstream：**
  1. 检查远程同名分支是否存在：

```bash
git ls-remote --exit-code --heads origin <branch>
```

  2. 如果存在，执行：

```bash
git rebase origin/<branch>
```

  3. 如果不存在，说明这是首次推送分支，跳过本步的 rebase，保留第 4 步使用 `git push -u origin <branch>`。

- 如果 pull / rebase 过程中产生冲突，进入第 3 步的冲突处理流程。

## 3. 冲突评估与处理

发生冲突时，先收集信息：

```bash
git status --porcelain | grep -E '^(UU|AA|AU|UA|DU|UD)' || true
git diff --name-only --diff-filter=U
```

对每个冲突文件读取冲突内容，按以下标准判断严重程度：

**严重冲突（满足任一条件）→ 停止操作，直接返回冲突报告：**

- 冲突涉及 3 个及以上文件
- 单个文件冲突块超过 3 处
- 冲突涉及核心逻辑：数据库 schema / 迁移文件、认证鉴权、支付、加密、公共 API 接口签名、生产配置
- 双方修改语义矛盾，无法机械合并（例如同一函数被双方以不同方式重写）
- 涉及 lock 文件或大规模自动生成文件，无法明确判断保留哪一侧更安全

严重时执行 `git rebase --abort`（或 `git merge --abort`），输出冲突报告（见第 5 步格式），**不要**强行提交。

**轻微冲突 → 自动修复：**

- 双方只是修改了同一文件的不同逻辑但恰好在相邻行
- 追加式冲突（双方都新增 import、新增配置项等），可以安全地两边都保留
- 修复后运行项目可用的 lint / test / build（如果存在对应脚本）验证无误，然后：

```bash
git add <修复的文件>
git rebase --continue   # 或 git commit 完成 merge
```

## 4. 推送到远程同名分支

```bash
git branch --show-current
```

- 如果当前分支已有 upstream，执行：

```bash
git push origin <当前分支名>
```

- 如果没有 upstream，或确认远程同名分支尚不存在，执行：

```bash
git push -u origin <当前分支名>
```

- 如果推送被 reject（non-fast-forward），不要 force push，先同步一次再重试：
  1. 有 upstream 时执行 `git pull --rebase`
  2. 无 upstream 但远程分支已存在时执行 `git rebase origin/<branch>`
  3. 若产生冲突，进入第 3 步
  4. 重试 push 一次；仍失败则停止并报告

## 5. 生成 commit 报告

流程结束后，输出如下格式的报告：

```markdown
## 📋 Commit 报告

**分支**: <branch> → origin/<branch>
**提交**: <commit-hash> <commit-message> / 无新增本地提交

### 变更概览
- <n> 个文件变更，+x / -y 行
- 主要变更：
  - path/to/file1: 简述
  - path/to/file2: 简述

### 冲突处理
- 无冲突 / 自动修复了 n 处冲突（列出文件）/ 因严重冲突中止

### 状态
✅ 已推送 / ⚠️ 中止（原因）
```

若因严重冲突中止，报告改为：

```markdown
## ⚠️ 冲突报告

**冲突文件**:
- path/to/file: 冲突原因简述

**评估**: 为什么判定为严重冲突
**建议**: 人工处理的具体步骤
**当前状态**: 已执行 rebase/merge --abort，工作区恢复到操作前状态
```