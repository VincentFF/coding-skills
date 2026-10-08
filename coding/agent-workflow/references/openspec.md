# OpenSpec 接入

只在用户或适用项目约定明确选择 OpenSpec 时使用。旧项目仅保留 `openspec/` 目录，不足以决定本次入口；适用项目规则要求 OpenSpec 时不能擅自改用轻量计划。

## 计划来源

加载当前安装的相关 `openspec-*` skills，按它们选择明确的 change，并读取 proposal、delta specs、design、tasks 及项目规定的其他权威材料。CLI 语法和产物结构以当前安装版本为准；所需 skill/CLI 缺失时报告接入阻塞，不猜测替代命令或自动安装。

选定 change 的原生任务台账是进度权威。直接使用这些产物，不另外创建 `docs/changes/<change>.md`、第二份任务列表或长期 spec 镜像。

项目额外要求的 ADR 或系统设计文档继续维护，作为原生任务的交付物或相关来源。

## 规划与执行

- 新建或修改计划按相应 OpenSpec skill 处理；用户要求实施现有且已批准的计划时，不因切换执行 skill 再生成一套文档或重复审批。
- 检查重要要求、任务、文档义务和验证是否一致。CLI 的 artifact completion、ready 或 validate 结果不能证明语义一致。
- 初次 worker 交接使用当前 apply instructions 给出的相关材料、任务和指导，再补具体写入边界、验收方法及必要事实来源。使用路径指针，不复制全部产物。
- 实现和证据规则沿用 Agent Workflow。缺陷回到写入者；契约冲突先通过相应更新流程或用户决定修订原生计划。

## 验收与生命周期

Agent Workflow 的独立审查与最终代码树门禁之外，还要完成适用项目和当前 OpenSpec skills 要求的严格校验、coherence verification 和文档检查。项目要求 `/opsx-verify` 时继续执行，团队审查不能替代它。

长期规范同步遵守该项目的 OpenSpec 生命周期，不为套用轻量入口而提前复制或手动改写规范。确认文档内容与当前生命周期一致；仍待 archive 的 delta 不能被称为已经同步到现行基线。

只有相关验收通过且满足项目 archive 门禁时才建议归档。归档、迁移和删除旧产物各自需要明确授权；不要自动把已有 change 转换成轻量 Change。
