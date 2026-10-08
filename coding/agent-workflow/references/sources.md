# 来源与本地取舍

上游：[mattpocock/skills](https://github.com/mattpocock/skills/tree/f3fc5632f401156837ee3872f14fe33ccf1024ea)，参考修订 `f3fc5632f401156837ee3872f14fe33ccf1024ea`。保留的上游许可见 [MIT 许可](UPSTREAM-LICENSE)。

| 本地入口或支持 skill | 上游来源 | 保留方法 | 本地改造 |
| --- | --- | --- | --- |
| change-planning | [to-spec](https://github.com/mattpocock/skills/blob/f3fc5632f401156837ee3872f14fe33ccf1024ea/skills/engineering/to-spec/SKILL.md)、[to-tickets](https://github.com/mattpocock/skills/blob/f3fc5632f401156837ee3872f14fe33ccf1024ea/skills/engineering/to-tickets/SKILL.md) | 合成讨论、行为与测试决定、纵向切片、真实依赖 | 按交付目标生成多份 Change，任务默认内嵌，无 tracker 硬依赖，不强制长篇 user stories |
| agent-workflow | [implement](https://github.com/mattpocock/skills/blob/f3fc5632f401156837ee3872f14fe33ccf1024ea/skills/engineering/implement/SKILL.md)、[implement-spec](https://github.com/mattpocock/skills/blob/f3fc5632f401156837ee3872f14fe33ccf1024ea/skills/engineering/implement-spec/SKILL.md) | TDD、审查、任务依赖、上下文指针、隔离集成 | 主 agent 验收与同步；不默认最大并发、PR、reset 或清理 |
| tdd | [tdd](https://github.com/mattpocock/skills/blob/f3fc5632f401156837ee3872f14fe33ccf1024ea/skills/engineering/tdd/SKILL.md) 及 tests/mocking | 公开行为接口、有效 red/green、独立预期、避免批量横向实施 | 批准接口后不逐测试审批，证据分层，非行为检查和最终门禁分工 |
| code-review | [code-review](https://github.com/mattpocock/skills/blob/f3fc5632f401156837ee3872f14fe33ccf1024ea/skills/engineering/code-review/SKILL.md) | Spec/Standards 双轴、原始来源、质量启发式 | 默认一个独立只读 reviewer；覆盖未提交和新增文件；审核验证证据与处置类型 |
| codebase-design | [codebase-design](https://github.com/mattpocock/skills/blob/f3fc5632f401156837ee3872f14fe33ccf1024ea/skills/engineering/codebase-design/SKILL.md) 及配套参考 | 深模块、完整接口、变化集中、依赖与替换位置 | 服从项目语言；多方案委派需授权；旧测试先核对覆盖再移除 |
| domain-modeling | [domain-modeling](https://github.com/mattpocock/skills/blob/f3fc5632f401156837ee3872f14fe33ccf1024ea/skills/engineering/domain-modeling/SKILL.md) 及 ADR/Glossary 格式 | 术语澄清、具体场景、按需 ADR、一段式理由 | 主 agent 控制文档写入；候选不冒充已交付状态 |
| grilling | [grilling](https://github.com/mattpocock/skills/blob/f3fc5632f401156837ee3872f14fe33ccf1024ea/skills/productivity/grilling/SKILL.md) | 决策树、依赖已明确的问题轮次、推荐答案 | 只问影响当前范围的未决问题，查事实默认直接执行，委派需授权 |

## 用户定制的规则

长期 Spec/Design 加按需 ADR、Change 的增量同步、draft/approved/accepted 状态、两个显式入口、主 agent 收尾职责，以及技能路径预检，来自本地工作流设计，不是上游 Matt skills 的统一约定。

「文档精简」指减少重复内容和默认加载范围，不是省略重要契约。共享文档规则只在 change-planning 的文档契约参考维护，执行入口引用它。所有选定 skills 保存在同一 coding 集合中，运行不依赖上游仓库的本机路径。

执行机制依赖当前安装的 pi-subagents 指引，模型、插件设置和安装配置是部署策略。本集合不修改旧 team-workflow，不提供 OpenSpec 接入，不自动安装或配置上游 tracker。
