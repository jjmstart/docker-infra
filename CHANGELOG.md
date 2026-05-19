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

## [Unreleased] — V1.7 → V1.8 准备期

为下一版本 `arch-v1.8` 准备的累积变更。当前路线图见 `Docs/scheme/phase-1-architecture-upgrade.md`：V1.8 主题为"基础可复现性修复（Docker 网络 + MySQL 数据层 bootstrap）"。

### Added

- 新增 `Docs/reviews/` 文档子体系，承载跨版本主题型横向复盘 (`d4ffcc4`)
- 首篇审计 `Docs/reviews/v1.7-iac-completeness-audit.md`：从 `mysql_source_delay` 倒推出 21 个 IaC 完整性问题（10 个已修 + 11 个排进 V1.8-V1.10） (`d4ffcc4`)
- 新增 `.cursor/rules/15-current-facts.mdc` 作为动态事实查询入口，AI 不再把规则文件示例当作权威事实 (`e97fe64`)

### Changed

- `inventory/group_vars/all.yml` 服务地址改为按主机分组派生（`hostvars[groups[...]].ansible_host`），节点 IP 调整不再需要手动改服务地址 (`d175ec9`)
- `mysqld-exporter` 从 `monitor-stack` 中拆出独立 role，role 职责对应单一组件 (`1e85509`)
- `roles/docker-daemon/templates/daemon.json.j2` 中 `insecure-registries` 改为按主机变量驱动 (`dcc9669`)
- `roles/diary/` 与 `playbooks/setup_diary.yml` 拆出，diary 部署链路与静态资源独立由 diary role 持有 (`070e55d`, `4c43171`)
- `playbooks/setup_backup.yml` 接入 `playbooks/site.yml` 主流程，告别"忘了跑"风险 (`f07f0e6`)
- `.cursor/rules/00-project-context.mdc` / `10-docs-workflow.mdc` 去除硬编码的四节点表与 IP / 规格 / 延迟，统一指向 inventory 与最新架构快照 (`0c9581b`)
- `.cursor/rules/01 / 20 / 30 / 40 / 50` 统一去硬编码，红线与示例不再钉在某一版本 (`d578515`)
- `.cursor/rules/01-user-profile.mdc` 重写决策原则 2：能力标准是"能拿 offer + 能干活"，不是"必须 0 AI 完成" (`f21b12f`)
- `.cursor/rules/00-project-context.mdc` 澄清 AI 协作边界：AI 是合法生产力工具，要点是"能讲清"而非"AI 是否参与" (`4a14c10`)
- `roles/nginx/templates/**` 站点配置与主模板注释精简 (`2f20877`, `d9667da`)

### Fixed

- `roles/nginx/handlers/main.yml` 修复 reload 在 compose 重启前执行的 handler 顺序 bug (`22b2306`)

### Removed

- 废弃 V1.8 节点接入 proposal 与对应 runbook（路线调整后节点接入顺延至 V1.11） (`a307b5e`)
- `roles/nginx/templates/conf.d/jjmstart.conf` 移除历史测试 location 块 (`e24b153`)
- `roles/nginx/templates/conf.d/test.conf.disabled` 移除未使用站点配置 (`4efd596`)

### Docs

> 项目内工程治理类（注释维护、Pipeline 注释补全等），不影响运行行为。

- `Jenkinsfile` 与 `Jenkinsfile.registry-gc` 补全 Pipeline 步骤注释 (`8e4644f`, `9a60feb`)
- `jenkins-build/Dockerfile` 注释补全 (`5a2da66`)
- `roles/jenkins/`、`roles/ruoyi/`、`roles/mysql-replica/`、`roles/docker-daemon/` 内 task / template / handler 注释维护 (`4bf4fd0`, `a7f6800`, `d18afc7`, `e49918a`, `fdc2ae0`)
- `inventory/group_vars/all.yml` 注释更新 (`57a2ef3`)

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
