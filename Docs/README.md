# /opt/docker/Docs — 文档中心

本目录是集群文档的统一入口，按用途分为三个子文件夹。

---

## 文件夹说明

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
| 我想快速了解当前集群是什么样的（节点、服务、IP、拓扑） | [architecture/](architecture/) → 读 `v1.2.md`（最新版） |
| 我要开始执行某次架构演进，需要逐步操作指引 | [runbooks/](runbooks/) → 读对应版本手册（如 `v1.1-to-v1.2.md`） |
| 演进刚完成，我想复盘做了什么、踩了什么坑、加深理解 | [retrospectives/](retrospectives/) → 读对应版本总结（如 `v1.2-retrospective.md`） |

---

## 文件索引

### architecture/

| 文件 | 版本 | 说明 |
|------|------|------|
| [architecture/README.md](architecture/README.md) | — | 各版本说明索引 |
| [architecture/v1.0.md](architecture/v1.0.md) | V1.0 | 初始拓扑，监控位于 bj-01（历史归档） |
| [architecture/v1.1.md](architecture/v1.1.md) | V1.1 | 监控迁至 gz-01，补齐 Node Exporter（历史归档） |
| [architecture/v1.2.md](architecture/v1.2.md) | V1.2 | MySQL 一主两从 + ProxySQL 读写分离（历史归档） |
| [architecture/v1.3.md](architecture/v1.3.md) | V1.3 | `/opt/docker` 纳入 Git + GitHub Private 管理（**当前最新**） |

### runbooks/

| 文件 | 演进路径 | 说明 |
|------|----------|------|
| [runbooks/README.md](runbooks/README.md) | — | 手册列表索引 |
| [runbooks/v1.0-to-v1.1.md](runbooks/v1.0-to-v1.1.md) | V1.0 → V1.1 | 监控栈迁移操作手册 |
| [runbooks/v1.1-to-v1.2.md](runbooks/v1.1-to-v1.2.md) | V1.1 → V1.2 | MySQL 高可用升级操作手册（6 个 Phase） |
| [runbooks/v1.2-to-v1.3.md](runbooks/v1.2-to-v1.3.md) | V1.2 → V1.3 | Git + GitHub 管理集群配置操作手册（6 个 Phase） |

### retrospectives/

| 文件 | 版本 | 说明 |
|------|------|------|
| [retrospectives/README.md](retrospectives/README.md) | — | 总结列表索引 |
| [retrospectives/v1.1-retrospective.md](retrospectives/v1.1-retrospective.md) | V1.1 | 监控迁移复盘 |
| [retrospectives/v1.2-retrospective.md](retrospectives/v1.2-retrospective.md) | V1.2 | MySQL 高可用升级复盘（含故障演练 RTO 实测） |

---

## 新增版本 / 文档撰写

将 [文档撰写规范.md](文档撰写规范.md) 提供给 AI，即可让 AI 独立完成新版本所有文档的撰写，并知道需要同步更新哪些已有文件。

---

## 历史归档文件

以下文件为原始文档，已归档，内容保留供参考，请以上方重整后的文档为准。

| 文件 | 说明 |
|------|------|
| [集群架构-V1.0.md](集群架构-V1.0.md) | 原始架构文档 V1.0 |
| [集群架构-V1.1.md](集群架构-V1.1.md) | 原始架构文档 V1.1 |
| [集群架构-V1.2.md](集群架构-V1.2.md) | 原始架构文档 V1.2 |
| [系统初始化指令-V15.0-MySQL高可用.md](系统初始化指令-V15.0-MySQL高可用.md) | MySQL 高可用专项系统指令 V15.0 |
