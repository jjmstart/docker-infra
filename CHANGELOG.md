# Changelog

本文件聚合 `docker-infra` 集群配置仓库的所有可见变更，按 [Keep a Changelog 1.1.0](https://keepachangelog.com/zh-CN/1.1.0/) 风格组织，版本号对应 `arch-vX.Y` Git tag。

## 维护节奏

- 每次 commit **不**强制同步更新（避免重复 commit 已有的能力）
- 每个 `arch-vX.Y` tag 发布前回填一次：从前一个 tag 起到本次 tag 的变更聚合写入对应版本段
- 当前 unreleased 段记录"上一个 tag 到 HEAD 之间"的变更，作为下一版本的预览

## 与其他文档的边界

- **commit message**：事件级记录（What），细粒度，git log 可查
- **CHANGELOG.md**（本文件）：版本级聚合（What），中粒度，按版本分组
- **Docs/architecture/vX.Y.md**：版本级最终状态（State），快照
- **Docs/retrospectives/vX.Y-retrospective.md**：单版本踩坑 + 决策原因（Why）
- **Docs/reviews/v{触发版本}-{主题}.md**：跨版本主题型审计（Why & Gap）

---

## [Unreleased]

下一版本主题为 [`helm-chart-mgmt`](Docs/scheme/phase-1-architecture-upgrade.md#helm-chart-mgmt)（将 ruoyi 的 K3s 裸 YAML 封装为 Helm Chart，统一镜像 tag 注入，消除 v1.9 遗留的 Deployment image 双轨制）。当前路线图见 `Docs/scheme/phase-1-architecture-upgrade.md`。

---

## [arch-v1.9] — 2026-06-12

**K3s 迁移无状态服务**：新接入腾讯云·广州常驻节点 gz-06（复用 v1.8 onboarding 流程），在 gz-05（server）+ gz-04 / gz-06（agent worker）上以全离线 airgap 方式拉起 K3s `v1.35.5+k3s1` 三节点集群（bj-01 经 DaoCloud 代理预置二进制 + 镜像包，apiserver / flannel VXLAN 流量强制走 Tailscale）；ruoyi 无状态后端从 gz-02 / gz-03 Compose 迁为 `ruoyi` namespace 下 Deployment 2 副本（跨双 worker 打散、`maxUnavailable:0`），经 NodePort 30080 由 gz-01 Nginx 变量驱动切流；滚动更新 + `rollout undo` 回滚与四类故障演练（坏镜像 / 坏探针 / drain / reboot 冷启动自愈）全部通过，演练期间公网探针 305/305 全 `200`。有状态服务（MySQL / Redis / ProxySQL）不进 K3s，数据层零配置变更；Compose 老副本 `stop` 转为有界冷备（至 2026-06-19）。新增 IaC 资产：`roles/k3s-airgap|k3s-server|k3s-agent|k3s-ruoyi/`、`playbooks/setup_k3s{,_app}.yml`、inventory K3s 分组、vault `k3s_token`；`jenkins-ansible` 镜像固化 `kubernetes` Python SDK。

- 详细架构状态：[`Docs/architecture/v1.9.md`](Docs/architecture/v1.9.md)
- 升级手册：[`Docs/runbooks/v1.8-to-v1.9.md`](Docs/runbooks/v1.8-to-v1.9.md)
- 复盘：[`Docs/retrospectives/v1.9-retrospective.md`](Docs/retrospectives/v1.9-retrospective.md)

---

## [arch-v1.8] — 2026-06-04

**六节点基础接入**：广州两台新节点 gz-04（百度云·Ubuntu 24.04）、gz-05（腾讯云·Ubuntu 22.04）完成 Tailscale 入网、Docker Engine 29.5.2 安装与 node-exporter 监控接入（仅基础接入，不承载业务）；新增 `roles/base-access/` 在 6 台节点幂等创建统一运维身份 `admin-alex`（纯追加，不删 root），`roles/docker-daemon/` 扩展为可管理 Docker Engine 安装与 `global_gateway` 网络。Prometheus `up{job="node-exporter"}` 扩至 6 个 target 全 `up=1`，同城实测 ~10-11ms、跨城 ~35-38ms。本版本同期收束了 V1.7 准备期的 IaC 完整性审计（新增 `Docs/reviews/` 子体系、`mysql_source_delay` 倒推 21 个可复现性问题、凭据泄漏事件复盘与全历史脱敏）与监控链路修复（`prometheus.yml.j2` 单文件 bind mount + atomic rename + reload 三件叠加导致的 6 版本潜伏失效、blackbox Jenkins 探测路径误报）。

- 详细架构状态：[`Docs/architecture/v1.8.md`](Docs/architecture/v1.8.md)
- 升级手册：[`Docs/runbooks/v1.7-to-v1.8.md`](Docs/runbooks/v1.7-to-v1.8.md)
- 复盘：[`Docs/retrospectives/v1.8-retrospective.md`](Docs/retrospectives/v1.8-retrospective.md)

---

## [arch-v1.7] — 2026-05-07

**备份恢复最小闭环**：gz-03 MySQL Master 每日 02:00 `mysqldump` 逻辑备份 `ry-vue` 业务库 → 压缩上传阿里云 OSS（本地保留 3 天）；bj-01 通过临时容器 `mysql-restore-test`（127.0.0.1:3307）完成恢复演练，实测 RTO ≈ 34 秒、最大 RPO 24 小时。

- 详细架构状态：[`Docs/architecture/v1.7.md`](Docs/architecture/v1.7.md)
- 升级手册：[`Docs/runbooks/v1.6-to-v1.7.md`](Docs/runbooks/v1.6-to-v1.7.md)
- 复盘：[`Docs/retrospectives/v1.7-retrospective.md`](Docs/retrospectives/v1.7-retrospective.md)

---

## [arch-v1.6] — 2026-05-04

**完整应用交付流水线**：私有 Docker Registry（bj-01）+ ruoyi CI 自动触发构建推送 + 参数化 CD 部署 + Smoke Test + Registry GC（两段式 DELETE + 裸 GC，避免 BuildKit OCI image index 误删）+ 飞书双机器人通知。

- 详细架构状态：[`Docs/architecture/v1.6.md`](Docs/architecture/v1.6.md)
- 升级手册：[`Docs/runbooks/v1.5-to-v1.6.md`](Docs/runbooks/v1.5-to-v1.6.md)
- 复盘：[`Docs/retrospectives/v1.6-retrospective.md`](Docs/retrospectives/v1.6-retrospective.md)

---

## [arch-v1.5] — 2026-04-28

**告警闭环**：Prometheus 规则 + blackbox-exporter（HTTP 探测）+ Alertmanager（bj-01）+ prometheus-alert 飞书 webhook 转换，建立服务异常主动告警链路。

- 详细架构状态：[`Docs/architecture/v1.5.md`](Docs/architecture/v1.5.md)
- 升级手册：[`Docs/runbooks/v1.4-to-v1.5.md`](Docs/runbooks/v1.4-to-v1.5.md)
- 复盘：[`Docs/retrospectives/v1.5-retrospective.md`](Docs/retrospectives/v1.5-retrospective.md)

---

## [arch-v1.4] — 2026-04-25

**Ansible + Jenkins CI/CD 统一配置管理**：仓库重构为 Ansible 目录结构（`roles/` + `inventory/` + `playbooks/`），bj-01 作为 Ansible 控制节点，`vault/secrets.yml` 用 ansible-vault 加密，Jenkins git push 全自动下发配置。

- 详细架构状态：[`Docs/architecture/v1.4.md`](Docs/architecture/v1.4.md)
- 升级手册：[`Docs/runbooks/v1.3-to-v1.4.md`](Docs/runbooks/v1.3-to-v1.4.md)
- 复盘：[`Docs/retrospectives/v1.4-retrospective.md`](Docs/retrospectives/v1.4-retrospective.md)

---

## [v1.3] — 2026-04-25（未打 tag）

**`/opt/docker` 纳入 Git 管理**：建立配置即代码基础管理层，推送至 GitHub Private 仓库；建立 `.env.example` 模板。本版本未单独打 `arch-v1.3` tag，因 V1.4 引入 Ansible 后仓库结构整体重构。

- 详细架构状态：[`Docs/architecture/v1.3.md`](Docs/architecture/v1.3.md)
- 升级手册：[`Docs/runbooks/v1.2-to-v1.3.md`](Docs/runbooks/v1.2-to-v1.3.md)（Phase 6 未执行，已由 V1.4 取代）

---

## [arch-v1.2] — 2026-04-24

**MySQL 主从高可用**：MySQL 升级为一主两从（gz-03 Master + gz-02 实时 Slave1 + bj-01 延迟 Slave2，`SOURCE_DELAY=3600`）+ 半同步复制 + GTID + ProxySQL 读写分离 + mysqld_exporter 监控接入。

- 详细架构状态：[`Docs/architecture/v1.2.md`](Docs/architecture/v1.2.md)
- 升级手册：[`Docs/runbooks/v1.1-to-v1.2.md`](Docs/runbooks/v1.1-to-v1.2.md)
- 复盘：[`Docs/retrospectives/v1.2-retrospective.md`](Docs/retrospectives/v1.2-retrospective.md)

---

## [v1.1] — 早期版本（未打 tag）

**监控栈迁至 gz-01**：原本部署于 bj-01 的 Prometheus/Grafana 迁移到 gz-01；补齐四节点 Node Exporter 全量采集。

- 详细架构状态：[`Docs/architecture/v1.1.md`](Docs/architecture/v1.1.md)
- 升级手册：[`Docs/runbooks/v1.0-to-v1.1.md`](Docs/runbooks/v1.0-to-v1.1.md)
- 复盘：[`Docs/retrospectives/v1.1-retrospective.md`](Docs/retrospectives/v1.1-retrospective.md)

---

## [v1.0] — 项目起点（未打 tag）

**初始四节点拓扑**：gz-01 / gz-02 / gz-03 / bj-01 Tailscale 内网互联；监控栈位于 bj-01；Redis 主从 + Sentinel；ruoyi 业务后端。

- 详细架构状态：[`Docs/architecture/v1.0.md`](Docs/architecture/v1.0.md)
