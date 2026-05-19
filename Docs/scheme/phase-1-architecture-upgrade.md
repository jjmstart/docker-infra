# 阶段一：架构升级（v1.5 → v2.0）

## 阶段目标

将 `docker-infra` 集群从 v1.5 路线起点延伸至 v2.0，当前已落地到 v1.7。每个子任务对应一个架构版本，每版本完成后需按文档规范完成 proposal → runbook → retrospective → 架构快照的完整流程。

**当前起点**：v1.7 已完成备份恢复最小闭环（gz-03 mysqldump 逻辑备份 + 阿里云 OSS 上传 + bj-01 恢复演练，实测 RTO 34 秒），已通过端到端验收，Git tag `arch-v1.7` 已推送。

**路线调整**：v1.7 全量复盘中发现，当前 Ansible 已能稳定管理 Compose 配置和服务启动，但仍存在若干"历史一次性操作未纳入 IaC"的问题，例如 `mysql_source_delay` 已写入 inventory 但复制通道初始化仍依赖历史手工状态。为避免在扩节点 / K3s 迁移前继续放大这些隐性状态，v1.8-v1.10 先补齐架构可复现性与 HA 状态管理，原六节点接入及后续 K3s 路线顺延。

---

## 子任务完成状态

| 子任务                                          | 目标版本  | 状态      |
| --------------------------------------------- | ----- | ------- |
| 告警系统                                          | v1.5  | ✅ 已完成   |
| 应用交付流水线（镜像构建推送）                               | v1.6  | ✅ 已完成   |
| 备份恢复最小闭环                                      | v1.7  | ✅ 已完成   |
| 基础可复现性修复（Docker 网络 + MySQL 数据层 bootstrap）       | v1.8  | 待开始     |
| HA 配置状态修复（ProxySQL runtime + Redis Sentinel 边界）  | v1.9  | 待开始     |
| CI/CD 状态 IaC 化（Jenkins 镜像、Job、Credentials 备份边界） | v1.10 | 待开始     |
| 六节点基础接入（gz-04/gz-05 Tailscale + Docker + 监控纳管） | v1.11 | 待开始     |
| 服务迁移（gz-01/gz-02 角色重整）                        | v1.12 | 待开始     |
| K3s 迁移无状态服务                                   | v1.13 | 待开始     |
| Helm Chart 管理 K3s 应用                          | v1.14 | 待开始     |
| Loki 日志集中管理                                   | v1.15 | 待开始     |
| 可观测性增强（Dashboard 整合）                          | v2.0  | 待开始     |


---

## 任务 1：应用交付流水线（目标 v1.6）

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

## 任务 2：备份恢复最小闭环（目标 v1.7）

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

## 任务 3：基础可复现性修复（目标 v1.8）

### 背景

v1.7 全量复盘中，从 `inventory/hosts.yml` 中 `mysql_source_delay: 3600` 未被 role 实际消费这一点，暴露出当前 IaC 的边界：Ansible 已经能管理容器编排文件，但部分集群关键状态仍来自历史手工初始化，例如 Docker external network、MySQL 业务库和账号、MySQL 复制通道。v1.8 目标是补齐"新机器从零部署时必须存在的基础状态"，让 Ansible 不只会拉起容器，还能复现数据层的最低可用形态。

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

## 任务 4：HA 配置状态修复（目标 v1.9）

### 背景

v1.8 解决"新机器能否从零复现基础状态"，但 HA 组件还存在运行时状态与配置文件不完全一致的问题。典型例子是 ProxySQL 的 `proxysql.cnf` 只在首次初始化 datadir 时读取，之后变更需要通过 Admin 接口 `LOAD ... TO RUNTIME` / `SAVE ... TO DISK`；Redis Sentinel 则会在运行过程中改写自身状态，静态 `replicaof` 配置在故障切换后可能与实际主从拓扑冲突。v1.9 目标是修复这类"配置已写入 Git，但运行时不一定生效"的问题。

### 要完成的内容

**ProxySQL Runtime 状态同步**

- 将 `roles/proxysql/` 从"只渲染 `proxysql.cnf`"升级为"配置文件 + Admin SQL 双路径管理"
- 对 `mysql_servers`、`mysql_users`、`mysql_query_rules` 执行幂等同步
- 每次变更后执行对应的 `LOAD MYSQL ... TO RUNTIME` 和 `SAVE MYSQL ... TO DISK`
- 明确区分 cnf 首次初始化职责与 runtime DB 持久化职责

**Redis Sentinel / Replica 边界修复**

- 梳理 `replicaof {{ redis_master_host }}` 与 Sentinel 自动 failover 的冲突边界
- 在 role 中避免故障切换后无条件把已提升的 master 重新降级为 replica
- 明确哪些状态由 Ansible 管，哪些状态由 Sentinel 在运行时接管
- 必要时拆分"初始拓扑 bootstrap"与"运行期健康检查"两类任务

**HA 验证与负向演练**

- 设计最小故障演练：ProxySQL 后端调整、Redis master 故障、Sentinel 选主后 Ansible 重跑
- 验证 Ansible 重跑不会破坏已完成的 HA 切换结果

### 验收标准

- 修改 ProxySQL 后端或规则后，无需删除 datadir，runtime 查询能看到新配置
- `mysql_servers`、`mysql_users`、`mysql_query_rules` 均已 `SAVE ... TO DISK`
- Redis Sentinel 完成一次主从切换后，重复执行 Redis role 不会把新 master 错误降级
- runbook 清楚写明 ProxySQL 与 Redis Sentinel 的运行时状态边界

### 面试叙事点

- 为什么有些组件不能只改配置文件（ProxySQL runtime DB 与 cnf 的职责不同）
- 为什么 HA 系统要区分"期望拓扑"和"故障后的运行时拓扑"
- 为什么 Ansible 幂等不等于"每次都强行覆盖到初始状态"

---

## 任务 5：CI/CD 状态 IaC 化（目标 v1.10）

### 背景

bj-01 同时承担 Ansible 控制节点、Jenkins、Registry 和运维入口职责。当前 Jenkins 容器本身由 Ansible 管理，但 Jenkins 的关键内部状态仍依赖手工配置：自定义镜像 `jenkins-ansible:latest` 的构建过程、插件清单、Job、Credentials、`jenkins_home` 数据目录。v1.10 目标是把 CI/CD 平台自身也纳入"可备份、可恢复、可迁移"的管理边界。

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

## 任务 6：六节点基础接入（目标 v1.11）

### 背景

v1.10 已完成现有架构的基础可复现性、HA 状态边界和 CI/CD 状态管理修复，后续路线图（K3s 承载无状态服务、gz-02 角色重整）均需要额外节点支撑。v1.11 目标是将新购两台广州节点（gz-04、gz-05）安全、可重复地接入现有基础设施，**只做基础接入，不承载任何业务**：网络层纳入 Tailscale + Ansible IaC 管理，Docker 安装与基础运行时保持与现有节点一致，节点纳入 Prometheus 监控，新节点 SSH 采用 `admin-alex + become` 替代 root 直接登录还清技术债。

### 要完成的内容

**新增 Ansible Role**

- `roles/tailscale/`：安装 Tailscale（apt），加入 tailnet（`tailscale up --authkey --hostname`），幂等检查（已接入则跳过）

**现有 Role 扩展**

- `roles/docker-daemon/`：在现有 pip/SDK/daemon.json 配置任务之前**插入 Docker Engine 安装任务**（docker-ce + docker-compose-plugin），保持幂等；`global_gateway` 网络创建沿用 v1.8 已纳入的管理逻辑，gz-04/gz-05 走相同 role 时自动具备该网络；现有四台节点重复执行 `changed=0`
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

## 任务 7：服务迁移（目标 v1.12）

### 背景

v1.11 完成 gz-04/gz-05 基础接入后，gz-04 具备接管 gz-02 副本职责的前提（v1.11 计划标注：gz-04"v1.12 接管 gz-02 副本职责"），gz-05 作为 K3s 控制面候选节点在 v1.12 保持空闲。v1.12 目标是在不下线现有服务的前提下，完成三步角色重整：① gz-04 接管 gz-02 的数据库/缓存副本职责；② gz-02 接管 gz-01 的 Nginx 入口与主监控职责；③ gz-01 降级为轻量 watchdog 节点，守住"监控主节点宕机时仍有人告警"这条可用性底线。

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

**gz-05 保持空闲**

- v1.12 阶段不部署任何业务；为 v1.13 K3s 控制面做准备，避免引入额外职责后给后续迁移增加复杂度

### 验收标准

- gz-04 的 MySQL 从库 `Seconds_Behind_Source=0`；Redis `master_link_status:up`
- gz-02 的 Prometheus targets 全部 `up`；Grafana 面板通过 gz-02 地址可访问
- 公网域名 DNS A 记录已指向 gz-02 公网 IP；若依业务可正常访问
- gz-01 watchdog 的 blackbox probe 及所有 node-exporter target 均 `up`
- 重复执行相关 Ansible role，全节点 `changed=0 failed=0`

### 面试叙事点

- 为什么 gz-01 不直接下线而是保留为 watchdog（"谁来监控监控系统"是经典可用性设计话题；单点监控是架构盲区，watchdog 以极低代价闭合这条链路）
- 服务迁移为什么走"旁路部署 → DNS 切换 → 降级旧节点"三步而非直接替换（保证零中断、任一步失败可回滚，体现灰度发布思路）
- gz-04 接管 gz-02、gz-05 留作 K3s 控制面——为何与 v1.11 草案预设的角色互换（根据节点后续职责调整：K3s 控制面需要保持专一，而承载 MySQL 从库的节点不适合同时成为 K3s master）

---

## 任务 8：K3s 迁移无状态服务（目标 v1.13）

### 背景

当前若依后端（ruoyi-admin）以 Docker Compose 方式运行在 gz-03 和 gz-02 上。v1.13 目标是在不动有状态服务（MySQL、Redis、ProxySQL）的前提下，将若依后端迁移到 K3s，验证 Kubernetes 基础工作流。

### 要完成的内容

**K3s 部署**

- 选择 gz-05 作为 K3s 控制面节点（v1.11 已完成基础接入，运行时已就绪；v1.12 专为此保持空闲）
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

## 任务 9：Helm Chart 管理 K3s 应用（目标 v1.14）

### 背景

v1.13 的 K3s 应用以裸 YAML 方式管理。v1.14 目标是将若依后端的 Kubernetes YAML 封装成 Helm Chart，支持多环境参数化部署。

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

## 任务 10：Loki 日志集中管理（目标 v1.15）

### 背景

当前排障依赖各节点分散的 `docker logs`，没有集中日志检索能力。v1.15 目标是将 Docker Compose 和 K3s 的容器日志统一采集到 Loki，通过 Grafana 实现跨节点日志查询。

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

## 任务 11：可观测性增强（目标 v2.0）

### 背景

到 v1.15 为止，监控（Prometheus）、告警（Alertmanager）、日志（Loki）已各自成型但相互独立。v2.0 目标是在 Grafana 中将三者整合，形成"指标异常 → 定位时间点 → 关联日志 → 追查根因"的完整排障链路。

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
- 不把代理作为当前 v1.8-v2.0 主线任务的硬性前置条件
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

- 横向任务**不依赖** v1.X 子任务进度，可独立推进，但下游会引用：例如 v1.9 HA 配置状态修复落地后，应在 `Docs/drills/` 新增一份 v1.9 故障切换演练，验证修复结果
- 横向任务暴露的问题如构成跨版本盲点，按 `.cursor/rules/10-docs-workflow.mdc` §6 规则升级为 `Docs/reviews/v{触发版本}-{slug}.md`，再链接到本节或 v1.X 子任务
- 横向任务**不**进入 `arch-vX.Y` Git tag 的发布前置条件，避免与版本演进节奏耦合

---

## 文档规范提醒

每个子任务完成后，按 `.cursor/rules/10-docs-workflow.mdc` 完成以下文档：

1. **Proposal**：`Docs/proposals/vX.X-<主题>.md`，开工前写，说明"做什么、为什么这么做、不做什么"
2. **Runbook**：`Docs/runbooks/<操作名>.md`，操作手册，可操作可复现
3. **Retrospective**：`Docs/retrospectives/vX.X.md`，完成后写，复盘踩坑和决策
4. **架构快照**：`Docs/architecture/vX.X.md`，打版本标签前更新
