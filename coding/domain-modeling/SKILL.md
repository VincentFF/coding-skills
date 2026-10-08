---
name: domain-modeling
description: 澄清项目特有概念和重要决定，为主 agent 提供术语及 ADR 方法。用于术语冲突、领域关系讨论或 ADR 创建和修订。
license: MIT
---

# Domain Modeling

统一概念和记录重要决定，不把 Glossary 当 Spec、Design 或任务容器。只是消费现有术语时无需启动建模过程。

## 澄清概念

读取相关 Glossary；存在 GLOSSARY-MAP 时只沿相关上下文读取。若没有文件正常继续，不要求提前建目录或文件。

用户用语与现有定义冲突时指出不同含义；模糊名称提出精确候选，使用具体场景检验概念、关系和所有权。代码用于核实当前事实，与目标不一致时提出冲突而不是自动选一方。

确认规范名称后按 [Glossary 格式](references/glossary-format.md) 简短记录。仅收项目特有概念，不收入普通编程术语、实现步骤或配置清单。

## ADR 候选

只有真实决定同时具备以下性质时默认提出 ADR：难以逆转；没有背景会让未来读者困惑；比较过合理替代方案。项目强制记录的决定遵守项目规则，其他实现细节不为数量建 ADR。

背景、决定、理由通常一段足够；按 [ADR 格式](references/adr-format.md) 增加确实有价值的替代方案、非显然后果或替代关系。需要重开既有 ADR 时明确指出冲突，交用户确认，不悄悄覆盖原理由。

## 写入职责

只讨论时返回建议，不自动写长期文档。change-planning 阶段将确认的术语变化和 ADR 候选记录在 Change；agent-workflow 收尾由主 agent 按 [文档契约](../change-planning/references/document-contract.md) 持久化完成的变化。

用户明确请求独立创建或修订 Glossary/ADR 时，主 agent 可在已授权范围内写入，但只能记录真实共识。批准技术决定不等于相关能力已交付，不能在现行 Design 中冒充实现完成。

worker/reviewer 可以读取本参考，但不因此获得修改契约或长期文档的权限。来源见 [来源](../agent-workflow/references/sources.md)。
