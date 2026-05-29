# 架构草案 — 六节点基础接入

> **状态**：草案讨论中
> **目标主题**：[`six-node-onboarding`](../scheme/phase-1-architecture-upgrade.md#six-node-onboarding)（当前目标版本以路线图为准）
> **前置主题**：无（v1.7 已完成；本主题是 phase-1 路线"K3s 前置三连"的起点）
> **后置主题**：[`k3s-stateless`](../scheme/phase-1-architecture-upgrade.md#k3s-stateless) 直接依赖本主题接入的 gz-04 / gz-05 作为控制面与 worker 节点
> **关联 review**：本主题顺带回收 [`base-reproducibility-fix`](../scheme/phase-1-architecture-upgrade.md#base-reproducibility-fix) 中"Docker 运行时基础状态"段（详见 §2.5）

---

## 1. 设计目标

把 gz-04 / gz-05 两台广州 4G 节点安全、可重复地接入现有基础设施，**只做基础接入，不承载业务工作负载**。本主题落地后的边界：

- **网络层**：两节点加入现有 Tailscale tailnet，所有后续运维操作走 Tailscale 内网，不依赖公网端口暴露；Tailscale 安装与入网由 Ansible IaC 管理
- **运行时层**：Docker Engine 以与现有四台节点一致的方式安装；`global_gateway` overlay 网络由 Ansible 管理（顺带把现有四台节点的手工创建状态收编到 IaC，[`base-reproducibility-fix`](../scheme/phase-1-architecture-upgrade.md#base-reproducibility-fix) 主题对应一项随之结清）
- **观测层**：两节点纳入 Prometheus 监控（仅 node-exporter，不装 mysqld-exporter），Grafana 主机面板可见
- **接入身份层**：新增可复用 `roles/base-access/`，在**全部 6 台节点**幂等创建 `admin-alex` 用户 + `authorized_keys` + NOPASSWD sudoers（**纯追加，不删除 root**）；但只把 **gz-04 / gz-05 的 `ansible_user` 翻转为 admin-alex**，现有四台节点保持 `ansible_user: root` 连接不变。现有四台节点的 `ansible_user` 翻转 + Jenkins 凭据切换 + 关闭 root 登录拆为独立后续任务，原因见 §6——它咬住 Jenkins `ansible-ssh-key` 凭据与 bj-01 自管理两个耦合，且对象是生产节点，需要单独的回滚预案窗口

完成本主题后，[`k3s-stateless`](../scheme/phase-1-architecture-upgrade.md#k3s-stateless) 主题可以直接在 gz-04 / gz-05 上展开 K3s 安装，不再需要做任何节点接入相关的前置工作；同时 6 台节点的 admin-alex 鉴权身份已就绪，后续 cutover 任务只需翻转连接用户与切换 CI 凭据，不必再碰节点侧。

---

## 2. 方案范围

### 2.1 新增 Ansible Role

| Role | 职责 | 关键任务 |
|---|---|---|
| `roles/tailscale/` | Tailscale 安装 + 入网 + 幂等检查 | apt 安装 `tailscale` 包；幂等运行 `tailscale up --authkey={{ tailscale_authkey }} --hostname={{ inventory_hostname }} --ssh=false`；通过 `tailscale status --json` 解析判断"已接入则跳过"；接入完成后输出节点的 Tailscale IP 用于回填 inventory |
| `roles/base-access/` | 节点接入身份管理（admin-alex 用户 + 公钥 + sudoers） | ① `ansible.builtin.user` 幂等创建 `admin-alex`（指定 shell、主组）；② `ansible.posix.authorized_key` 把 `admin_alex_authorized_keys` 列表里的公钥写入 admin-alex（含运维操作机公钥 + Jenkins `ansible-ssh-key` 对应公钥，**两把都加，为后续 cutover 预留**）；③ `ansible.builtin.copy` + `validate: 'visudo -cf %s'` 写 `/etc/sudoers.d/admin-alex`（`admin-alex ALL=(ALL) NOPASSWD:ALL`，避免 `requiretty` 以兼容 `pipelining=True`）。对 6 台全节点幂等：gz-04/05 上 cloud-init 已建则收敛，老四台上是首次创建（纯追加，root 不动） |

### 2.2 现有 Role 扩展

| Role | 改动 | 备注 |
|---|---|---|
| `roles/docker-daemon/` | ① 在现有 pip / SDK / daemon.json 任务**之前**插入 Docker Engine 安装任务（apt 仓库 + GPG key + `docker-ce` + `docker-compose-plugin`，固定版本由 `group_vars/all.yml` 变量控制）；② 新增 `community.docker.docker_network` 任务管理 `global_gateway` overlay 网络 | 必须保持幂等；现有四台节点重复执行不破坏运行状态——**这是回收 `base-reproducibility-fix` 主题的关键校验**（详见 §2.5） |
| `roles/node-exporter/` | **不动 role 模板**——node-exporter 与 mysqld-exporter 已是两个独立 role，setup_monitor.yml 也已是两个独立 play 各指向不同分组；本主题只需在 inventory 把 gz-04 / gz-05 加入 `exporter_nodes`、不加入 `mysqld_exporter_nodes` 即可 | 路线图原文"模板改为条件渲染"是基于"两个 exporter 还在同一 role"的过时假设；该差异在路线图同 PR 修正（详见 §2.5） |

### 2.3 新增 Playbook

| Playbook | 入口 | 触发场景 |
|---|---|---|
| `playbooks/setup_tailscale.yml` | `tailscale` role on `managed_nodes` 分组 | 新节点 bootstrap 专用：经公网 IP 一次性执行；接入完成后即切换至 Tailscale IP，后续日常 site.yml 不再走此 playbook |
| `playbooks/setup_access.yml` | `base-access` role on `managed_nodes` 分组 | 节点接入身份收敛；同时 `import_playbook` 进 `site.yml` **最前**（在 docker-daemon 之前），保证每次全量部署都收敛 admin-alex 鉴权状态。日常重跑幂等 `changed=0` |

bootstrap 期间 `ansible_host` 通过命令行覆盖为公网 IP，inventory 文件本身保持 Tailscale IP 不动：

```bash
ansible-playbook playbooks/setup_tailscale.yml \
  --limit gz-04 \
  -e "ansible_host=<gz-04 公网 IP>" \
  --ask-vault-pass
```

playbook 文件在 git 中长期保留，下次再加新节点（gz-06、bj-02 等）可直接复用，不归档到 `playbooks/archive/`。

### 2.4 inventory / vault 变更

**inventory/hosts.yml**

新增 gz-04 / gz-05 主机段（`ansible_host` 写最终 Tailscale IP）。**只有这两台新节点把 `ansible_user` 设为 admin-alex**；现有 gz-01 / gz-02 / gz-03 / bj-01 保持 `ansible_user: root` 不变（翻转拆出，见 §6）：

```yaml
gz-04:
  ansible_host: <Tailscale IP，bootstrap 后回填>
  ansible_user: admin-alex

gz-05:
  ansible_host: <Tailscale IP，bootstrap 后回填>
  ansible_user: admin-alex
```

新增 / 修改分组：

```yaml
managed_nodes:                 # 新增分组：本主题及之后所有 IaC 管理的节点
  hosts:
    gz-01:
    gz-02:
    gz-03:
    gz-04:
    gz-05:
    bj-01:

exporter_nodes:                # 在原有 4 台基础上新增 gz-04 / gz-05
  hosts:
    gz-01:
    gz-02:
    gz-03:
    gz-04:
    gz-05:
    bj-01:

# mysqld_exporter_nodes 不变，仍只含 gz-02 / gz-03 / bj-01
```

**vault/secrets.yml**

新增 1 项加密变量：

| 变量名 | 用途 | 形态 |
|---|---|---|
| `tailscale_authkey` | Tailscale 入网认证 | 单一 reusable + preauthorized + tag:server 的 auth key；后续加新节点共用同一 key，vault 不再扩条目 |

**不引入** `gz04_become_password` / `gz05_become_password`——admin-alex 走 NOPASSWD sudoers，become 无密码交互。

**inventory/group_vars/all.yml**

新增 Docker / Tailscale 安装相关公共变量，以及 base-access 的授权公钥列表：

```yaml
# Docker Engine 版本（所有 managed_nodes 统一；变更需走 base-reproducibility-fix 之后的 IaC 流程）
docker_ce_version: "5:27.3.1-1~debian.12~bookworm"  # 占位值，落地前以实测确定的稳定版本回填
docker_compose_plugin_version: "2.29.7-1~debian.12~bookworm"

# Tailscale apt repo 与 hostname 约定（hostname 直接复用 inventory_hostname）
tailscale_apt_distribution: "bookworm"  # 占位值；待 gz-04 / gz-05 系统版本确定后回填

# admin-alex 授权公钥列表（公钥非敏感，可明文进 Git）
# 两把都加：运维操作机公钥 + Jenkins ansible-ssh-key 对应公钥，为后续 cutover 预留
admin_alex_authorized_keys:
  - "<运维操作机 admin-alex 公钥，落地前回填>"
  - "<Jenkins ansible-ssh-key credential 对应公钥，落地前回填>"
```

> 公钥可明文入 Git，私钥与 vault 凭据不在本主题改动范围。Jenkins `ansible-ssh-key` 凭据本身（用户名仍为 root）本主题**不动**，只是把其公钥提前授权给 admin-alex，等后续 cutover 任务再翻转用户名。

### 2.5 路线图与 base-reproducibility-fix 回收范围

本主题 PR 同时修订 `Docs/scheme/phase-1-architecture-upgrade.md` 两处：

1. **`six-node-onboarding` 段**：把"`roles/node-exporter/` 模板改为条件渲染"改为"`roles/node-exporter/` 与 `roles/mysqld-exporter/` 已是两个独立 role，本主题仅在 inventory 把 gz-04/gz-05 加入 `exporter_nodes`，不加入 `mysqld_exporter_nodes`"
2. **`base-reproducibility-fix` 段**：把"Docker 运行时基础状态"小节标注为"已在 v1.8 `six-node-onboarding` 主题落地，本主题只剩 MySQL 数据层 bootstrap 与复制状态管理两件事"

修订后路线图的"主题 → 当前目标版本"映射不变，slug 不动，下游 review 引用无需联动。

### 2.6 决策点对比表

#### Docker Engine 安装方式

| 方案 | 优点 | 缺点 | 推荐 |
|---|---|---|---|
| Docker 官方 apt 仓库（`download.docker.com`） | 版本可控、可固定；与中小型公司主流做法一致；GPG key + apt source 可由 Ansible 完整管理 | 安装路径稍复杂（需配 GPG + repo + apt update + 安装） | **推荐** |
| convenience script（`get.docker.com`） | 一行命令最简单 | 与现有四台节点不一致；版本不可固定；依赖外网；不适合 IaC | 不选 |
| 云厂商镜像源默认 docker-ce | 开箱即用 | 版本受云厂商节奏控制；多云节点版本不齐；难以统一管理 | 不选 |

**结论**：选 Docker 官方 apt 仓库；版本通过 `docker_ce_version` 变量固定，重复执行幂等。现有四台节点首次跑新版 docker-daemon role 时如果检测到本地 Docker 版本与变量目标版本一致，apt 模块返回 `changed=0`，不会触发升级。

#### Tailscale auth key 形态

| 属性 | 选择 | 理由 |
|---|---|---|
| reusable | ✓ 启用 | 节点池小且稳，单 key 共用便于 vault 收敛 |
| preauthorized | ✓ 启用 | 否则 Ansible 自动化卡在控制台手动审批 |
| ephemeral | ✗ 不启用 | 长期服务器节点不能因离线被自动剔除 |
| tag | ✓ 设 `tag:server` | 为未来 Tailscale ACL 留接口 |

#### SSH / become 模型

| 项 | 选择 | 理由 |
|---|---|---|
| admin-alex 供给 | `roles/base-access/` 在 6 台全节点幂等创建（gz-04/05 cloud-init 已建则收敛，老四台首次创建，纯追加） | 一次建好可复用 role，6 台鉴权身份统一就绪，避免"2 台 admin-alex + 4 台 root"split-brain 长期化 |
| become 模式 | NOPASSWD sudoers（写 `/etc/sudoers.d/admin-alex`，visudo 校验） | 无密码交互，Ansible 体验与现有 root 一致；vault 不存 become_password |
| 本主题翻转 `ansible_user` 的节点 | **仅 gz-04 / gz-05** | 这两台是空节点，配错无业务影响；翻转即验收 admin-alex 接入闭环 |
| 现有四台节点 `ansible_user` | 保持 `root` 不变 | 翻转咬住 Jenkins `ansible-ssh-key` 凭据 + bj-01 自管理，且对象是生产节点，拆出独立任务做（见 §6） |

---

## 3. 影响范围

| 影响项 | 说明 |
|---|---|
| 现有四台节点 base-access role | **纯追加**：在 gz-01/02/03/bj-01 上首次创建 admin-alex + 授权公钥 + NOPASSWD sudoers，不删除 root、不改 root 的 authorized_keys、不改 `ansible_user`。即使本次只想动新节点，对老节点跑 base-access 也是安全的增量操作；admin-alex 建好但暂不作为连接用户 |
| Jenkins `ansible-ssh-key` 凭据 | **本主题不动**。CD Pipeline 仍以凭据里的 root 用户名 + 私钥连各节点；本主题只把该私钥对应公钥提前授权给 admin-alex，不改凭据本身，CD 行为零变化 |
| 现有四台节点 docker-daemon role | 首次执行新版 role 时：① 如本地 Docker 版本与 `docker_ce_version` 一致则 `changed=0`，否则**会触发 docker-ce apt 升级**，对运行容器有 daemon reload 风险 → 落地节奏：先在新节点 gz-04 / gz-05 上跑通新 role，再单独评估是否对现有节点执行升级；inventory 默认值锁定为现有节点已运行的版本，避免误升级 |
| 现有四台节点 `global_gateway` 网络 | 新增 `community.docker.docker_network` 任务在已存在网络上 `state=present`，幂等行为为 `changed=0`；如检测到 driver / subnet 不一致则报错而非自动重建，安全 |
| Prometheus targets | `prometheus.yml.j2` 新增 gz-04 / gz-05 的 node-exporter target，需在 bootstrap 完成、Tailscale IP 回填 inventory 后才能生效；落地节奏走两次提交（详见 §6） |
| Grafana 面板 | 现有 Node Exporter 面板自动按 `instance` label 展示新节点，无需改 dashboard JSON |
| 公网 SSH 端口 | bootstrap 期间通过公网 IP 经 admin-alex SSH 进 gz-04 / gz-05；接入完成后建议在云厂商安全组只放行 Tailscale 网段（100.64.0.0/10），关闭公网 22；具体安全组动作由人工在云厂商控制台操作，不进 Ansible 范围 |
| `vault/secrets.yml` | 新增 1 项 `tailscale_authkey`，对加密文件做增量更新 |
| 业务工作负载 | 无影响。本主题 gz-04 / gz-05 不加入 `app_nodes` / `db_*` / `redis_*` / `gateway_nodes` / `monitor_nodes` 等业务分组，site.yml 任何业务相关 play 都不会调度到这两台 |
| CI/CD Pipeline | 无影响。本主题不涉及 Jenkinsfile 与 Registry |

---

## 4. 验收标准

| 验收项 | 可验证的成功条件 |
|---|---|
| Tailscale 接入 | gz-04 / gz-05 出现在 Tailscale 控制台，状态 `connected`；从 bj-01 执行 `tailscale ping gz-04` / `tailscale ping gz-05` 30 秒内有 `pong via DERP` 或 `pong via direct connection`，丢包率 0% |
| Tailscale 内网延迟实测 | 在 runbook 与新版本架构快照中记录 gz-04/gz-05 ↔ 现有四节点的 `tailscale ping` 实测延迟均值（10 次取均），写入快照"网络互联"段 |
| admin-alex 供给（6 台全节点） | 每台 `getent passwd admin-alex` 存在；从 bj-01 经 admin-alex 公钥可 `ssh admin-alex@<节点>`；`ssh admin-alex@<节点> sudo -n true` 返回 0（NOPASSWD 生效）；`sudo visudo -cf /etc/sudoers.d/admin-alex` 校验通过 |
| 现有四台节点连接方式不变 | base-access 跑完后，gz-01/02/03/bj-01 的 `ansible_user` 仍为 root；`ansible all -m ping` 经 root 全绿；root 的 authorized_keys 未被改动；CD Pipeline 一次部署仍以 root 连接成功 |
| SSH 切换（仅新节点） | bootstrap 完成后，从 bj-01 经 `ssh admin-alex@<gz-04 Tailscale IP>` 可登录并 `sudo -n true`；gz-04/05 的 `ansible_user` 已是 admin-alex，`ansible gz-04,gz-05 -m ping` 经 admin-alex + become 全绿；公网 IP SSH 不再使用（关闭与否走人工节奏） |
| Docker 运行时 | `docker info` 输出 Server Version 与 `docker_ce_version` 一致；`docker compose version` 与 `docker_compose_plugin_version` 一致 |
| `global_gateway` 网络 | gz-04 / gz-05 上 `docker network ls` 可见 `global_gateway`；现有四台节点重跑 docker-daemon role 后 `global_gateway` driver / subnet 与 role 期望一致，`changed=0` |
| Prometheus 抓取 | Prometheus UI `up{instance="<gz-04 Tailscale IP>:9100"} == 1` 与 `up{instance="<gz-05 Tailscale IP>:9100"} == 1` |
| mysqld-exporter 边界 | gz-04 / gz-05 上 `docker ps` 不可见 `mysqld-exporter` 容器；Prometheus targets 中 mysqld-exporter job 仅含 gz-02 / gz-03 / bj-01 |
| Grafana 主机面板 | 在现有 Node Exporter dashboard 中能选到 gz-04 / gz-05，CPU / 内存 / 磁盘 / 网络指标曲线连续无断点 |
| Ansible 幂等 | `ansible-playbook playbooks/setup_access.yml` / `setup_tailscale.yml --limit gz-04,gz-05` 二次执行 `changed=0 failed=0`；`playbooks/site.yml --limit managed_nodes --check` 全节点 `changed=0 failed=0`（覆盖 base-access、docker-daemon、node-exporter） |
| 业务隔离 | `ansible-inventory --list` 中 gz-04 / gz-05 仅出现在 `managed_nodes`、`exporter_nodes` 两个分组；不出现在 `app_nodes` / `db_*` / `redis_*` / `gateway_nodes` / `monitor_nodes` / `mysqld_exporter_nodes` |
| 敏感信息 | `vault/secrets.yml` 保持 AES256 加密；`grep -rEi "tskey-|authkey" inventory roles playbooks Docs` 无明文凭据；ansible 日志中 `tailscale_authkey` 全部脱敏（task 加 `no_log: true`） |

---

## 5. 待确认问题

下表的"占位值"在 proposal 阶段写入 `group_vars/all.yml` 与 inventory，**runbook 起步前必须由人工实地确认后回填**——这些值取决于云厂商选型与节点开机后的实测信息。

| 序号 | 待确认项 | 影响 | 触发节奏 |
|---|---|---|---|
| Q1 | gz-04 / gz-05 云厂商与城市内可用区 | 决定 `inventory/hosts.yml` 主机变量中是否需要加 `docker_registry_mirrors`（与现有 gz-02 腾讯云、gz-03 火山引擎做法对齐） | runbook 起步前 |
| Q2 | gz-04 / gz-05 系统发行版与版本（Debian / Ubuntu / 版本号） | 决定 `tailscale_apt_distribution`、Docker 官方 apt 仓库 codename、`extra_args` PEP 668 判断分支 | runbook 起步前 |
| Q3 | gz-04 / gz-05 具体规格（CPU / 内存 / 磁盘 / 带宽） | 路线图说"广州 4G 节点"，需实测确认；影响 [`k3s-stateless`](../scheme/phase-1-architecture-upgrade.md#k3s-stateless) 主题的资源边界 | runbook 起步前；架构快照"节点总览"段必须填实际值 |
| Q4 | 公网 IP（用于 bootstrap） | bootstrap playbook `-e ansible_host=<...>` 命令的实际值 | bootstrap 当天 |
| Q5 | 接入后是否立即关闭公网 22 端口 | 安全收紧节奏；建议是"接入 + 验收完成后立即关"，但操作在云厂商控制台进行，需要人工节奏决定 | 验收完成后 |
| Q6 | `docker_ce_version` 锁定到哪个具体版本号 | 决定现有四台节点首次跑新版 docker-daemon role 是否触发升级；建议先在 bj-01 运行 `dpkg -l docker-ce` 取当前版本作为锁定基线，避免误升级 | runbook 起步前 |
| Q7 | `admin_alex_authorized_keys` 收哪几把公钥 | 至少含运维操作机 admin-alex 公钥 + Jenkins `ansible-ssh-key` 凭据对应公钥；从现有 root `~/.ssh/authorized_keys` 与 Jenkins 凭据导出后回填 | runbook 起步前 |

待确认项不构成 proposal 阶段的阻塞——proposal 当前框架是稳定的，Q1-Q6 是落地动作里需要回填的具体数值，不影响主题方案设计。

---

## 6. 范围边界与后续 cutover（拆出项）

### 6.1 本主题做什么 / 不做什么

| 维度 | 本主题（v1.8）做 | 拆出到后续任务做 |
|---|---|---|
| admin-alex 用户 | 6 台全节点幂等创建 + 授权公钥 + NOPASSWD sudoers（纯追加） | — |
| `ansible_user` 翻转 | 仅 gz-04 / gz-05 | gz-01 / gz-02 / gz-03 / bj-01 四台翻转为 admin-alex |
| Jenkins `ansible-ssh-key` 凭据 | 不动（仍 root），仅把其公钥提前授权给 admin-alex | 凭据用户名改为 admin-alex（私钥可沿用） |
| root SSH 登录 | 保留（作为 cutover 期间的回退通道） | 验收稳定后评估关闭 4 台老节点 root 直登 |

### 6.2 为什么 4 台老节点翻转必须拆出

| 耦合 / 风险 | 说明 |
|---|---|
| Jenkins 凭据双真相源 | `Jenkinsfile` 用 `-u "$ANSIBLE_USER"` 覆盖 inventory，SSH 用户来自 `ansible-ssh-key` 凭据（当前 root）。**只改 inventory 不改凭据，下一次 webhook 触发的 CD 立刻断**。该凭据是 [`cicd-state-iac`](../scheme/phase-1-architecture-upgrade.md#cicd-state-iac) 主题要专门治理的手工状态，宜与其一起做或单列小任务 |
| bj-01 自管理 | bj-01 既是控制节点又被自己管理，为兼容 Jenkins 容器执行走 SSH 连自己（见 `Docs/retrospectives/v1.5-retrospective.md`）。翻转时 Jenkins 容器的密钥必须进 bj-01 admin-alex 的 authorized_keys 且 sudoers 就绪，否则容器内部署连自己失败 |
| 生产锁定风险 | gz-01/02/03 带真实业务流量，gz-03 是 MySQL Master。sudoers / key 配错即生产事故。base-access 的"纯追加创建"已在本主题验证 admin-alex 可登录可 sudo，cutover 任务只需翻转用户名，风险显著降低，但仍需独立回滚窗口 |

### 6.3 后续 cutover 任务 checklist（拆出执行）

> 落地时建议并入 [`cicd-state-iac`](../scheme/phase-1-architecture-upgrade.md#cicd-state-iac) 主题或单列一个带 runbook 的小任务；前置条件是本主题 base-access 已在 4 台老节点验收通过（admin-alex 可登录 + `sudo -n true` 返回 0）。

1. 预检：4 台老节点 `ssh admin-alex@<node> sudo -n true` 全部返回 0
2. 翻转 inventory：gz-01/02/03/bj-01 的 `ansible_user` 改 admin-alex
3. 手工跑一次 `ansible all -m ping` + `site.yml --check`，确认 admin-alex + become 全绿
4. 更新 Jenkins `ansible-ssh-key` 凭据：用户名 root → admin-alex（私钥沿用，其公钥已在本主题授权）
5. 触发一次 CD（或 `Ansible Check` stage）验证 Jenkins 经 admin-alex 部署成功，**重点验 bj-01 自连**
6. 稳定观察一段时间后，评估关闭 4 台老节点 root SSH 直登（保留紧急 break-glass 通道）
7. 回滚预案：任一步失败，inventory `ansible_user` 改回 root + Jenkins 凭据用户名改回 root 即恢复（root 通道全程保留）

---

## 7. 后续推进

| 阶段 | 动作 | 触发条件 |
|---|---|---|
| **proposal 评审** | 当前文档评审通过 | 本次完成 |
| **路线图修订** | 同 PR 提交 `phase-1-architecture-upgrade.md` 中 `six-node-onboarding` 与 `base-reproducibility-fix` 两段的描述修订 | 与本 proposal 同 PR |
| **节点准备 D-2** | 人工 / cloud-init 在 gz-04 / gz-05 创建 admin-alex 用户、写 SSH key、配 NOPASSWD sudoers | 主题启动前 |
| **节点准备 D-1** | 回填 §5 的 Q1-Q4、Q6、Q7 实测值；Tailscale 控制台生成 reusable + preauthorized + tag:server 的 auth key，写入 vault `tailscale_authkey`；收集 `admin_alex_authorized_keys` 公钥列表 | 主题启动前 |
| **D-0 开工** | 按 `.cursor/rules/10-docs-workflow.mdc` §1 九步流程：role 改造（base-access + tailscale + docker-daemon）→ runbook → base-access 收敛 6 台 admin-alex（纯追加）→ bootstrap 执行（公网 IP）→ Tailscale IP 回填 inventory（**第二次提交**）→ Prometheus targets 更新（**第三次提交**）→ retrospective → 架构快照 → tag | proposal 评审 + 节点准备完成 |
| **后续接力（K3s）** | 进入 [`k3s-stateless`](../scheme/phase-1-architecture-upgrade.md#k3s-stateless) 主题，gz-04 / gz-05 直接作为 K3s 控制面 + worker | 本主题 retrospective 通过 |
| **后续接力（鉴权 cutover）** | 按 §6.3 checklist 翻转 4 台老节点 `ansible_user` + 切换 Jenkins 凭据 + 评估关 root 登录；并入 [`cicd-state-iac`](../scheme/phase-1-architecture-upgrade.md#cicd-state-iac) 或单列小任务 | 本主题 base-access 在 4 台老节点验收通过后 |

**为什么 bootstrap 拆三次提交**：

1. **第一次**（role + playbook + inventory 框架）：合入新 role 与 playbook，inventory 加 gz-04 / gz-05 主机段（`ansible_host` 暂用占位 `0.0.0.0`，明确不工作；只在 `managed_nodes` 分组占位，不进 `exporter_nodes`），用于 Code review 与 runbook 落稿
2. **第二次**（bootstrap 完成，IP 回填）：执行 bootstrap，把 inventory 中 `ansible_host` 改为实测 Tailscale IP，把 gz-04 / gz-05 加入 `exporter_nodes` 分组
3. **第三次**（Prometheus 抓取生效）：更新 `prometheus.yml.j2` 加 target，重跑 monitor-stack role，验证 `up == 1`

三次提交的目的是让"配置在 git 中的状态"始终与"集群在线运行的状态"对得上，避免出现"inventory 写了但节点还没接入"的不一致窗口。

如本主题落地过程中发现新的子问题（例如 Docker apt 仓库在某云厂商被墙、admin-alex sudoers 与云厂商默认配置冲突等），按 [`.cursor/rules/10-docs-workflow.mdc`](../../.cursor/rules/10-docs-workflow.mdc) §6 规则升级为 `Docs/reviews/v1.8-<slug>.md`，再链接到本主题。
