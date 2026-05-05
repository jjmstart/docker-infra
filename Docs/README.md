# /opt/docker-infra/Docs — 文档中心

本目录是集群文档的统一入口，按用途分为四个子文件夹。V1.4 之后，`/opt/docker-infra` 是 Git/Ansible 控制仓库，Docs 跟随 Git 工作流管理：草案用于讨论，runbook 用于执行，architecture 记录已落地状态，retrospective 用于复盘。

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

---

## 如何选择要读哪份文档

| 场景 | 应读的文件夹 |
|------|-------------|
| 我想讨论下一次架构升级，还没有开始实施 | [proposals/](proposals/) → 读对应草案（如 `v1.5-alerting.md`） |
| 我想快速了解当前集群是什么样的（节点、服务、IP、拓扑） | [architecture/](architecture/) → 读 `v1.5.md`（最新版） |
| 我要开始执行某次架构演进，需要逐步操作指引 | [runbooks/](runbooks/) → 读对应版本手册（如 `v1.3-to-v1.4.md`） |
| 演进刚完成，我想复盘做了什么、踩了什么坑、加深理解 | [retrospectives/](retrospectives/) → 读对应版本总结（如 `v1.2-retrospective.md`） |

---

## 文件索引

### proposals/

| 文件 | 目标版本 | 说明 |
|------|----------|------|
| [proposals/README.md](proposals/README.md) | — | 架构设计草案索引 |
| [proposals/v1.5-alerting.md](proposals/v1.5-alerting.md) | V1.5 | 告警系统设计草案（已落地，最终状态见 `architecture/v1.5.md`） |
| [proposals/v1.6-pipeline.md](proposals/v1.6-pipeline.md) | V1.6 | 应用交付流水线设计草案（已落地，最终状态见 `architecture/v1.6.md`） |
| [proposals/v1.7-backup.md](proposals/v1.7-backup.md) | V1.7 | 备份恢复最小闭环设计草案（讨论中） |

### architecture/

| 文件 | 版本 | 说明 |
|------|------|------|
| [architecture/README.md](architecture/README.md) | — | 各版本说明索引 |
| [architecture/v1.0.md](architecture/v1.0.md) | V1.0 | 初始拓扑，监控位于 bj-01（历史归档） |
| [architecture/v1.1.md](architecture/v1.1.md) | V1.1 | 监控迁至 gz-01，补齐 Node Exporter（历史归档） |
| [architecture/v1.2.md](architecture/v1.2.md) | V1.2 | MySQL 一主两从 + ProxySQL 读写分离（历史归档） |
| [architecture/v1.3.md](architecture/v1.3.md) | V1.3 | `/opt/docker` 纳入 Git + GitHub Private 管理（历史归档） |
| [architecture/v1.4.md](architecture/v1.4.md) | V1.4 | Ansible + Jenkins CI/CD，git push 全自动下发配置（历史归档） |
| [architecture/v1.5.md](architecture/v1.5.md) | V1.5 | Prometheus + Alertmanager + blackbox-exporter + 飞书通知告警闭环（历史归档） |
| [architecture/v1.6.md](architecture/v1.6.md) | V1.6 | 私有 Docker Registry + ruoyi CI 自动触发 + 参数化 CD + Smoke Test + Registry GC + 飞书双机器人（**当前最新**） |

### runbooks/

| 文件 | 演进路径 | 说明 |
|------|----------|------|
| [runbooks/README.md](runbooks/README.md) | — | 手册列表索引 |
| [runbooks/v1.0-to-v1.1.md](runbooks/v1.0-to-v1.1.md) | V1.0 → V1.1 | 监控栈迁移操作手册 |
| [runbooks/v1.1-to-v1.2.md](runbooks/v1.1-to-v1.2.md) | V1.1 → V1.2 | MySQL 高可用升级操作手册（6 个 Phase） |
| [runbooks/v1.2-to-v1.3.md](runbooks/v1.2-to-v1.3.md) | V1.2 → V1.3 | Git + GitHub 管理集群配置操作手册（Phase 6 未执行，已由 v1.4 取代） |
| [runbooks/v1.3-to-v1.4.md](runbooks/v1.3-to-v1.4.md) | V1.3 → V1.4 | Ansible + Jenkins CI/CD 全自动配置管理操作手册（9 个 Phase） |
| [runbooks/v1.4-to-v1.5.md](runbooks/v1.4-to-v1.5.md) | V1.4 → V1.5 | Prometheus + Alertmanager + blackbox-exporter + 飞书通知告警系统操作手册（8 个 Phase） |
| [runbooks/v1.5-to-v1.6.md](runbooks/v1.5-to-v1.6.md) | V1.5 → V1.6 | 私有 Registry + ruoyi CI/CD 全链路操作手册 |
| [runbooks/v1.6-to-v1.7.md](runbooks/v1.6-to-v1.7.md) | V1.6 → V1.7 | MySQL 逻辑备份 + OSS 上传 + bj-01 恢复演练操作手册 |

### retrospectives/

| 文件 | 版本 | 说明 |
|------|------|------|
| [retrospectives/README.md](retrospectives/README.md) | — | 总结列表索引 |
| [retrospectives/v1.1-retrospective.md](retrospectives/v1.1-retrospective.md) | V1.1 | 监控迁移复盘 |
| [retrospectives/v1.2-retrospective.md](retrospectives/v1.2-retrospective.md) | V1.2 | MySQL 高可用升级复盘（含故障演练 RTO 实测） |
| [retrospectives/v1.4-retrospective.md](retrospectives/v1.4-retrospective.md) | V1.4 | Ansible + Jenkins CI/CD 统一配置管理复盘 |
| [retrospectives/v1.5-retrospective.md](retrospectives/v1.5-retrospective.md) | V1.5 | 告警系统闭环与故障演练复盘 |
| [retrospectives/v1.6-retrospective.md](retrospectives/v1.6-retrospective.md) | V1.6 | 应用交付流水线全链路复盘 |

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

| 文件 | 说明 |
|------|------|
| [集群架构-V1.0.md](集群架构-V1.0.md) | 原始架构文档 V1.0 |
| [集群架构-V1.1.md](集群架构-V1.1.md) | 原始架构文档 V1.1 |
| [集群架构-V1.2.md](集群架构-V1.2.md) | 原始架构文档 V1.2 |
| [系统初始化指令-V15.0-MySQL高可用.md](系统初始化指令-V15.0-MySQL高可用.md) | MySQL 高可用专项系统指令 V15.0 |
