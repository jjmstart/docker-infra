# proposals/ — 架构设计草案

本目录用于存放尚未落地的架构设计草案。草案用于讨论目标、方案、影响范围、风险与验收标准；只有在 Jenkins / Ansible 部署验证通过后，才能沉淀为正式的 `architecture/` 架构快照。

## 草案索引

| 文件 | 目标版本 | 主题 | 状态 |
|------|----------|------|------|
| [v1.5-alerting.md](v1.5-alerting.md) | V1.5 | 告警系统：Prometheus + Alertmanager + 手机通知 | 草案设计中 |

## 使用约定

- 草案可以反复修改，不代表当前线上状态。
- 草案中不得记录敏感信息、明文密码、通知渠道 token 或 webhook secret。
- 草案通过评审后，需要先新增或完善对应的 `runbooks/`；部署验证通过后，再新增 `architecture/` 和 `retrospectives/` 文档。
- 正式架构状态以 `architecture/`、Git commit 和 `arch-vX.Y` tag 为准。
