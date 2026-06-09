# 架构草案 — K3s 迁移无状态服务

> **状态**：草案讨论中
> **目标主题**：[`k3s-stateless`](../scheme/phase-1-architecture-upgrade.md#k3s-stateless)（当前目标版本以路线图为准）
> **前置主题**：[`six-node-onboarding`](../scheme/phase-1-architecture-upgrade.md#six-node-onboarding)（已落地 v1.8；gz-04 / gz-05 已接入 Tailscale + Docker Engine + node-exporter，admin-alex 鉴权就绪）
> **节点增量**：v1.8 落地后新增腾讯云·广州 4C4G 节点 **gz-06**（常驻 K3s worker），onboarding（Tailscale `--hostname gz-06` + base-access + docker-daemon + node-exporter，复用 v1.8 流程）是本主题 K3s 安装的前置；引入动因见 §1.1
> **后置主题**：[`helm-chart-mgmt`](../scheme/phase-1-architecture-upgrade.md#helm-chart-mgmt)（把本主题的裸 YAML 封装成 Helm Chart）、[`service-migration`](../scheme/phase-1-architecture-upgrade.md#service-migration)（gz-01 入口/监控迁 gz-04）、[`proxysql-ha`](../scheme/phase-1-architecture-upgrade.md#proxysql-ha)（K3s 放大的 ProxySQL 单点）
> **关联红线**：有状态服务（MySQL / Redis / ProxySQL）不进 K3s，继续 Compose 承载（见 `.cursor/rules/00-project-context.mdc`）

---

## 1. 设计目标

把当前以 Docker Compose 跑在 gz-03（`ruoyi-admin-1`）和 gz-02（`ruoyi-admin-2`）上的 ruoyi 无状态后端，迁移到 **gz-05 server+agent + gz-04 worker + gz-06 worker** 组成的 K3s 三节点集群，验证 Kubernetes 基础工作流，并为 [`helm-chart-mgmt`](../scheme/phase-1-architecture-upgrade.md#helm-chart-mgmt)、[`loki-logging`](../scheme/phase-1-architecture-upgrade.md#loki-logging) 打基础。本主题落地后的边界：

- **集群层**：gz-05 单 control-plane（K3s server，同时可调度业务 Pod），gz-04 / gz-06 作为 agent worker；节点间 apiserver / flannel 流量强制走 Tailscale，不依赖跨云公网直连
- **应用层**：ruoyi 后端以 `Deployment` 2 副本运行，通过 `topologySpreadConstraints` 分布到 gz-04 / gz-06 两个专职 worker；通过 NodePort + gz-01 Nginx 对外提供服务，具备滚动更新与 `kubectl rollout undo` 回滚能力
- **有状态边界**：MySQL / Redis / ProxySQL **不迁入 K3s**，仍由 Compose 承载；K3s 中的 ruoyi Pod 经 Tailscale 内网访问现有 gz-03 ProxySQL（`100.92.5.116:6033`）与 Redis Sentinel（`mymaster`）
- **控制入口层**：kubectl 与 YAML 下发统一从 bj-01 发起（与现有 Ansible / Jenkins 控制入口一致），不在业务节点本地操作
- **回滚窗口**：迁移验证通过后，gz-02 / gz-03 的 ruoyi Compose 副本**停而不删**，保留 3 天只读回滚窗口，期满后从 `app_nodes` 摘除

完成本主题后，集群首次具备容器编排能力，gz-04 / gz-05 / gz-06 的算力被业务真正利用；ruoyi 的发布从"Ansible 改 compose + recreate"升级为"K8s 滚动更新 + 声明式回滚"。

> **范围红线**：不引入抢占式实例 / Cluster Autoscaler / 跨云动态扩缩容（路线图已明确）；本主题只验证固定三节点 K3s、固定节点调度、滚动更新、回滚与三类故障演练。gz-06 是**常驻** worker，不做"用完销毁、下次重建"的 ephemeral 节点——常驻线上服务下 ruoyi 没有"用完"时刻，ephemeral 既不省钱（gz-06 已购、销毁式还要持续按量付费）又会把"重建即换 IP / 重新入 tailnet / Prometheus target 漂移"的复杂度请回来。

### 1.1 gz-06 引入动因（为什么是常驻 worker，不是 ephemeral）

v1.8 落地后新增腾讯云·广州 4C4G 节点 gz-06，定位常驻 K3s worker。决策链路：

- **要解决的真实矛盾**：路线图 v1.11 [`service-migration`](../scheme/phase-1-architecture-upgrade.md#service-migration) 要 gz-04 接管 gz-01 的 Nginx 入口 + 主监控（Prometheus/Grafana/blackbox）。若 gz-04 同时仍是唯一 worker，2C4G 上叠加监控栈（~1.5G）与 ruoyi Pod（limits 1.5Gi）会内存吃紧，v1.12 再叠 Loki 直接超预算。
- **为什么是常驻而非 ephemeral**：增加一台**常驻** worker，让 v1.11 时 gz-04 可退出 worker 池只承载入口/监控，ruoyi 副本落到 gz-06 + gz-05；全程不引入抢占式 / 动态入网 / 节点回收复杂度，踩在路线图"有意识不做动态扩缩容"红线安全侧。
- **不要误判 gz-06 是"内存救星"**：gz-06 内存仍为 4G（与 gz-04/gz-05 同档），本设计的绑定约束一直是内存而非 CPU。gz-06 的价值是"多一台 4G 节点摊负载" + 4 核 CPU 给 JVM/Prometheus 多点余量，不是单台容量更大。
- **节点封顶**：至此 gz-01～gz-06 + bj-01 共 7 台角色解耦完整，phase-1 余下版本只做"用透现有节点"，不再扩节点；再扩须先有"现有节点哪个具体能力被卡住"的硬理由（`.cursor/rules/00-project-context.mdc` 节点扩容红线）。

---

## 2. 方案范围

### 2.1 新增 Ansible Role

| Role | 职责 | 关键任务 |
|---|---|---|
| `roles/k3s-server/` | 在 gz-05 安装 K3s server，可调度业务 Pod | ① `docker --version` 式能力探测：`k3s` 已装且 `systemctl is-active k3s` 为 active 则跳过安装；② 渲染 `/etc/rancher/k3s/registries.yaml`（私有 Registry，见 §2.4）；③ 经离线/镜像化的 `install.sh` 安装 K3s，使用 vault 中固定 `k3s_token` 初始化 server，启动参数见下；④ 把 `/etc/rancher/k3s/k3s.yaml` 拉回 bj-01、改 `server:` 为 gz-05 Tailscale IP，落到 `~/.kube/k3s-ruoyi.yaml` |
| `roles/k3s-agent/` | 在 gz-04 / gz-06 以 agent 加入 gz-05 集群（`hosts: k3s_agent_nodes`，两台同一套任务）| ① 同款能力探测（`k3s-agent` 服务已 active 则跳过）；② 渲染同一份 `registries.yaml`；③ 以 `K3S_URL=https://<gz-05 Tailscale IP>:6443`、`K3S_TOKEN={{ k3s_token }}` 经 `install.sh` 加入；④ 启动参数指定 `--node-ip`（各自 Tailscale IP）/ `--flannel-iface=tailscale0` |

**K3s server 启动参数（gz-05）**：

```bash
--node-ip=100.106.130.117 \           # 强制以 Tailscale IP 作为节点地址
--flannel-iface=tailscale0 \          # flannel VXLAN 走 Tailscale 网卡（跨云关键，决策 D1）
--disable=traefik \                   # 禁用内置 Ingress，本主题走 NodePort（决策 D4）
--write-kubeconfig-mode=0644 \        # 便于 Ansible 读取 kubeconfig
--token={{ k3s_token }}               # 固定 token，保证 agent join 幂等可重复
```

> ServiceLB（Klipper）、local-path、CoreDNS 保留默认；本主题不申请 PVC，local-path 仅作为后续 [`loki-logging`](../scheme/phase-1-architecture-upgrade.md#loki-logging) 的伏笔，不在此启用。

### 2.2 应用编排：裸 Kubernetes YAML（模板化）

ruoyi 后端的 K8s 对象以 Jinja 模板渲染、从 **bj-01** 经 `kubernetes.core.k8s` 模块 apply（决策 D5）。新增 `roles/k3s-ruoyi/`，`hosts` 指向 `ops_nodes`（bj-01），用 `delegate_to` 模式对集群操作：

| 对象 | 文件 | 要点 |
|---|---|---|
| `Namespace` | `namespace.yaml.j2` | `ruoyi`，所有对象归属此 ns |
| `Secret` | `secret.yaml.j2` | DB 账号/密码、Redis 密码，**值来自 vault**（`ruoyi_datasource_username` / `ruoyi_datasource_password` / `ruoyi_redis_password`），Ansible 渲染时 `b64encode`（决策 D6） |
| `ConfigMap` | `configmap.yaml.j2` | 非敏感环境变量：`JAVA_TOOL_OPTIONS`、ProxySQL host/port、Redis Sentinel master name 与 nodes 列表 |
| `Deployment` | `deployment.yaml.j2` | `replicas: 2`；`image: {{ registry_host }}/{{ ruoyi_image_name }}:{{ ruoyi_image_tag }}`；`rollingUpdate`（`maxSurge:1 maxUnavailable:0`）；`topologySpreadConstraints` 按 `kubernetes.io/hostname` 跨节点打散；`resources` requests/limits（见 §2.6）；`readinessProbe` / `livenessProbe`（基线 TCP 8080，详见 §5 Q3）；env 从 Secret + ConfigMap 注入 |
| `Service` | `service.yaml.j2` | `type: NodePort`，`port: 8080` → `nodePort: 30080`（决策 D4） |

> 不在 K3s 内重建 ProxySQL / Redis Sentinel。ruoyi 容器原本通过 Compose 环境变量拼 `jdbc:mysql://${DB_HOST}:${DB_PORT}/ry-vue...`，迁移后这些值改由 ConfigMap/Secret 注入，**连接目标不变**（仍是 gz-03 ProxySQL 与现有 Sentinel）。

### 2.3 新增 Playbook

| Playbook | 入口 | 触发场景 |
|---|---|---|
| `playbooks/setup_k3s.yml` | 先 `k3s-server` on `k3s_server_nodes`，再 `k3s-agent` on `k3s_agent_nodes` | 集群安装/收敛；server 必须先于 agent（agent 依赖 server token 与 6443 可达） |
| `playbooks/setup_k3s_app.yml` | `k3s-ruoyi` on `ops_nodes`（bj-01 经 kubeconfig 操作集群） | ruoyi manifest apply / 滚动更新；日常重跑幂等 |

两者按依赖顺序 `import_playbook` 进 `site.yml`（在业务 play 之后）。bootstrap 节奏沿用三次提交模式（详见 §7）。

### 2.4 私有 Registry 接入（K3s containerd）

K3s 用 containerd，**不读 Docker 的 `daemon.json`**，因此 v1.8 配的 `insecure-registries` 对 K3s 无效。必须单独渲染 `/etc/rancher/k3s/registries.yaml`（决策 D3）：

```yaml
mirrors:
  "{{ registry_host }}":           # 100.118.69.78:5000
    endpoint:
      - "http://{{ registry_host }}"   # 私有 Registry 是 HTTP，显式 http:// 端点
configs:
  "{{ registry_host }}":
    auth:
      username: "{{ registry_auth_username }}"   # 复用 vault 现有凭据
      password: "{{ registry_auth_password }}"
```

> 该任务 `no_log: true`；复用 v1.8 已存在的 `registry_auth_username/password`，**vault 不新增 Registry 相关条目**。

### 2.5 inventory / vault / group_vars 变更

**inventory/hosts.yml** 新增分组（gz-04 / gz-05 已在 `managed_nodes` / `exporter_nodes`，本次叠加 K3s 分组，**不进 `app_nodes`**）。gz-06 需先按 v1.8 流程完成 onboarding（新增主机段：公网 IP bootstrap → 回填 Tailscale IP；加入 `managed_nodes` / `exporter_nodes`），再叠加 `k3s_agent_nodes`：

```yaml
k3s_server_nodes:
  hosts:
    gz-05:
k3s_agent_nodes:
  hosts:
    gz-04:
    gz-06:           # v1.8 后新增的常驻 worker
k3s_nodes:
  children:
    k3s_server_nodes:
    k3s_agent_nodes:
```

**vault/secrets.yml** 新增 1 项：

| 变量名 | 用途 | 形态 |
|---|---|---|
| `k3s_token` | K3s server/agent 集群加入共享 token | 随机长字符串；固定写入保证 agent join 幂等可重复，避免每次读 server 动态 token 产生的非确定性 |

**inventory/group_vars/all.yml** 新增非敏感变量：

```yaml
# ── K3s（k3s-stateless 主题）──────────────────────────────────────
k3s_server_host: "{{ hostvars[groups['k3s_server_nodes'][0]].ansible_host }}"  # gz-05 Tailscale IP
k3s_api_port: 6443
ruoyi_k3s_namespace: ruoyi
ruoyi_k3s_nodeport: 30080      # gz-01 Nginx upstream 指向各 K3s 节点此端口
ruoyi_k3s_replicas: 2
```

### 2.6 资源预算（4G 内存节点）

三节点内存均为 4G（gz-04 / gz-05 为 2C4G，gz-06 为 4C4G），内存是主要约束。本主题 ruoyi 2 副本经 `topologySpreadConstraints` 落到 gz-04 / gz-06 两个专职 worker，gz-05 作为 control-plane 稳态不承载业务 Pod：

| 节点 | 承载 | 内存账（约）|
|---|---|---|
| gz-05（2C4G）| K3s server + agent（control-plane，稳态无业务 Pod）| server 开销 ~0.5-1G / 4G，余量充足 |
| gz-04（2C4G）| agent + 1 ruoyi Pod | agent ~0.5G + Pod ~1.5G ≈ 2G / 4G |
| gz-06（4C4G）| agent + 1 ruoyi Pod | 同上 ≈ 2G / 4G，4 核 CPU 给 JVM 更宽裕 |

> 相比原双节点方案（gz-05 server+agent **叠** 1 个 Pod 导致 4G 偏紧、需考虑降 JVM 堆），三节点把 Pod 从 control-plane 挪到专职 worker，gz-05 不再叠 Pod，§5 Q2「是否降 JVM 堆」的压力显著缓解。

| 项 | 当前 Compose | K3s 建议 | 说明 |
|---|---|---|---|
| JVM 堆 | `-Xms1024m -Xmx1024m` + Metaspace 256m | 维持 `-Xmx1024m` | Pod 落 gz-04/gz-06、不再与 server 抢内存，1024m 可行；实测 OOM 再降，结论写 retrospective |
| Pod requests | — | `memory: 1Gi, cpu: 500m` | 保证调度落点有足量内存 |
| Pod limits | — | `memory: 1.5Gi, cpu: "1"` | 防止单 Pod 拖垮节点；超限 OOMKill 由 K8s 兜底 |

runbook 中必须记录迁移前后 gz-04 / gz-05 / gz-06 的 `free -m` 与 `kubectl top nodes` 基线。

### 2.7 决策点对比表

#### K3s 安装与 IaC 方式（决策 D2）

| 方案 | 优点 | 缺点 | 推荐 |
|---|---|---|---|
| 官方 `install.sh` + Ansible shell 能力探测封装 | 安装过程透明可复述；与 v1.8 docker-daemon 的能力探测风格一致；参数完全由我们控制 | 需自己保证幂等（靠 `systemctl is-active` 探测） | **推荐** |
| 社区 `k3s-ansible` galaxy role | 功能全、覆盖 HA/升级 | 黑盒、抽象层厚，违背项目"能复述"红线；对单 server 场景过重 | 不选 |

#### 网关接入方式（决策 D4）

| 方案 | 优点 | 缺点 | 推荐 |
|---|---|---|---|
| 禁用 Traefik，NodePort + gz-01 Nginx upstream | 链路最短、回滚清晰（Nginx upstream 切回 Compose 即可）；只迁一个后端不需要 Ingress 抽象 | NodePort 端口需固定管理 | **推荐** |
| 保留 Traefik，用 Ingress | 更"云原生"、贴近生产 | 引入 Traefik 配置 + 证书是额外变量，本主题只迁一个服务，收益不抵复杂度 | 留给后续主题 |

#### 操作入口与 YAML 下发（决策 D5）

| 项 | 选择 | 理由 |
|---|---|---|
| kubectl/apply 发起机 | **bj-01** | 与现有 Ansible / Jenkins / Git 控制入口统一；kubeconfig 从 gz-05 拷回并改 server 为 Tailscale IP |
| YAML 下发方式 | `kubernetes.core.k8s` 模块 | Ansible 原生、支持 check mode、幂等好；需在 bj-01 装 `kubernetes` Python SDK |
| 控制面位置 | gz-05（K3s server，**不变**） | 这是集群事实，与"从哪操作"是两回事 |

#### Secret 管理（决策 D6）

| 方案 | 优点 | 缺点 | 推荐 |
|---|---|---|---|
| vault → Ansible 渲染 Secret manifest | 真值源头仍是 vault（单一事实源）；不引入新组件 | K8s Secret 仅 base64 非加密，依赖 etcd/节点访问控制 | **推荐** |
| SOPS / sealed-secrets | etcd 内也加密 | 对当前规模 overkill，新增密钥管理复杂度 | 不选 |

---

## 3. 影响范围

| 影响项 | 说明 |
|---|---|
| gz-05 / gz-04 / gz-06 运行时 | 新增 K3s server / agent 进程与 containerd（K3s 自带，**与 v1.8 装的 Docker Engine 共存**，互不接管）；新增 flannel `cni0` / VXLAN 接口，绑定 `tailscale0` |
| gz-06 接入（前置）| 新节点需先按 v1.8 流程 onboarding：Tailscale 入网（`--hostname gz-06`）+ base-access（admin-alex）+ docker-daemon + node-exporter，并入 `managed_nodes` / `exporter_nodes`；完成后才叠加 K3s 分组 |
| gz-04 / gz-05 / gz-06 inventory 角色 | 叠加 `k3s_server_nodes` / `k3s_agent_nodes` / `k3s_nodes`；**仍不进 `app_nodes`**——Compose 业务 play 不会调度到这三台，避免与 K3s 工作负载冲突 |
| 现有 ruoyi Compose（gz-02 / gz-03） | 迁移验证通过后停而不删（决策 D7）；3 天回滚窗口内可一键 `docker compose up -d` 恢复；期满从 `app_nodes` 摘除并清理 |
| gz-01 Nginx | upstream 由指向 gz-02 / gz-03 的 Compose ruoyi（`:8080`）改为指向 gz-04 / gz-05 的 K3s NodePort（`:30080`）；保留切回 Compose upstream 的回滚配置 |
| gz-03 ProxySQL / Redis Sentinel | **无配置变更**；仅新增来自 K3s Pod 的连接来源。ProxySQL 单点被副本数放大的问题登记给 [`proxysql-ha`](../scheme/phase-1-architecture-upgrade.md#proxysql-ha)，不在本主题修 |
| bj-01 控制节点 | 新增 `~/.kube/k3s-ruoyi.yaml` kubeconfig 与 `kubernetes` Python SDK；新增 K3s 相关 playbook 与 role |
| Prometheus / 监控 | 本主题**不展开** K3s 指标采集（kube-state-metrics 等留给 [`observability-integration`](../scheme/phase-1-architecture-upgrade.md#observability-integration)）；gz-04 / gz-05 的 node-exporter 主机指标不变 |
| `vault/secrets.yml` | 新增 1 项 `k3s_token`，增量更新加密文件 |
| CI/CD Pipeline | 本主题**不改 Jenkinsfile**；Jenkins 经 Helm 部署 K3s 应用是 [`helm-chart-mgmt`](../scheme/phase-1-architecture-upgrade.md#helm-chart-mgmt) 的范围。本主题镜像 tag 沿用现有 `latest`，apply 由人工/Ansible 在 bj-01 触发 |
| 公网域名可用性 | 切换瞬间走 Nginx upstream 热切换；迁移期间保留 Compose 后端在线，确保灰度与回滚 |

---

## 4. 验收标准

| 验收项 | 可验证的成功条件 |
|---|---|
| 集群就绪 | bj-01 上 `kubectl get nodes -o wide` 显示 gz-04 / gz-05 / gz-06 均 `Ready`，且 `INTERNAL-IP` 为各自 Tailscale IP（验证 D1 生效） |
| 跨云通信走 Tailscale | gz-05 / gz-04 `k3s` 进程参数含 `--flannel-iface=tailscale0`；停掉任一节点公网出口（或 `tailscale ping` 验证）后集群仍 `Ready`，apiserver 经 6443 Tailscale 可达 |
| 私有 Registry 拉取 | `kubectl get pods -n ruoyi` 无 `ErrImagePull` / `ImagePullBackOff`；`crictl images`（在节点）可见 `100.118.69.78:5000/ruoyi:latest` |
| 副本跨节点分布 | `kubectl get pods -n ruoyi -o wide` 显示 2 个 Pod `Running`，分别落在 gz-04 / gz-06 两个专职 worker（`topologySpreadConstraints` 生效）；gz-05 control-plane 稳态无业务 Pod |
| 公网可达 | 经 gz-01 Nginx 的公网域名能访问 ruoyi 后端（登录页/接口返回正常），链路为 域名 → gz-01 Nginx → K3s NodePort 30080 → Pod |
| 数据/缓存连通 | ruoyi Pod 日志无 DB / Redis 连接异常；业务读写正常（经 ProxySQL `100.92.5.116:6033` 与 Redis Sentinel `mymaster`） |
| 滚动更新不中断 | `kubectl set image` 或改 tag 后 `kubectl rollout status` 成功，期间公网访问无 5xx 中断（`maxUnavailable:0`） |
| 回滚可用 | `kubectl rollout undo deployment/ruoyi -n ruoyi` 能回到上一 ReplicaSet，Pod 恢复旧版本 |
| 故障演练（3 类） | ① 发布不存在的镜像 tag → 观察 `ImagePullBackOff` 并 `kubectl describe pod` 定位；② 配错 readinessProbe → `rollout` 卡住、`kubectl rollout status` 超时、events 可定位；③ `kubectl drain gz-04` → Pod 迁到 gz-06（或 gz-05 可调度）、业务可用性影响记录到 `Docs/drills/`（按 §7 命名约定） |
| Compose 回滚窗口 | 迁移后 3 天内 gz-02 / gz-03 Compose ruoyi 可一键恢复；期满后 `app_nodes` 不再含这两台的 ruoyi 副本 |
| Ansible 幂等 | `playbooks/setup_k3s.yml --limit k3s_nodes` 二次执行 `changed=0 failed=0`；`setup_k3s_app.yml --check` 在 bj-01 `changed=0 failed=0` |
| 敏感信息 | `vault/secrets.yml` 保持 AES256；`grep -rEi "k3s_token|password|tskey-" inventory roles playbooks Docs` 无明文；registries.yaml 与 token 相关 task `no_log: true` |
| 概念可复述 | 能解释 Deployment → ReplicaSet → Pod → Service → NodePort 的层级关系，以及"为什么有状态服务不进 K3s" |

---

## 5. 待确认问题

| 序号 | 待确认项 | 当前倾向 / 验证方式 |
|---|---|---|
| Q1 | **K3s Pod 出站访问 Tailscale 上的 ProxySQL / Redis Sentinel 是否需额外 SNAT/路由**（与 D1 强相关） | flannel VXLAN Pod 默认经节点 SNAT 出站，节点本身在 tailnet，理论可达；但跨云 + Tailscale + VXLAN 叠加需 **PoC 实测**：装完集群先用一次性 `kubectl run --rm -it busybox` 测 `nc -vz 100.92.5.116 6033` 与 Sentinel 26379，再迁正式 Deployment |
| Q2 | gz-05 叠加 server 后 ruoyi JVM 堆是否需从 `-Xmx1024m` 降到 `768m` | 先按 1024m + limits 1.5Gi 部署，观察 `kubectl top` 与 OOM 情况；若 gz-05 内存紧张则降堆，结论写入 retrospective |
| Q3 | ruoyi readiness/liveness probe 用 TCP 还是 httpGet | 基线用 `tcpSocket: 8080`（确定能用）；RuoYi 若未开 actuator 健康端点则不强行 httpGet，是否有可用 HTTP 健康路径在 PoC 阶段确认 |
| Q4 | NodePort 30080 是否与 gz-04 / gz-05 现有端口冲突 | 两台目前仅 node-exporter（9100），30080 在 NodePort 默认段（30000-32767）内无冲突；落地前 `ss -ltnp` 复核 |
| Q5 | `install.sh` 在 gz-04（百度云）/ gz-05（腾讯云）的外网可达性 | K3s 安装脚本与二进制走 `get.k3s.io` / GitHub；若某云出口受限，改用镜像源或预下载二进制离线安装，按 §7 升级为 review |
| Q6 | ruoyi 镜像 tag 策略 | 本主题沿用 `latest`；精确 tag（`BUILD_NUMBER-SHA`）注入与 Jenkins 集成留给 [`helm-chart-mgmt`](../scheme/phase-1-architecture-upgrade.md#helm-chart-mgmt) |

---

## 6. 范围边界（本主题做什么 / 不做什么）

| 维度 | 本主题（v1.9）做 | 拆出到后续主题 |
|---|---|---|
| K3s 集群 | gz-05 server + gz-04 / gz-06 agent 三节点安装 | 控制面 HA / etcd quorum（当前不做：三节点中仅 gz-05 是 server，agent 不计入 etcd quorum，凑不齐 3 server） |
| worker 拓扑 | gz-04 + gz-06 两个专职 worker 承载 ruoyi 2 副本 | v1.11 gz-04 退出 worker 池接 Nginx/监控，副本落 gz-06 + gz-05（[`service-migration`](../scheme/phase-1-architecture-upgrade.md#service-migration)，退池前以 `kubectl top` 实测确认双 worker 容量） |
| 应用编排 | ruoyi 裸 YAML（Deployment/Service/ConfigMap/Secret） | Helm Chart 化、多环境 values（[`helm-chart-mgmt`](../scheme/phase-1-architecture-upgrade.md#helm-chart-mgmt)） |
| 网关 | gz-01 Nginx upstream 指 NodePort | gz-01 入口迁 gz-04 + Ingress 化（[`service-migration`](../scheme/phase-1-architecture-upgrade.md#service-migration)） |
| 数据库访问 | Pod 直连 gz-03 单 ProxySQL | ProxySQL 双实例 + JDBC 多端点 failover（[`proxysql-ha`](../scheme/phase-1-architecture-upgrade.md#proxysql-ha)） |
| 持久化 | 不申请 PVC | Loki 用 local-path PVC（[`loki-logging`](../scheme/phase-1-architecture-upgrade.md#loki-logging)） |
| K3s 监控 | 不接 kube-state-metrics | K3s Pod / rollout 告警与 Dashboard（[`observability-integration`](../scheme/phase-1-architecture-upgrade.md#observability-integration)） |
| CI/CD | 不改 Jenkinsfile，apply 人工/Ansible 触发 | Jenkins 经 Helm 部署（[`helm-chart-mgmt`](../scheme/phase-1-architecture-upgrade.md#helm-chart-mgmt)） |

---

## 7. 后续推进

| 阶段 | 动作 | 触发条件 |
|---|---|---|
| **proposal 评审** | 当前文档评审通过 | 本次完成 |
| **路线图修订** | 如需把任务 4 描述中"Ingress 或 NodePort"明确为 NodePort，同 PR 修订 `phase-1-architecture-upgrade.md`；slug / 目标版本不变 | 与本 proposal 同 PR |
| **gz-06 onboarding（D-2）** | 按 v1.8 流程把 gz-06 接入：`setup_tailscale.yml`（`--hostname gz-06`，经公网 IP）→ 回填 Tailscale IP → `setup_access.yml` → `setup_infra.yml` → `setup_monitor.yml`（node-exporter + Prometheus 纳入新 target）；旧四台 Tailscale 设备名顺手去 `ops-lab-` 前缀对齐 clean 命名 | K3s 安装前置 |
| **PoC（D-1）** | 装最小 K3s + 一次性 busybox Pod 验证 Q1（Tailscale 内网到 ProxySQL/Redis 可达）与 Q5（install.sh 可达性）；vault 生成 `k3s_token` | gz-06 onboarding 完成后、主题启动前 |
| **D-0 开工** | 按 `.cursor/rules/10-docs-workflow.mdc` §1 九步：role 改造（k3s-server + k3s-agent + k3s-ruoyi）→ runbook → 集群安装（**第一次提交**）→ 应用迁移 + Nginx 切换（**第二次提交**）→ 3 类故障演练 + drills 记录 → Compose 摘除（**第三次提交**，期满后）→ retrospective → 架构快照 → tag | proposal 评审 + PoC 通过 |
| **后续接力（Helm）** | 进入 [`helm-chart-mgmt`](../scheme/phase-1-architecture-upgrade.md#helm-chart-mgmt)，把本主题裸 YAML 封装成 Chart | 本主题 retrospective 通过 |

**为什么 bootstrap 拆三次提交**（与 v1.8 同思路：让 git 中的状态与集群在线状态始终对得上）：

1. **第一次（集群就绪）**：合入 k3s-server / k3s-agent role 与 `setup_k3s.yml`、inventory K3s 分组（gz-05 server + gz-04/gz-06 agent）、vault `k3s_token`；执行后 `kubectl get nodes` 三节点 `Ready`，但尚未承载业务
2. **第二次（应用迁移 + 切流）**：合入 `k3s-ruoyi` role 与 manifest 模板，apply 后 Pod `Running`；gz-01 Nginx upstream 切到 NodePort，公网验证通过；此时 Compose 老副本仍在线作回滚兜底
3. **第三次（摘除老副本）**：3 天回滚窗口期满，停 gz-02 / gz-03 Compose ruoyi 并从 `app_nodes` 摘除，清理

如落地中发现新子问题（如某云 install.sh 被墙、flannel over Tailscale MTU 问题、Pod 无法出站到 Tailscale 内网等），按 [`.cursor/rules/10-docs-workflow.mdc`](../../.cursor/rules/10-docs-workflow.mdc) §6 升级为 `Docs/reviews/v1.9-<slug>.md`，再链接回本主题。