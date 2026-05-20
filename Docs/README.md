# /opt/docker-infra/Docs — 文档中心

本目录是集群文档的统一入口，按用途分为五个子文件夹。V1.4 之后，`/opt/docker-infra` 是 Git/Ansible 控制仓库，Docs 跟随 Git 工作流管理：草案用于讨论，runbook 用于执行，architecture 记录已落地状态，retrospective 用于复盘，reviews 用于跨版本主题型横向复盘。仓库根目录另有 [`CHANGELOG.md`](../CHANGELOG.md)，按 Keep a Changelog 格式聚合每个 `arch-vX.Y` tag 之间的变更，作为"版本级聚合"层与 commit 互补。

---

## 文件夹说明

### [proposals/](proposals/) — 架构设计草案

每次架构演进落地前的一份设计草案，记录目标、方案、影响范围、风险与验收标准。草案可反复修改，不代表当前线上状态。

### [architecture/](architecture/) — 集群架构快照

每个架构版本一份文档，记录该版本集群的**完整静态状态**：节点角色、服务部署、网络拓扑、技术决策。将任意版本文件提供给 AI，即可在无需其他上下文的情况下完整理解当时的集群状态。适合作为新会话的"上下文启动文件"。

### [runbooks/](runbooks/) — 操作手册

每次架构演进一份文档，记录本次演进的**完整操作流程**：演进目标、前置条件、分步命令、预期输出、验证方法与回滚方式。将对应手册提供给 AI，即可进入操作协助模式，逐步引导完成演进。

### [retrospectives/](retrospectives/) — 演进总结

每次演进完成后的**复盘文档**：演进成果、技术决策背后的原因（Q/A 形式）、踩坑记录与操作心得。适合在演进完成后回顾，或作为面试复盘材料。

### [reviews/](reviews/) — 主题型横向复盘

跨版本、按主题展开的**深度审计文档**：识别架构演进过程中遗留的盲点（如 IaC 完整性、HA 状态边界、CI/CD 韧性），分析根因，给出修复排期。与 `retrospectives/` 按版本组织不同，本目录按主题组织，触发时机是"发现盲点"而非"完成升级"。

### [drills/](drills/) — 故障演练记录

主动验证已有能力的**故障演练记录**。每份记录"演练目标 → 前置基线 → 演练步骤 → 实测观察 → 心得 → 待改进项"完整闭环。与 `reviews/` 识别理论盲点不同，本目录用线上实操验证实际表现（告警链路时延、故障摘除自动性、恢复操作可重复性），不绑定 Git tag。

### [narratives/](narratives/) — 面试叙事素材集

从 `retrospectives/` / `reviews/` / `drills/` 中抽取的**可面试叙事切片**。每个架构版本一份文件，按"触发点 → 发现的问题 → 已解决 → 决策分析 → 面试一句话版 → 可追问 Q&A → 可迁移的工程原则"七要素组织。不引入新事实，只做"面向面试现场表达"的重写。作为 `scheme/phase-2-job-hunting.md` 任务 1 的录音自述清单使用。

### [sli-slo.md](sli-slo.md) — SLI/SLO 契约

集群对外承诺哪些可靠性指标、用什么指标衡量、违约时如何兜底的**运营层契约文档**。月初回填实测样本，季度评估 SLO 目标值。横向运营契约，跨架构版本稳定。

---

## 如何选择要读哪份文档


| 场景                           | 应读的文件夹                                                                  |
| ---------------------------- | ----------------------------------------------------------------------- |
| 我想讨论下一次架构升级，还没有开始实施          | [proposals/](proposals/) → 读对应草案（如 `v1.5-alerting.md`）                  |
| 我想快速了解当前集群是什么样的（节点、服务、IP、拓扑） | [architecture/](architecture/) → 读 `v1.7.md`（最新版）                       |
| 我要开始执行某次架构演进，需要逐步操作指引        | [runbooks/](runbooks/) → 读对应版本手册（如 `v1.3-to-v1.4.md`）                   |
| 演进刚完成，我想复盘做了什么、踩了什么坑、加深理解    | [retrospectives/](retrospectives/) → 读对应版本总结（如 `v1.2-retrospective.md`） |
| 我想深入理解当前架构有哪些跨版本的隐性盲点         | [reviews/](reviews/) → 读对应主题审计（如 `v1.7-iac-completeness-audit.md`）      |
| 我想看某次故障演练的实测数据（告警时延、摘除耗时等）       | [drills/](drills/) → 读对应演练记录（如 `v1.7-gz02-app-failure.md`）             |
| 我准备面试，想拿到某个版本的"叙事点速查 + 一句话版 + 可追问 Q&A" | [narratives/](narratives/) → 读对应版本叙事集（如 `v1.7.md`）          |
| 我想了解集群对外承诺什么可靠性指标、当前实测多少          | [sli-slo.md](sli-slo.md) → 单文件运营契约                                   |
| 我想快速扫读上一个 tag 到当前 HEAD 之间发生了什么 | [CHANGELOG.md](../CHANGELOG.md) → Unreleased 段 / 历史版本段                |


---

## 文件索引

### proposals/


| 文件                                                       | 目标版本 | 说明                                            |
| -------------------------------------------------------- | ---- | --------------------------------------------- |
| [proposals/README.md](proposals/README.md)               | —    | 架构设计草案索引                                      |
| [proposals/v1.5-alerting.md](proposals/v1.5-alerting.md) | V1.5 | 告警系统设计草案（已落地，最终状态见 `architecture/v1.5.md`）    |
| [proposals/v1.6-pipeline.md](proposals/v1.6-pipeline.md) | V1.6 | 应用交付流水线设计草案（已落地，最终状态见 `architecture/v1.6.md`） |
| [proposals/v1.7-backup.md](proposals/v1.7-backup.md)     | V1.7 | 备份恢复最小闭环设计草案（已落地，最终状态见 `architecture/v1.7.md`）  |


### architecture/


| 文件                                               | 版本   | 说明                                                                                        |
| ------------------------------------------------ | ---- | ----------------------------------------------------------------------------------------- |
| [architecture/README.md](architecture/README.md) | —    | 各版本说明索引                                                                                   |
| [architecture/v1.0.md](architecture/v1.0.md)     | V1.0 | 初始拓扑，监控位于 bj-01（历史归档）                                                                     |
| [architecture/v1.1.md](architecture/v1.1.md)     | V1.1 | 监控迁至 gz-01，补齐 Node Exporter（历史归档）                                                         |
| [architecture/v1.2.md](architecture/v1.2.md)     | V1.2 | MySQL 一主两从 + ProxySQL 读写分离（历史归档）                                                          |
| [architecture/v1.3.md](architecture/v1.3.md)     | V1.3 | `/opt/docker` 纳入 Git + GitHub Private 管理（历史归档）                                            |
| [architecture/v1.4.md](architecture/v1.4.md)     | V1.4 | Ansible + Jenkins CI/CD，git push 全自动下发配置（历史归档）                                            |
| [architecture/v1.5.md](architecture/v1.5.md)     | V1.5 | Prometheus + Alertmanager + blackbox-exporter + 飞书通知告警闭环（历史归档）                            |
| [architecture/v1.6.md](architecture/v1.6.md)     | V1.6 | 私有 Docker Registry + ruoyi CI 自动触发 + 参数化 CD + Smoke Test + Registry GC + 飞书双机器人（历史归档） |
| [architecture/v1.7.md](architecture/v1.7.md)     | V1.7 | mysqldump 逻辑备份 + 阿里云 OSS 上传 + bj-01 恢复演练，实测 RTO 34 秒（**当前最新**） |


### runbooks/


| 文件                                                   | 演进路径        | 说明                                                                      |
| ---------------------------------------------------- | ----------- | ----------------------------------------------------------------------- |
| [runbooks/README.md](runbooks/README.md)             | —           | 手册列表索引                                                                  |
| [runbooks/v1.0-to-v1.1.md](runbooks/v1.0-to-v1.1.md) | V1.0 → V1.1 | 监控栈迁移操作手册                                                               |
| [runbooks/v1.1-to-v1.2.md](runbooks/v1.1-to-v1.2.md) | V1.1 → V1.2 | MySQL 高可用升级操作手册（6 个 Phase）                                              |
| [runbooks/v1.2-to-v1.3.md](runbooks/v1.2-to-v1.3.md) | V1.2 → V1.3 | Git + GitHub 管理集群配置操作手册（Phase 6 未执行，已由 v1.4 取代）                         |
| [runbooks/v1.3-to-v1.4.md](runbooks/v1.3-to-v1.4.md) | V1.3 → V1.4 | Ansible + Jenkins CI/CD 全自动配置管理操作手册（9 个 Phase）                          |
| [runbooks/v1.4-to-v1.5.md](runbooks/v1.4-to-v1.5.md) | V1.4 → V1.5 | Prometheus + Alertmanager + blackbox-exporter + 飞书通知告警系统操作手册（8 个 Phase） |
| [runbooks/v1.5-to-v1.6.md](runbooks/v1.5-to-v1.6.md) | V1.5 → V1.6 | 私有 Registry + ruoyi CI/CD 全链路操作手册                                       |
| [runbooks/v1.6-to-v1.7.md](runbooks/v1.6-to-v1.7.md) | V1.6 → V1.7 | MySQL 逻辑备份 + OSS 上传 + bj-01 恢复演练操作手册                                    |


### retrospectives/


| 文件                                                                           | 版本   | 说明                               |
| ---------------------------------------------------------------------------- | ---- | -------------------------------- |
| [retrospectives/README.md](retrospectives/README.md)                         | —    | 总结列表索引                           |
| [retrospectives/v1.1-retrospective.md](retrospectives/v1.1-retrospective.md) | V1.1 | 监控迁移复盘                           |
| [retrospectives/v1.2-retrospective.md](retrospectives/v1.2-retrospective.md) | V1.2 | MySQL 高可用升级复盘（含故障演练 RTO 实测）      |
| [retrospectives/v1.4-retrospective.md](retrospectives/v1.4-retrospective.md) | V1.4 | Ansible + Jenkins CI/CD 统一配置管理复盘 |
| [retrospectives/v1.5-retrospective.md](retrospectives/v1.5-retrospective.md) | V1.5 | 告警系统闭环与故障演练复盘                    |
| [retrospectives/v1.6-retrospective.md](retrospectives/v1.6-retrospective.md) | V1.6 | 应用交付流水线全链路复盘                     |
| [retrospectives/v1.7-retrospective.md](retrospectives/v1.7-retrospective.md) | V1.7 | 备份恢复闭环复盘（含 6 条踩坑，实测 RTO 34 秒）  |


### reviews/


| 文件                                                                                 | 触发版本 | 说明                                                          |
| ---------------------------------------------------------------------------------- | ---- | ----------------------------------------------------------- |
| [reviews/README.md](reviews/README.md)                                             | —    | 主题型横向复盘索引与撰写规范                                              |
| [reviews/v1.7-iac-completeness-audit.md](reviews/v1.7-iac-completeness-audit.md)   | V1.7 | IaC 完整性审计：V1.7 → V1.8 准备期识别并处理的 21 个集群可复现性问题（10 个已修 + 11 个排进 v1.8-v1.10） |
| [reviews/v1.7-iac-ci-quality-gate-gap.md](reviews/v1.7-iac-ci-quality-gate-gap.md) | V1.7 | CI 链路缺少 IaC 自身质量门禁：Jenkins pre-merge 阶段补 `ansible-lint` / `gitleaks` / vault 加密状态校验 |


### drills/


| 文件                                                                       | 演练时架构版本 | 演练范围                                                          | 状态 |
| ------------------------------------------------------------------------ | ------- | ------------------------------------------------------------- | -- |
| [drills/README.md](drills/README.md)                                     | —       | 演练子体系索引与撰写规范                                                  | —  |
| [drills/v1.7-gz02-app-failure.md](drills/v1.7-gz02-app-failure.md)       | V1.7    | gz-02 ruoyi-admin-2 故障下 Nginx upstream 摘除、blackbox 探测延迟、告警链路时延 | 计划中 |


### narratives/


| 文件                                           | 版本   | 叙事点数量 | 备注                                            |
| -------------------------------------------- | ---- | ----- | --------------------------------------------- |
| [narratives/README.md](narratives/README.md) | —    | —     | 面试叙事素材集索引与撰写规范                                |
| [narratives/v1.7.md](narratives/v1.7.md)     | V1.7 | 3     | IaC 完整性 / CI 质量门禁 / 网关耦合演进时机（含一句话版、Q&A、工程原则） |


### 单文件文档


| 文件                         | 说明                                                              |
| -------------------------- | --------------------------------------------------------------- |
| [sli-slo.md](sli-slo.md)   | 集群 SLI/SLO 契约：服务清单、SLI 定义、SLO 目标、错误预算、月度实测样本与违约响应等级 |


---

## 新增版本 / 文档撰写

新增架构版本时遵循 `.cursor/rules/10-docs-workflow.mdc`。该规则是文档撰写规范的唯一来源，AI 应依据它完成新版本所有文档的撰写，并同步更新相关索引文件。

V1.4 之后，每次架构升级按 Git 发布流程管理：

1. 新建设计草案：`Docs/proposals/vX.Z-topic.md`
2. 新建或更新 Ansible 配置：如 `roles/xxx/`、`inventory/`、`playbooks/`
3. 新建升级手册：`Docs/runbooks/vX.Y-to-vX.Z.md`
4. Jenkins / Ansible 部署验证通过
5. 回查并修正 runbook 中与实际执行不一致的内容
6. 新建正式架构快照：`Docs/architecture/vX.Z.md`
7. 新建复盘：`Docs/retrospectives/vX.Z-retrospective.md`
8. 更新 README 索引
9. 打 tag：`arch-vX.Z`

---

## 历史归档文件

以下文件为原始文档，已归档，内容保留供参考，请以上方重整后的文档为准。


| 文件                                                     | 说明                    |
| ------------------------------------------------------ | --------------------- |
| [集群架构-V1.0.md](集群架构-V1.0.md)                           | 原始架构文档 V1.0           |
| [集群架构-V1.1.md](集群架构-V1.1.md)                           | 原始架构文档 V1.1           |
| [集群架构-V1.2.md](集群架构-V1.2.md)                           | 原始架构文档 V1.2           |
| [系统初始化指令-V15.0-MySQL高可用.md](系统初始化指令-V15.0-MySQL高可用.md) | MySQL 高可用专项系统指令 V15.0 |
