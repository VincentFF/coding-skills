## 修改范围

更新以下两处现有文件：

- `coding/git-commit/SKILL.md`：将工作流改为“feature 分支持续 commit/push，最终以远程 main 为基准 rebase，再按明确请求创建 PR”。
- `AGENTS.md`：补充项目级提醒，让完成开发后的提交询问同时覆盖“是否已准备好最终同步 main 并创建 PR”；不会自动创建分支、rebase、push 或 PR。

不修改 `resolving-merge-conflicts`：它继续只处理用户已明确交给它的冲突。`git-commit` 在最终 rebase 发生冲突时会停止，并明确移交给该 skill。

## `git-commit` 设计调整

1. **明确三类操作与授权范围**
   - `commit`：仅向当前 feature 分支创建本地提交。
   - `commit + push`：向远程同名 feature 分支做普通 fast-forward push。
   - `完成 feature / 创建 PR`：在最终阶段 fetch 并 rebase 到 `<base-remote>/<base-branch>`（默认经验证的 `origin/main`），成功后按明确授权创建 GitHub PR。
   - push 授权不等同于 PR 授权；创建 PR 仅在用户明确请求时执行。

2. **区分 base 与 feature 引用**
   - 定义 `<base-remote>`、`<base-branch>` / `<base-ref>`（例如 `origin/main`），以及 `<feature-branch>`、`<feature-remote>` / `<feature-ref>`（例如 `origin/feature/login`）。
   - 从目标 remote 的 HEAD 获取默认 base branch，用户可显式指定其他 base branch；无法确认时停止询问。
   - 当前分支必须是非默认 feature 分支；在 `main`、detached HEAD 或 base branch 上不创建 feature commit/push，改为停止并要求创建或切换 feature 分支。

3. **保留并细化日常迭代流程**
   - 保持当前 staged/index、scope、敏感信息和验证门禁。
   - 日常 feature 开发的重复 commit/push **不 rebase 已推送分支**；正常普通 push 到远程同名 feature 分支。
   - feature upstream 不存在时，通过 `git push -u <feature-remote> <feature-branch>` 建立同名远程分支。
   - 继续禁止所有 force push 和自动历史改写；feature remote 分支的比较用于确认待推送历史，而不是作为 rebase 基准。

4. **最终 PR 前同步 main**
   - 仅在用户声明 feature 已完成、要求创建 PR，或明确要求同步 main 时执行：fetch `<base-remote>`，检查 `<base-ref>`，然后 `git rebase <base-ref>`。
   - 由于用户选择“最终 PR 前处理”，不在每次日常 push 前 rebase main。
   - rebase 冲突时停止、报告冲突路径和当前 rebase 状态；不自动编辑、continue 或 abort，提示显式使用 `resolving-merge-conflicts`。
   - 若 rebase 后 feature 先前已 push，普通 push 会因历史重写失败。依据当前禁止 force-push 策略，skill 将停止并清楚说明：创建 PR 前的 rebase 需要用户另行决定是否允许受保护的 `--force-with-lease`，或改为 merge main / 手动处理；不会自动 force push。

5. **创建并校验 PR**
   - 只在最终 rebase 后 feature 分支已安全推送，且用户明确要求时调用 GitHub CLI：`gh pr create --base <base-branch> --head <feature-branch>`。
   - 创建前检查是否已有 head→base PR，避免重复创建；已有 PR 时报告其 URL/编号而非新建。
   - 验证新建或已有 PR 的 base、head、URL/编号，确保目标是 `<base-branch>` 而非 feature remote ref。
   - 保持不填写猜测性的 PR 标题/正文：没有用户内容时根据实际 diff 生成简短标题，并说明正文需用户提供或使用项目模板。

6. **更新报告结构**
   - 显示 feature 分支与远程、base ref、是否已在最终阶段同步 main、最新 feature commit、push 状态，以及 PR 状态（未请求 / 已存在 / 已创建 / 因 rebase 后需要显式 force-with-lease 授权而未创建）。
   - 明确区分日常迭代完成和 PR 前同步被安全门禁阻止的状态。

## 验证

- 审核 frontmatter、占位符和命令引用的一致性。
- 用文字场景逐项复查：
  1. 新 feature 的首次 commit/push；
  2. 已推送 feature 的后续日常 commit/push（不 rebase main）；
  3. 完成 feature 后 rebase `origin/main`，因已推送历史而需要用户决定是否 force-with-lease；
  4. 已同步且可普通推送时创建或识别现有 PR。
- 执行 `git diff --check`，并核对未跟踪的 `.zcode/` 不会被本次修改触及。