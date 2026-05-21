# drills/ — 故障演练记录

本目录沉淀**主动验证已有能力**的故障演练。每次演练对应一份独立文档，记录"演练目标 → 前置基线 → 演练步骤 → 实测观察 → 心得 → 待改进项"完整闭环。子体系定位、与 reviews / runbooks / retrospectives / sli-slo 的边界、撰写规范、章节结构、触发时机判断与写作约束，统一见 [`.cursor/rules/10-docs-workflow.mdc`](../../.cursor/rules/10-docs-workflow.mdc) §7。

## 索引

| 文件 | 演练时架构版本 | 演练范围 | 状态 |
|------|---------------|---------|------|
| [v1.7-gz02-app-failure.md](v1.7-gz02-app-failure.md) | V1.7 | gz-02 ruoyi-admin-2 故障下 Nginx upstream 摘除、blackbox 探测延迟、告警链路时延 | 计划中 |
