# reviews/ — 主题型横向复盘

本目录沉淀**跨版本、按主题**展开的深度复盘。触发时机不是"完成某次架构升级"，而是"在阅读代码 / 排障 / 评审过程中识别到一个之前没意识到的盲点"。撰写规范、与 retrospectives / proposals 的边界、单篇文档结构与写作约束，统一见 [`.cursor/rules/10-docs-workflow.mdc`](../../.cursor/rules/10-docs-workflow.mdc) §6。

## 索引

| 文件 | 触发版本 | 主题 | 状态 |
|------|----------|------|------|
| [v1.7-iac-completeness-audit.md](v1.7-iac-completeness-audit.md) | V1.7 | IaC 完整性审计：V1.7 → V1.8 准备期识别并处理的 21 个集群可复现性问题（10 个已修 + 11 个排进 v1.8-v1.10） | 部分已修 + 剩余已规划 |
| [v1.7-iac-ci-quality-gate-gap.md](v1.7-iac-ci-quality-gate-gap.md) | V1.7 | CI 链路缺少 IaC 自身质量门禁：`ansible-lint` / `yamllint` / `gitleaks` / vault 加密状态校验作为 Jenkins pre-merge stage | 已识别，待修复 |
| [v1.7-ingress-probe-functional-depth-gap.md](v1.7-ingress-probe-functional-depth-gap.md) | V1.7 | 入口探测的功能深度盲区：从一次 Ruoyi 后端 5 天无人感知的故障倒推 6 个 blackbox 探测目标的功能覆盖，三类盲区按 ROI 分级排进 v1.8/v1.9/v2.0 | 部分已修 + 剩余已规划 |
