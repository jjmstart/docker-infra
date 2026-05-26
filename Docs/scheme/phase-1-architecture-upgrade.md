# 阶段一：架构升级（v1.5 → v2.0）

## 阶段目标

将 `docker-infra` 集群从 v1.5 路线起点延伸至 v2.0，当前已落地到 v1.7。每个子任务对应一个架构版本，每版本完成后需按文档规范完成 proposal → runbook → retrospective → 架构快照的完整流程。

**当前起点**：v1.7 已完成备份恢复最小闭环（gz-03 mysqldump 逻辑备份 + 阿里云 OSS 上传 + bj-01 恢复演练，实测 RTO 34 秒），已通过端到端验收，Git tag `arch-v1.7` 已推送。

**路线调整**：v1.7 全量复盘中发现，当前 Ansible 已能稳定管理 Compose 配置和服务启动，但仍存在若干"历史一次性操作未纳入 IaC"的问题，例如 `mysql_source_delay` 已写入 inventory 但复制通道初始化仍依赖历史手工状态。为避免在扩节点 / K3s 迁移前继续放大这些隐性状态，v1.9-v1.12 先补齐架构可复现性与 HA 状态管理，原六节点接入及后续 K3s 路线顺延一格。

**v1.10 范围调整**：原"HA 配置状态修复"主题混合了 ProxySQL 配置 runtime 同步与 Redis Sentinel 边界修复两件落地动作完全不同的事，叙事不聚焦。结合 v1.7 IaC 审计中识别的 ProxySQL 进程级 SPOF（gz-03 整机宕机即应用读写全断），v1.10 重新定义为"**ProxySQL HA 化**"，把"双实例 Cluster + Admin SQL runtime 同步 + 应用 JDBC 多端点"合成一个完整的 ProxySQL HA 闭环；原方案中的 Redis Sentinel 边界修复（review #5 #6）独立成 v1.11。后续 CI/CD 状态 IaC 化等版本号顺延一格。

**v1.8 新增**：v1.7 复盘期间识别到告警链路出口端单点（Alertmanager / prometheus-alert / bj-01 整机故障时告警链路无替代路径），原 v1.13 watchdog 设计仅覆盖采集端而未覆盖出口端。v1.8 独立成 "Dead Man's Switch（DMS）告警出口冗余" 任务，通过 Prometheus 永真心跳规则 + Alertmanager 路由到外部 SaaS（与自建集群物理隔离）覆盖出口 / 链路 / 送达三层失效域；原 v1.8 起所有版本号顺延一格，watchdog 在顺延后的 v1.14 保留并显式声明仅覆盖采集端边界。详见 `Docs/reviews/v1.7-alerting-pipeline-spof-audit.md`。

---

## 子任务完成状态

> **跨文档引用约定**：reviews / narratives / 新建 proposals / sli-slo / inventory 注释 / CHANGELOG 等所有 live 文档引用本路线主题时，统一使用 slug 锚点（例如 `[ProxySQL HA 主题](Docs/scheme/phase-1-architecture-upgrade.md#proxysql-ha)`），不写 v1.X 硬编码。slug 一旦写入下表视为永久稳定标识；目标版本号在路线调整（顺延 / 拆分 / 主题重新定义）时仅修改本表"当前目标版本"列，引用 slug 的 live 文档无需联动回写。详细规范见 `.cursor/rules/10-docs-workflow.mdc` §11 路线主题引用规范。

| slug | 子任务 | 当前目标版本 | 状态 |
|---|---|---|---|
| `alerting-mvp` | 告警系统 | v1.5 | ✅ 已完成 |
| `app-cd-pipeline` | 应用交付流水线（镜像构建推送） | v1.6 | ✅ 已完成 |
| `backup-restore-mvp` | 备份恢复最小闭环 | v1.7 | ✅ 已完成 |
| `dms-outlet-redundancy` | Dead Man's Switch（告警出口冗余） | v1.8 | 待开始 |
| `base-reproducibility-fix` | 基础可复现性修复（Docker 网络 + MySQL 数据层 bootstrap） | v1.9 | 待开始 |
| `proxysql-ha` | ProxySQL HA 化（Cluster + runtime 同步 + 应用多端点） | v1.10 | 待开始 |
| `redis-sentinel-boundary` | Redis Sentinel 边界修复（运行时拓扑 vs Ansible 静态配置） | v1.11 | 待开始 |
| `cicd-state-iac` | CI/CD 状态 IaC 化（Jenkins 镜像、Job、Credentials 备份边界） | v1.12 | 待开始 |
| `six-node-onboarding` | 六节点基础接入（gz-04/gz-05 Tailscale + Docker + 监控纳管） | v1.13 | 待开始 |
| `service-migration` | 服务迁移（gz-01/gz-02 角色重整 + watchdog） | v1.14 | 待开始 |
| `k3s-stateless` | K3s 迁移无状态服务 | v1.15 | 待开始 |
| `helm-chart-mgmt` | Helm Chart 管理 K3s 应用 | v1.16 | 待开始 |
| `loki-logging` | Loki 日志集中管理 | v1.17 | 待开始 |
| `observability-integration` | 可观测性增强（Dashboard 整合） | v2.0 | 待开始 |


---

<a id="app-cd-pipeline"></a>

## 任务 1：应用交付流水线（目标 v1.6）

> **slug**：`app-cd-pipeline` ｜ 落地于 v1.6（已完成）

### 背景

当前 v1.5 的 Jenkins Pipeline 只做配置下发（Jenkins → Ansible → 各节点），不涉及应用镜像的构建与推送。v1.6 目标是在现有流水线基础上补全"构建 → 推镜像 → 部署新版本 → 可回滚"的完整应用交付链路。

### 要完成的内容

**Jenkins Pipeline 改造**

- 在现有 `Jenkinsfile` 中区分以下 stage：
  - `Checkout`：拉取应用代码（可复用若依或另建测试服务仓库）
  - `Build`：构建 Docker 镜像，tag 格式为 `<服务名>:<git-commit-sha[:7]>`
  - `Push`：推送镜像到镜像仓库（Docker Hub 或私有 Registry）
  - `Deploy`：通过 Ansible 将新镜像版本写入 compose 并 `docker compose up -d`
  - `Rollback`（手动触发 stage）：回滚到上一个 tag

**Ansible 配合改造**

- `roles/ruoyi/` 或新建服务 role 的 `docker-compose.yml.j2` 模板中，镜像 tag 改为 Ansible 变量（如 `{{ app_image_tag }}`）
- Jenkins 通过 `-e app_image_tag=<tag>` 传参给 Ansible，实现镜像版本的精准控制

**镜像仓库**

- 配置 Jenkins Credentials 管理 Registry 凭据（不能明文写入 Jenkinsfile）
- 至少完成一次成功推送并在目标节点验证拉取

### 验收标准

- git push 后 Jenkins 自动触发，完整跑完 Checkout → Build → Push → Deploy
- 新版本镜像在 gz-03 或 gz-02 上成功替换运行
- 手动触发 Rollback stage 后，旧版本镜像恢复运行
- 能解释 Jenkinsfile 每个 stage 的作用，以及"为什么要分 stage 而不是写成一个大 shell"

### 面试叙事点

- 为什么镜像 tag 用 commit sha 而不是 latest
- Credentials 为什么不写在 Jenkinsfile 里
- 回滚的边界在哪里（镜像回滚 vs 配置回滚 vs 数据库 migration 回滚）

---

<a id="backup-restore-mvp"></a>

## 任务 2：备份恢复最小闭环（目标 v1.7）

> **slug**：`backup-restore-mvp` ｜ 落地于 v1.7（已完成）

### 背景

当前架构有 MySQL 主从（gz-03 主库、gz-02 实时从库、bj-01 延迟从库 3600s），以及 Redis 主从。但没有定期的逻辑备份和经过验证的恢复流程。v1.7 目标是建立"定期备份 → 上传远端存储 → 至少完成一次恢复演练"的闭环。

### 要完成的内容

**MySQL 逻辑备份**

- 对 gz-03 MySQL Master 配置 `mysqldump` 定时备份
- 备份脚本由 Ansible 下发，通过 cron 定时执行（每天 02:00）
- 备份文件命名包含时间戳，保留最近 3 天，自动清理旧备份

**备份上传**

- 将备份文件上传至阿里云 OSS
- 上传凭据通过 ansible-vault 管理，不明文存储

**恢复演练**

- 从 OSS 下载备份文件
- 恢复到 bj-01 本地一个临时测试库（不影响主从复制链路）
- 记录恢复耗时、数据完整性校验方法

### 验收标准

- 定时任务每天自动产生备份文件并上传
- 能从 OSS 完整恢复出一个可查询的测试库，实测 RTO 34 秒
- runbook 写清楚，能在面试中回答"你们的备份策略是什么、恢复过吗"
- Ansible 管理备份脚本，重复执行 `changed=0`

### 面试叙事点

- 为什么有主从还需要逻辑备份（主从不能防误删）
- bj-01 延迟从库 3600s 的价值（误操作后 1 小时窗口）
- RPO 和 RTO 各是多少，如何权衡

---

<a id="dms-outlet-redundancy"></a>

## 任务 3：Dead Man's Switch — 告警出口冗余（目标 v1.8）

> **slug**：`dms-outlet-redundancy` ｜ 当前目标版本：**v1.8**

### 背景

v1.7 复盘中识别到告警链路存在出口端单点：被监控对象 → Prometheus（gz-01）→ Alertmanager（bj-01:9093）→ prometheus-alert（bj-01 容器内）→ 飞书 webhook → 飞书群。bj-01 整机宕机或 Alertmanager 容器 OOM 重启失败时，整条出口链路一起失效，**Prometheus 即使评估出 firing 告警也送不出去**。

原 v1.13（顺延后 v1.14）watchdog 设计只解决"采集 / 评估端"的冗余（监控主节点宕机时仍有备份采集），watchdog 自己也是把告警发给同一个 Alertmanager，**完全没覆盖"告警出口端"单点**——即"who watches the watcher"问题在自己的设计里再嵌套了一层"who watches the watcher's mouth"。完整的告警链路按独立失效域应拆 4 段：采集 / 评估 / 出口 / 送达，watchdog 只覆盖第 1 段，2-4 段全单点。

v1.8 引入 Dead Man's Switch（DMS）反向心跳机制：Prometheus 配 `expr: vector(1)` 永真心跳规则，Alertmanager 通过独立 route 把这条永真告警以固定间隔发给外部 SaaS（与自建集群物理隔离），SaaS 在期望间隔内未收到心跳则反向告警到独立邮箱。一招覆盖出口 / 链路 / 送达三层失效域，与 watchdog 形成互补关系。详见 `Docs/reviews/v1.7-alerting-pipeline-spof-audit.md`。

### 要完成的内容

**Prometheus rule**

- `roles/monitor-stack/templates/rules/` 下新增 watchdog 永真规则（`expr: vector(1)`、`for: 0m`、固定 label 用于路由）
- 规则始终 firing，是 DMS 心跳的源头，不进入飞书机器人正常告警通道

**Alertmanager 路由**

- `roles/alertmanager/templates/alertmanager.yml.j2` 新增独立 receiver `dms-saas-webhook` 与对应 route，按 watchdog label 精确匹配
- 心跳间隔 `repeat_interval` 与外部 SaaS 期望间隔对齐（具体数值在 v1.8 proposal 阶段决定）
- DMS route 与现有飞书机器人 route 完全独立，互不影响

**外部 SaaS**

- 在 v1.8 proposal 阶段二选一：Healthchecks.io（免费档基线方案）/ Better Uptime / Dead Man's Snitch
- SaaS 反向告警接收侧使用与日常运维不同的独立邮箱，避免飞书侧故障同时屏蔽通知
- 心跳间隔与"超阈值反向告警"具体数值在 proposal 阶段评估，验收标准给出确定数值

**vault**

- `vault/secrets.yml` 新增 SaaS webhook URL（占位变量名 `dms_saas_webhook_url`，由 ansible-vault 加密；不在任何普通文件出现明文）

### 验收标准

- 手动停 Alertmanager 容器后，外部 SaaS 在心跳超时阈值（具体值见 proposal）+ 缓冲窗口内反向告警送达独立邮箱，端到端 ≤ 5 分钟
- DMS 心跳与现有飞书机器人告警通道互不干扰，飞书机器人仍按现有路由正常工作
- `roles/alertmanager/` 重复执行 `changed=0 failed=0`
- 记录首次容灾演练数据到 `Docs/drills/v1.8-dms-failover.md`（如新建）

### 面试叙事点

- watchdog vs DMS 的互补关系——watchdog 解决采集端（"who watches the watcher"），DMS 解决出口/链路/送达（"who watches the watcher's mouth"），两者按独立失效域分别覆盖，互不替代
- 为什么 DMS 优先于 Alertmanager 集群——一条 rule + 一条 route 一次覆盖 4 个失效域里的 3 个；Alertmanager Gossip 集群只覆盖 1 个失效域且要扩 role，ROI 明显劣势
- 为什么外部 SaaS 而不是自建第二套 Alertmanager——物理隔离才是关键，自建第二份依旧依赖自有节点 / 自有公网出口 / 自有飞书机器人，任意一层故障同源失效
- HA 设计要按独立失效域分别审视，不能按组件名整体处理（去本项目化的可迁移工程原则）

---

<a id="base-reproducibility-fix"></a>

## 任务 4：基础可复现性修复（目标 v1.9）

> **slug**：`base-reproducibility-fix` ｜ 当前目标版本：**v1.9**

### 背景

v1.7 全量复盘中，从 `inventory/hosts.yml` 中 `mysql_source_delay: 3600` 未被 role 实际消费这一点，暴露出当前 IaC 的边界：Ansible 已经能管理容器编排文件，但部分集群关键状态仍来自历史手工初始化，例如 Docker external network、MySQL 业务库和账号、MySQL 复制通道。v1.9 目标是补齐"新机器从零部署时必须存在的基础状态"，让 Ansible 不只会拉起容器，还能复现数据层的最低可用形态。

### 要完成的内容

**Docker 运行时基础状态**

- 在 `roles/docker-daemon/` 或独立基础设施 role 中管理 `global_gateway` 网络创建
- 确保所有需要运行 Compose 栈的节点上，`docker network ls` 可见 `global_gateway`
- 网络创建必须幂等，重复执行不报错、不产生 changed

**MySQL Master 数据层初始化**

- 在 `roles/mysql-master/` 中加入幂等 SQL 初始化任务，创建业务库与必要账号
- 至少覆盖 `ry-vue` 业务库、`repl_user` 复制账号、`mysql_exporter_user`、`proxysql_monitor_user`、`ruoyi_datasource_username`
- 所有密码继续来自 `vault/secrets.yml`，不得写入普通文件
- 使用 `CREATE USER IF NOT EXISTS`、`CREATE DATABASE IF NOT EXISTS`、`GRANT` 等方式保证重复执行安全

**MySQL Replica 状态管理**

- `roles/mysql-replica/` 负责校验已有复制通道的期望状态：`Source_Host`、`Auto_Position`、`SQL_Delay`、双线程运行状态
- `mysql_source_delay | default(0)` 成为复制通道期望状态变量：实时从库默认为 0，bj-01 延迟从库为 3600
- 普通 role 不自动执行 `RESET REPLICA ALL`、清空数据目录、重建通道；发现复制通道不存在时明确 fail，提示执行 bootstrap playbook

**一次性 Bootstrap Playbook**

- 新增受保护的 MySQL 从库 bootstrap playbook，用于新机器 / 空数据目录初始化
- 流程包括：主库一致性 dump → Tailscale 传输 → 从库导入 → `CHANGE REPLICATION SOURCE TO ... SOURCE_DELAY={{ mysql_source_delay | default(0) }}` → `START REPLICA`
- 破坏性步骤必须要求显式确认变量，例如 `mysql_replica_bootstrap_confirm=true`

### 验收标准

- 新节点执行基础 role 后，`global_gateway` 网络存在且重复执行幂等
- 新初始化的 MySQL Master 存在 `ry-vue`、复制账号、Exporter 账号、ProxySQL 监控账号和应用账号
- gz-02 从库复制通道 `SQL_Delay=0`，bj-01 从库复制通道 `SQL_Delay=3600`
- 删除从库数据目录并按 bootstrap playbook 重建后，能重新接入主库并通过 `SHOW REPLICA STATUS\G` 验证
- 普通 `setup_database.yml` 不会在已有从库上执行破坏性重建

### 面试叙事点

- 为什么 `mysql_source_delay` 不能写进 `docker-compose.yml`（它是复制通道参数，不是 mysqld 启动参数）
- 为什么把"一次性初始化"也写成 playbook（HA 和迁移的前提是可重复、可审计）
- 为什么普通 role 不自动重建复制通道（防止误接主库、误清数据目录造成更大事故）

---

<a id="proxysql-ha"></a>

## 任务 5：ProxySQL HA 化（目标 v1.10）

> **slug**：`proxysql-ha` ｜ 当前目标版本：**v1.10**

### 背景

当前 ProxySQL 仅在 gz-03 单实例运行，构成业务链路上影响面最大的 SPOF：gz-03 整机宕机时，即使 Redis Sentinel 自动选主、gz-02 ruoyi 实例还在、MySQL 主从能人工切换，应用配置的 `100.92.5.116:6033` 已不可达，读写全断，需要人工修改应用配置或 DNS 才能恢复，影响面比 MySQL 主库宕机更大。

同时 ProxySQL 的 `proxysql.cnf` 只在首次初始化 datadir 时读取，之后所有配置变更需通过 Admin Interface (`:6032`) 执行 `LOAD ... TO RUNTIME` / `SAVE ... TO DISK`；当前 `roles/proxysql/tasks/main.yml` 只做"渲染 cnf + recreate"，cnf 改动无法推到 runtime（review #7）。

这两件事本质同源：做 ProxySQL Cluster 双实例的前置就是打通 Admin SQL runtime 管理（两节点之间的配置同步走 Cluster 协议，必须经过 runtime DB），单独做 runtime 同步而不做 Cluster 没有完整闭环。v1.10 合并解决"进程级 HA"与"配置 runtime 同步"两层问题，把 ProxySQL HA 化做成一个完整的章节。

### 故障模型边界

| 场景 | v1.10 前 | v1.10 后 |
|---|---|---|
| ProxySQL 进程挂（机器还在） | 应用读写全断 | JDBC 多端点自动 failover，30s 内恢复 |
| gz-03 MySQL 进程挂、机器还在 | 写不可用 | 同 v1.10 前（**MySQL 自动 failover 不在本版本范围**） |
| gz-03 整机宕机 | ProxySQL + MySQL Master 都没了，应用全断 | 读路径：gz-02 ProxySQL 从 gz-02 MySQL Slave 读；写路径：仍需人工 MySQL 切主 |

**有意识不做的事**：不引入 MHA / Orchestrator 等 MySQL 自动 failover 工具。理由是 MySQL 自动 failover 是另一个量级的复杂度（脑裂处理、VIP 漂移、写一致性），对当前规模和初级岗叙事 ROI 不匹配；ProxySQL HA 是一个清晰可独立闭环的子问题。

### 要完成的内容

**ProxySQL 双实例 + Cluster 配置同步**

- 在 gz-02 新增 ProxySQL 实例，与 gz-03 现有实例组成 ProxySQL Cluster（无中心化 push-based 同步协议，2 节点无需 quorum）
- `roles/proxysql/templates/proxysql.cnf.j2` 新增 `admin_variables.cluster_username/cluster_password` 与 `proxysql_servers` 段，让两节点通过 Admin Interface 互相感知
- inventory `db_proxy_nodes` 分组从 `[gz-03]` 扩到 `[gz-03, gz-02]`
- `vault/secrets.yml` 新增 `proxysql_cluster_password`；`group_vars/all.yml` 新增多端点列表变量

**ProxySQL Runtime 状态同步（review #7）**

- 将 `roles/proxysql/` 从"只渲染 cnf"升级为"配置文件首次初始化 + Admin SQL 幂等同步"双路径
- 对 `mysql_servers`、`mysql_users`、`mysql_query_rules` 通过 `:6032` Admin Interface 执行幂等同步
- 每次变更后执行 `LOAD MYSQL ... TO RUNTIME` 和 `SAVE MYSQL ... TO DISK`
- 明确区分 cnf 首次初始化职责与 runtime DB 持久化职责
- Cluster 模式下任一节点的 `LOAD ... TO RUNTIME` 会自动 push 给 peer，配置无需在两节点各自重复 Admin 操作

**应用 JDBC 多端点改造**

- `roles/ruoyi/templates/.env.j2` 中 `DB_HOST/DB_PORT` 改为多端点变量（如 `DB_ENDPOINTS=100.92.5.116:6033,100.79.132.125:6033`）
- `roles/ruoyi/templates/docker-compose.yml.j2` 中 JDBC URL 协议从 `jdbc:mysql://` 升级为 `jdbc:mysql:loadbalance://` 或 `failover://`（具体协议参数和 Druid 透传方式在 PoC 阶段验证）
- 应用容器重启后 JDBC 驱动具备多端点自动 failover 能力，单 ProxySQL 实例宕机不影响整体可用性

**监控与告警补齐**

- `roles/monitor-stack/templates/prometheus.yml.j2` 新增 gz-02 ProxySQL TCP 探活 target
- 新增"ProxySQL 实例不可达"告警规则；区分"单实例不可达"（Warning）和"双实例都不可达"（Critical）两个级别

**故障演练**

- v1.10 落地后在 `Docs/drills/` 新增 `v1.10-proxysql-failover.md`：kill gz-03 ProxySQL 容器，验证 JDBC 30s 内 failover 到 gz-02 ProxySQL；恢复后验证 Cluster 配置自动收敛

### 验收标准

- gz-02 ProxySQL 容器 up；两节点 `SELECT * FROM stats_proxysql_servers_status` 互相可见
- 在任一节点 Admin Interface 修改 `mysql_servers`，另一节点 `SELECT * FROM runtime_mysql_servers` 5 秒内同步可见
- 应用使用 `jdbc:mysql:loadbalance` URL 实测读写正常
- kill gz-03 ProxySQL 容器，应用 30 秒内通过 gz-02 ProxySQL 继续读写
- 全节点重跑 proxysql role `changed=0 failed=0`

### 待确认问题（PoC 阶段验证）

- Druid 数据源是否支持透传 `jdbc:mysql:loadbalance://...`，还是需要改应用 `application.yml` 层面的连接池参数
- ProxySQL Cluster 在 datadir 已存在（即 gz-03 现有实例）的情况下首次启用 Cluster 模式的兼容性——是否需要 `proxysql_servers` 段同时写进 cnf 和 runtime DB
- 两节点 cnf 渲染差异（hostname 不同导致 `comment` 字段不同）是否会触发 Cluster 不必要的同步收敛

### 面试叙事点

- 为什么先做 ProxySQL HA 而不是 MySQL 自动 failover（复杂度量级与初级岗叙事 ROI 不匹配；ProxySQL HA 是清晰可独立闭环的子问题）
- 为什么 2 节点 ProxySQL Cluster 不需要 quorum（与 etcd/Zookeeper 的对比；ProxySQL Cluster 是无中心化的 push-based 配置同步协议，不做 leader 选举）
- 为什么 ProxySQL 配置不能只改 cnf（cnf 只在 datadir 首次初始化时被读取；runtime DB 是 SQLite，是真正的"运行时配置真相"）
- 为什么"读 HA 路径"和"写 HA 路径"分开做（范围控制；v1.10 先把读路径做到自动 failover，写路径仍依赖人工 MySQL 切主，是有意识的灰度演进节奏）

---

<a id="redis-sentinel-boundary"></a>

## 任务 6：Redis Sentinel 边界修复（目标 v1.11）

> **slug**：`redis-sentinel-boundary` ｜ 当前目标版本：**v1.11**

### 背景

v1.10 把 ProxySQL 这层"运行时状态与 IaC 协同"的边界问题做透之后，Redis 侧还有一个完全对称的盲点（review #5 #6）：

- `roles/redis-replica/templates/redis.conf.j2` 当前硬编码 `replicaof {{ redis_master_host }}`，指向 gz-03
- Sentinel 在 master 故障时会自动把某个 replica 提升为新 master，并向 conf 文件追加 `sentinel known-replica`、`current-epoch` 等运行时状态
- 当前 Ansible 流程下任何一次 redis role 重跑都会把 conf 重新渲染到"以 gz-03 为 master"的初始状态，破坏 Sentinel 已完成的故障切换结果

v1.11 目标是修复这类"故障切换后 Ansible 重跑会破坏切换结果"的边界冲突。主题与 v1.10 同属"HA 系统运行时状态与 Ansible 静态配置的协同"，但落地动作完全不同（Redis 没有 Cluster 协议，是 conf 文件 + Sentinel runtime 自治），所以从 v1.10 拆出独立成版本。

### 要完成的内容

**Redis Replica 拓扑边界**

- 梳理 `replicaof {{ redis_master_host }}` 与 Sentinel 自动 failover 的冲突边界
- 在 role 中避免故障切换后无条件把已提升的 master 重新降级为 replica
- 明确哪些状态由 Ansible 管（初始拓扑 bootstrap），哪些状态由 Sentinel 在运行时接管

**Sentinel 配置文件管理边界**

- `sentinel.conf` 通过 bind mount 挂载，Sentinel 启动后会向 conf 追加 `sentinel known-replica` / `current-epoch` 等运行时状态
- Ansible 重渲染 sentinel.conf 不能覆盖这类运行时段；拆分"初始 bootstrap" vs "运行期健康检查"两类任务
- 必要时引入"探测当前真实 master"的预步骤，让 role 重跑时以 Sentinel 视角的真实拓扑为期望状态

**故障演练**

- 设计最小故障演练：手工 kill gz-03 redis-master，验证 Sentinel 选主后重跑 Redis role 不会把新 master 错误降级
- 结果记录到 `Docs/drills/v1.11-redis-sentinel-failover.md`

### 验收标准

- Redis Sentinel 完成一次主从切换后，重复执行 Redis role 不会把新 master 错误降级回 replica
- Sentinel 自动追加的运行时段（`sentinel known-replica` / `current-epoch` 等）不会被 Ansible 重渲染覆盖
- runbook 清楚写明 Redis Sentinel 的运行时状态边界，与 v1.10 的 ProxySQL 边界共同构成"HA 状态边界"完整文档
- 全节点重跑 redis-master / redis-replica role `changed=0 failed=0`

### 面试叙事点

- 为什么 HA 系统要区分"期望拓扑"和"故障后的运行时拓扑"（Ansible 持有的是期望拓扑，Sentinel 持有的是当下真实拓扑；两者冲突时要让 Sentinel 赢）
- 为什么 Ansible 幂等不等于"每次都强行覆盖到初始状态"（幂等的真正定义是"重复执行不偏离当前期望状态"，而不是"重复执行覆盖到初始期望状态"）
- Sentinel 自动选主与 Ansible 静态 `replicaof` 的冲突如何在 IaC 层面化解（两种思路对比：① role 内探测当前 master 再渲染；② 把 `replicaof` 完全交给 Sentinel，Ansible 只管首次 bootstrap）

---

<a id="cicd-state-iac"></a>

## 任务 7：CI/CD 状态 IaC 化（目标 v1.12）

> **slug**：`cicd-state-iac` ｜ 当前目标版本：**v1.12**

### 背景

bj-01 同时承担 Ansible 控制节点、Jenkins、Registry 和运维入口职责。当前 Jenkins 容器本身由 Ansible 管理，但 Jenkins 的关键内部状态仍依赖手工配置：自定义镜像 `jenkins-ansible:latest` 的构建过程、插件清单、Job、Credentials、`jenkins_home` 数据目录。v1.12 目标是把 CI/CD 平台自身也纳入"可备份、可恢复、可迁移"的管理边界。

### 要完成的内容

**Jenkins 镜像构建可追溯**

- 将 `jenkins-ansible:latest` 的 Dockerfile 和构建脚本纳入 Git
- 固定 Jenkins 基础镜像版本、Ansible 依赖、Docker CLI / Compose 插件依赖
- 明确镜像 tag 策略，避免只依赖不可追溯的本地 `latest`

**Jenkins 配置即代码**

- 评估并引入 Jenkins Configuration as Code（JCasC）或 Job DSL
- 将 `ansible-deploy`、`ruoyi-jenkins-docker-auto-deploy`、`registry-gc` 等 Job 定义纳入 Git
- 插件清单进入 Git，避免迁移时靠 UI 手工点选

**Credentials 与备份边界**

- Credentials 真值不进 Git，继续放在 Jenkins Credentials 或 vault 等安全位置
- 文档中明确每个 Credential 的用途、ID、恢复方式
- 设计 `jenkins_home` 备份 / 恢复策略，至少覆盖 Job、插件、Credentials 元数据与必要运行状态

### 验收标准

- 新机器可通过 Git 中的 Dockerfile 重新构建 Jenkins 运行镜像
- Jenkins 启动后能自动恢复核心 Job 定义
- 三个核心 Job 的触发方式、参数、凭据 ID 与现有生产状态一致
- 凭据真值未出现在普通文件、Git diff、日志输出中
- 完成一次 Jenkins 迁移或恢复演练，验证 CI/CD 链路可恢复

### 面试叙事点

- 为什么 CI/CD 平台自身也要被 IaC 管理（运维平台也是生产系统的一部分）
- Jenkins Job / Credentials 哪些适合进 Git，哪些只能记录恢复边界
- 为什么 `jenkins_home` 备份比普通应用容器备份更重要

---

<a id="six-node-onboarding"></a>

## 任务 8：六节点基础接入（目标 v1.13）

> **slug**：`six-node-onboarding` ｜ 当前目标版本：**v1.13**

### 背景

v1.12 已完成现有架构的基础可复现性、HA 状态边界和 CI/CD 状态管理修复，后续路线图（K3s 承载无状态服务、gz-02 角色重整）均需要额外节点支撑。v1.13 目标是将新购两台广州节点（gz-04、gz-05）安全、可重复地接入现有基础设施，**只做基础接入，不承载任何业务**：网络层纳入 Tailscale + Ansible IaC 管理，Docker 安装与基础运行时保持与现有节点一致，节点纳入 Prometheus 监控，新节点 SSH 采用 `admin-alex + become` 替代 root 直接登录还清技术债。

### 要完成的内容

**新增 Ansible Role**

- `roles/tailscale/`：安装 Tailscale（apt），加入 tailnet（`tailscale up --authkey --hostname`），幂等检查（已接入则跳过）

**现有 Role 扩展**

- `roles/docker-daemon/`：在现有 pip/SDK/daemon.json 配置任务之前**插入 Docker Engine 安装任务**（docker-ce + docker-compose-plugin），保持幂等；`global_gateway` 网络创建沿用 v1.9 已纳入的管理逻辑，gz-04/gz-05 走相同 role 时自动具备该网络；现有四台节点重复执行 `changed=0`
- `roles/node-exporter/`：模板改为条件渲染，`mysqld-exporter` 只在 `mysqld_exporter_nodes` 分组内启动；gz-04/gz-05 只跑 node-exporter；`setup_monitor.yml` 对应 play 从 `mysqld_exporter_nodes` 改为 `exporter_nodes:!monitor_nodes`

**新增 Playbook**

- `playbooks/setup_tailscale.yml`：Tailscale bootstrap 专用 playbook，经公网 IP 执行；接入完成后即切换 Tailscale IP，后续操作不再使用此 playbook

**Prometheus 更新**

- `prometheus.yml.j2` 新增 gz-04/gz-05 Tailscale IP 的 node-exporter target（IP 接入后填入）

**inventory / vault 变更**

- gz-04/gz-05 以公网 IP bootstrap，接入后替换为 Tailscale IP
- 新增 `managed_nodes` 分组；gz-04/gz-05 加入 `exporter_nodes`（不加入 `mysqld_exporter_nodes`）
- vault 新增 `tailscale_authkey`、`gz04_become_password`、`gz05_become_password` 三项加密变量

### 验收标准

- gz-04/gz-05 出现在 Tailscale 控制台，`tailscale ping` 从 bj-01 可达
- `docker info` 正常；`docker network ls` 可见 `global_gateway`
- Prometheus UI 中 `up{instance="<gz-04/gz-05 tailscale_ip>:9100"} = 1`
- Grafana Node Exporter 面板新节点主机指标（CPU / 内存 / 磁盘）可见
- 重复执行 docker-daemon + node-exporter role，`changed=0 failed=0`
- `vault/secrets.yml` 保持 AES256 加密，inventory 中不含明文密码

### 面试叙事点

- 为什么 Tailscale 接入也纳入 Ansible（IaC 可重复可审计，节点重建后重跑 playbook 即可）
- Reusable vs Ephemeral Auth Key 的区别（长期服务器用 Reusable，CI 临时容器用 Ephemeral）
- 为什么新节点改用 admin-alex + become 而不继续用 root（最小权限原则；旧节点延迟迁移是有意识的技术债管理）
- `global_gateway` 网络的作用（跨 compose stack 容器互通的共享 overlay 网络）

---

<a id="service-migration"></a>

## 任务 9：服务迁移（目标 v1.14）

> **slug**：`service-migration` ｜ 当前目标版本：**v1.14**

### 背景

v1.13 完成 gz-04/gz-05 基础接入后，gz-04 具备接管 gz-02 副本职责的前提（v1.13 计划标注：gz-04"v1.14 接管 gz-02 副本职责"），gz-05 作为 K3s 控制面候选节点在 v1.14 保持空闲。v1.14 目标是在不下线现有服务的前提下，完成三步角色重整：① gz-04 接管 gz-02 的数据库/缓存副本职责；② gz-02 接管 gz-01 的 Nginx 入口与主监控职责；③ gz-01 降级为轻量 watchdog 节点，守住"监控主节点宕机时仍有人告警"这条可用性底线。

> **同步识别到的网关侧耦合（不在 v1.14 修复范围）**：当前 `roles/nginx/templates/docker-compose.yml.j2:16` 把若依前端静态目录以 `./ruoyi-ui-dist:/usr/share/nginx/ruoyi:ro` 形式直接挂载进 Nginx 容器，导致**网关 compose 文件持有业务知识**——新增前端业务必须改这个 compose 并重启 Nginx。v1.0 引入 Nginx 时只有一个前端，做静态托管是单业务场景下的合理简化；但 v1.14 起若有更多前端业务接入，这层耦合会越来越显性。v1.14 网关迁移会**原样**把这层耦合搬到 gz-02，不在本版本展开解耦；自然演进路径是 v1.15 K3s 落地后，前端走"独立 Nginx 镜像 + Ingress"，把"路由声明"从网关层下沉到 app 自身，那时再彻底解耦。中间状态如果出现新增前端业务，按现有方式补 conf.d 与 volume 即可，**不为此提前重构网关 role**。

### 要完成的内容

**gz-04 接管 gz-02 副本职责**

- MySQL 从库（复制链路：gz-03 Master → gz-04 Slave1）；`Seconds_Behind_Source=0` 为验收基线
- Redis 从库 + Sentinel
- mysqld-exporter + node-exporter 纳入 Prometheus 监控
- 从 `app_nodes`、`db_replica_nodes`、`redis_replica_nodes`、`mysqld_exporter_nodes` 等 inventory 分组角度完整替换 gz-02 的位置

**gz-02 接管 gz-01 入口 + 主监控职责**

- Nginx 反向代理（公网入口；TLS 证书随配置迁移至 gz-02）
- Prometheus + Grafana + blackbox-exporter（主监控栈迁移）
- 迁移顺序：先在 gz-02 旁路部署并验证，再执行 DNS A 记录切换，验证业务可达后再降级 gz-01
- ProxySQL upstream 和若依后端配置中的入口地址相应更新

**gz-01 降级为轻量 watchdog 监控节点**

- 保留一个极轻量的监控实例，只盯 gz-02：node-exporter 健康、Nginx 健康端点、Prometheus 健康端点，以及公网域名 blackbox probe
- 告警走现有 Alertmanager（bj-01），仅数据来源从 gz-01 降为 watchdog 采集
- 目的：防范监控主节点（gz-02）宕机后告警链路中断，解决"谁来监控监控系统"的问题
- **覆盖边界声明**：watchdog 仅覆盖**采集端 / 评估端**单点（gz-02 主监控节点宕机时仍有备份采集），不覆盖**告警出口端**单点（Alertmanager / prometheus-alert / bj-01 整机）。出口端冗余由 v1.8 引入的 Dead Man's Switch 通过外部 SaaS 反向心跳兜底。两者是互补关系，不可替代。详见 `Docs/reviews/v1.7-alerting-pipeline-spof-audit.md`。

**gz-05 保持空闲**

- v1.14 阶段不部署任何业务；为 v1.15 K3s 控制面做准备，避免引入额外职责后给后续迁移增加复杂度

### 验收标准

- gz-04 的 MySQL 从库 `Seconds_Behind_Source=0`；Redis `master_link_status:up`
- gz-02 的 Prometheus targets 全部 `up`；Grafana 面板通过 gz-02 地址可访问
- 公网域名 DNS A 记录已指向 gz-02 公网 IP；若依业务可正常访问
- gz-01 watchdog 的 blackbox probe 及所有 node-exporter target 均 `up`
- 重复执行相关 Ansible role，全节点 `changed=0 failed=0`

### 面试叙事点

- 为什么 gz-01 不直接下线而是保留为 watchdog（"谁来监控监控系统"是经典可用性设计话题；单点监控是架构盲区，watchdog 以极低代价闭合这条链路）
- 为什么 watchdog 仍需要 v1.8 DMS 兜底（watchdog 只覆盖采集端单点，告警出口端单点必须由 DMS 通过外部 SaaS 反向心跳覆盖；HA 设计要按独立失效域分别审视而非按组件名整体处理）
- 服务迁移为什么走"旁路部署 → DNS 切换 → 降级旧节点"三步而非直接替换（保证零中断、任一步失败可回滚，体现灰度发布思路）
- gz-04 接管 gz-02、gz-05 留作 K3s 控制面——为何与 v1.13 草案预设的角色互换（根据节点后续职责调整：K3s 控制面需要保持专一，而承载 MySQL 从库的节点不适合同时成为 K3s master）
- 网关与业务前端的 volume 挂载耦合识别后**为何不在 v1.14 修复**（v1.14 网关迁移本身复杂度已足够，硬塞解耦会扩大失败面；解耦的最佳时机是 v1.15 K3s + Ingress 自然达成，体现"识别问题但克制处理"的工程取舍——既不忽视盲点，也不为"看起来专业"而越级重构）

---

<a id="k3s-stateless"></a>

## 任务 10：K3s 迁移无状态服务（目标 v1.15）

> **slug**：`k3s-stateless` ｜ 当前目标版本：**v1.15**

### 背景

当前若依后端（ruoyi-admin）以 Docker Compose 方式运行在 gz-03 和 gz-02 上。v1.15 目标是在不动有状态服务（MySQL、Redis、ProxySQL）的前提下，将若依后端迁移到 K3s，验证 Kubernetes 基础工作流。

### 要完成的内容

**K3s 部署**

- 选择 gz-05 作为 K3s 控制面节点（v1.13 已完成基础接入，运行时已就绪；v1.14 专为此保持空闲）
- 验证 `kubectl get nodes` 状态 Ready

**应用迁移**

- 将若依后端或一个等效的测试 Web 服务迁移到 K3s
- 编写以下 Kubernetes 对象：
  - `Deployment`（副本数 ≥ 2，配置 `rollingUpdate` 策略）
  - `Service`（ClusterIP 或 NodePort）
  - `Ingress`（使用 K3s 内置 Traefik 或自装 Nginx Ingress）
  - `ConfigMap`（管理应用配置，如数据库连接指向 ProxySQL 100.92.5.116:6033）
  - `Secret`（管理数据库密码，不明文写入 YAML）
  - `livenessProbe` / `readinessProbe`（基于若依的 `/actuator/health` 或等效路径）

**保留有状态服务**

- MySQL、Redis、ProxySQL 继续在 Docker Compose 中运行，不迁移
- K3s 中的应用通过 Tailscale 内网地址访问 ProxySQL（100.92.5.116:6033）

**Nginx 网关配置**

- gz-01 的 Nginx upstream 新增 K3s 节点的 NodePort 或 Ingress 地址
- 验证通过公网域名可以访问 K3s 上运行的服务

### 验收标准

- `kubectl get pods` 显示 Running，副本数正确
- 服务可通过 Ingress 从外部访问
- 执行滚动更新（更改镜像 tag），更新过程中服务不中断
- 故意配置一个不存在的镜像，能看到 `ImagePullBackOff`，通过 `kubectl describe pod` 定位原因
- 故意配错 `readinessProbe` 路径，能看到滚动更新卡住，通过 `kubectl rollout status` 和 events 定位原因
- 能解释 Deployment → ReplicaSet → Pod → Service → Ingress 的层级关系

### 面试叙事点

- 为什么有状态服务不迁移到 K3s（复杂度超出初级岗，现有 Compose 方案成熟）
- K3s 和 K8s 的区别（K3s 是 CNCF 认证的轻量 K8s，二进制单文件，适合边缘和学习）
- readinessProbe 和 livenessProbe 的区别，以及实际触发效果

---

<a id="helm-chart-mgmt"></a>

## 任务 11：Helm Chart 管理 K3s 应用（目标 v1.16）

> **slug**：`helm-chart-mgmt` ｜ 当前目标版本：**v1.16**

### 背景

v1.15 的 K3s 应用以裸 YAML 方式管理。v1.16 目标是将若依后端的 Kubernetes YAML 封装成 Helm Chart，支持多环境参数化部署。

### 要完成的内容

**Helm Chart 结构**

```
charts/ruoyi/
├── Chart.yaml
├── values.yaml          ← 默认值（dev 环境）
├── values-prod.yaml     ← 生产覆盖值
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    └── secret.yaml
```

**参数化内容**

`values.yaml` 中至少支持通过参数控制：

- `image.repository` 和 `image.tag`
- `replicaCount`
- 数据库连接地址（ProxySQL host/port）
- Ingress hostname

**多环境支持**

- 准备 `values.yaml`（dev）和 `values-prod.yaml`（prod）两套配置
- 验证 `helm install ruoyi ./charts/ruoyi -f values-prod.yaml` 能部署生产配置

**Jenkins 集成（可选但加分）**

- Jenkinsfile 中的 Deploy stage 改为 `helm upgrade --install` 命令
- 镜像 tag 通过 `--set image.tag=<commit-sha>` 注入

### 验收标准

- `helm install` 成功部署应用
- `helm upgrade --set image.tag=<new-tag>` 成功更新镜像
- `helm rollback` 成功回滚到上一版本
- `helm template` 能渲染出正确的 YAML，用于 debug
- 能解释 Chart / templates / values / release 各自的职责

### 面试叙事点

- 为什么用 Helm 而不是直接 kubectl apply（版本管理、rollback、参数化）
- values.yaml 和 values-prod.yaml 的区别如何体现 12-factor 配置原则
- `helm upgrade --install` 为什么比 `helm install` 更常用

---

<a id="loki-logging"></a>

## 任务 12：Loki 日志集中管理（目标 v1.17）

> **slug**：`loki-logging` ｜ 当前目标版本：**v1.17**

### 背景

当前排障依赖各节点分散的 `docker logs`，没有集中日志检索能力。v1.17 目标是将 Docker Compose 和 K3s 的容器日志统一采集到 Loki，通过 Grafana 实现跨节点日志查询。

### 要完成的内容

**部署 Loki**

- 在 gz-01 的 monitor-stack compose 中增加 Loki 容器（与 Prometheus/Grafana 同 compose）
- 使用 `filesystem` 存储模式（单机 boltdb-shipper 或 TSDB），不引入额外对象存储依赖

**日志采集**

- 在各节点部署 Promtail（或 Grafana Alloy），通过 Ansible 统一管理配置
- 采集对象：所有 Docker Compose 容器日志（通过 `/var/lib/docker/containers/*/*.log` 方式）；K3s 节点额外采集 `/var/log/pods/`
- label 设计至少包含：`node`（gz-01/gz-02/gz-03/bj-01）、`service`（容器名）、`env`（prod）

**Grafana 配置**

- 添加 Loki 作为 Grafana 数据源
- 在 Explore 中验证能按 node 和 service 过滤日志
- 创建一个简单的日志 dashboard：展示各节点最新错误日志

### 验收标准

- 在 Grafana Explore 中能查到 gz-03 ruoyi 容器最近 1 小时的日志
- 能通过日志定位一次模拟故障（如后端报 DB 连接失败）
- Ansible 管理 Promtail 配置，重复执行 `changed=0`
- 能讲清日志从容器 stdout → Promtail → Loki → Grafana 的完整链路

### 面试叙事点

- 为什么用 Loki 而不是 ELK（资源消耗、与 Grafana 生态集成、够用原则）
- label 设计为什么重要（高基数 label 会导致性能问题）
- Loki 和 Prometheus 在数据模型上的相似之处（label + 时间序列 vs label + 日志流）

---

<a id="observability-integration"></a>

## 任务 13：可观测性增强（目标 v2.0）

> **slug**：`observability-integration` ｜ 当前目标版本：**v2.0**

### 背景

到 v1.17 为止，监控（Prometheus）、告警（Alertmanager）、日志（Loki）已各自成型但相互独立。v2.0 目标是在 Grafana 中将三者整合，形成"指标异常 → 定位时间点 → 关联日志 → 追查根因"的完整排障链路。

### 要完成的内容

**Dashboard 整合**

- 为每个关键服务建立统一 dashboard，页面内包含：
  - 核心指标图表（来自 Prometheus）
  - 关联日志面板（来自 Loki）
  - 当前活跃告警状态（来自 Alertmanager API 或 Grafana Alerting）
- 重点 dashboard：
  - 节点总览（六节点 CPU/内存/磁盘/网络）
  - MySQL 主从状态（复制延迟、QPS、连接数）
  - 若依后端（请求量、响应时间、错误率）
  - Nginx 入口（连接数、4xx/5xx 比例、upstream 健康）

**新增 Exporter（按需）**

- Redis Exporter（监控 Redis 主从状态和内存使用）
- ProxySQL Exporter（监控后端连接池和查询路由状态）
- Nginx Exporter 或通过 Prometheus 解析 Nginx stub_status

**告警规则补充**

- 根据 Dashboard 观测到的正常区间，调优 v1.5 的告警阈值
- 新增若依后端层面的业务告警（如 5xx 率超阈值）

### 验收标准

- 能从 Grafana 一个页面判断集群整体健康状态
- 能通过时间关联，从指标图表跳转到对应时间段的日志
- 告警阈值经过实际观测数据调优，不是经验值
- 能在 5 分钟内完成一次从"收到告警"到"定位根因"的完整排障演示

---

## 可选增强项：bj-01 管理面出站代理

### 背景

bj-01 是 Ansible 控制节点、Jenkins、Registry 和运维工具所在节点。部分管理面任务会从 bj-01 直接访问境外服务，例如 Cursor SSH Remote 下的 AI 请求、OpenAI / Claude API、GitHub、Docker Hub、Maven / npm / pip 依赖源等。本地电脑开启代理只能影响本地到 bj-01 的 SSH 连接，不能保证 bj-01 自身出站访问稳定。

### 建议范围

- 仅在 bj-01 上按需配置代理，不纳入业务节点默认配置
- 代理只服务管理面出站能力，不承载线上业务流量
- 优先给 Jenkins、Docker daemon、Git、Maven / npm / pip、AI CLI 或 Cursor Remote shell 等需要访问境外服务的进程配置代理环境变量
- 必须配置 `NO_PROXY`，覆盖 `localhost`、`127.0.0.1`、Tailscale 网段和集群内网服务，避免 Ansible SSH、Registry 拉取、Prometheus 采集、MySQL / Redis 复制等内网链路误走代理

### 不做范围

- 不做全节点统一代理
- 不做业务流量透明代理
- 不把代理作为当前 v1.8-v2.0 主线任务的硬性前置条件（v1.8 起为新主线起点）
- 不在普通配置文件中写入代理账号、密码、token 等敏感信息

### 触发条件

当 bj-01 上的 CI/CD、依赖下载、模型 API 调用或 Cursor Remote AI 功能频繁出现境外访问失败时，再作为独立小任务补充 proposal / runbook / retrospective，并通过 Ansible 管理相关配置。

---

## 横向运维成熟度任务（不绑定版本号）

本节登记**不构成架构语义变更、不需要 Git tag**、但属于运维成熟度建设的横向任务。这类任务跨架构版本稳定存在，不进入上方 v1.X 子任务路线，由对应文档自身承担追踪职责。

| 任务 | 主文档 | 触发节奏 | 当前状态 |
|---|---|---|---|
| SLI/SLO 契约定义与月度回填 | [`Docs/sli-slo.md`](../sli-slo.md) | 月初回填 §5 实测样本；季度评估 §3 SLO 目标 | 骨架已建，待首次月度数据回填 |
| 故障演练 | [`Docs/drills/`](../drills/) | 每季度至少 1 次；新版本上线后验证新增能力 | 子体系已建；首份演练 [`v1.7-gz02-app-failure.md`](../drills/v1.7-gz02-app-failure.md) 计划中 |

**与 v1.X 子任务的关系**：

- 横向任务**不依赖** v1.X 子任务进度，可独立推进，但下游会引用：例如 v1.10 ProxySQL HA 化落地后，应在 `Docs/drills/` 新增一份 ProxySQL 单实例故障切换演练，验证 JDBC 多端点 failover；v1.11 Redis Sentinel 边界修复落地后，类似补一份 Sentinel 选主后 Ansible 重跑验证演练
- 横向任务暴露的问题如构成跨版本盲点，按 `.cursor/rules/10-docs-workflow.mdc` §6 规则升级为 `Docs/reviews/v{触发版本}-{slug}.md`，再链接到本节或 v1.X 子任务
- 横向任务**不**进入 `arch-vX.Y` Git tag 的发布前置条件，避免与版本演进节奏耦合

---

## 文档规范提醒

每个子任务完成后，按 `.cursor/rules/10-docs-workflow.mdc` 完成以下文档：

1. **Proposal**：`Docs/proposals/vX.X-<主题>.md`，开工前写，说明"做什么、为什么这么做、不做什么"
2. **Runbook**：`Docs/runbooks/<操作名>.md`，操作手册，可操作可复现
3. **Retrospective**：`Docs/retrospectives/vX.X.md`，完成后写，复盘踩坑和决策
4. **架构快照**：`Docs/architecture/vX.X.md`，打版本标签前更新
