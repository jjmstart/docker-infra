# SLI / SLO 定义

## 文档定位

本文件是 `docker-infra` 集群的**运营层指标契约**：定义对哪些服务做可靠性承诺、用什么指标衡量、目标值是多少、违约时如何兜底。它和架构快照（"集群是什么样"）、reviews（"架构有哪些盲点"）正交：架构定义能力、reviews 识别缺口、SLI/SLO 给出**对外承诺与判定标准**。

**单一来源原则**：

- 服务清单与采集来源以 `Docs/architecture/` 最新版为准
- 告警阈值字段以 `inventory/group_vars/all.yml` 中 `alert_*` 系列变量为准，本文件**不复制具体值**，只引用变量名 + 当前值
- 实测样本以 Prometheus 真实查询结果回填，禁止造数

**与 SRE 经典 SLO 的边界说明**：本项目当前为单人维护的学习集群，不存在多团队责任划分、不需要错误预算燃烧率（burn-rate）告警、不签订 SLA 对外赔付。本文件保留 SLI/SLO 框架但**不引入完整 SRE 错误预算策略**，避免为面试堆砌而无法叙事——保留"指标定义 + 月度目标 + 违约时怎么办"三段最小核心。

---

## 1. 服务清单与可靠性分级

按"对外可见性"和"业务重要性"两个维度分级，决定本文件是否纳管：

| 服务 | 对外入口 | 可靠性分级 | 是否纳入本文件 | 理由 |
|---|---|---|---|---|
| 若依业务后端 | `https://ruoyi.jjmstart.com` | Tier-1 | ✓ | 业务主入口，最贴近"用户视角可用性" |
| Jenkins | `https://jenkins.jjmstart.com` | Tier-2 | ✓ | CI/CD 平台，挂掉会阻断发布 |
| Grafana | `https://monitor.jjmstart.com` | Tier-2 | ✓ | 监控视图，挂掉会让排障盲飞 |
| Diary | `https://diary.jjmstart.com` | Tier-3 | ✓ | 个人记事工具，可用性影响小但纳入对外探测一致性 |
| Prometheus | `http://100.117.7.75:9090/-/healthy` | Tier-2 | ✓ | 告警判断者本身，挂掉则告警链路失效 |
| Alertmanager | `http://100.118.69.78:9093/-/healthy` | Tier-2 | ✓ | 告警分发，挂掉则飞书无通知 |
| MySQL Master / 从库 | 无对外入口 | 内部基础设施 | ✗（在告警侧覆盖） | 不直接对外服务，但通过 `MySQLMasterDown` / `MySQLReplicationDown` 告警纳管 |
| Registry | `http://100.118.69.78:5000` | 内部基础设施 | ✗ | 仅集群内 Docker 客户端使用，不对公网 |

> Tier 等级仅用于本项目内部沟通与 SLO 目标差异化，不用于外部 SLA 承诺。

---

## 2. SLI 定义

每个 SLI 必须可观测、可量化、可在 Prometheus 中查询。

### 2.1 入口可用性（Availability）

| 字段 | 内容 |
|---|---|
| 指标 | `probe_success{job="blackbox-http"}` |
| 采集来源 | gz-01 blackbox-exporter（HTTP `http_2xx` 模块，允许 200 / 301 / 302） |
| 采集频率 | Prometheus `scrape_interval: 15s`（见 `roles/monitor-stack/templates/prometheus.yml.j2`） |
| 计算公式 | 月度窗口内 `avg_over_time(probe_success[30d]) * 100`，按 `service` 标签分组 |
| 适用服务 | `service` 标签：`ruoyi` / `jenkins` / `diary` / `grafana` / `prometheus` / `alertmanager` |

### 2.2 后端基础可达性（Backend Reachability）

| 字段 | 内容 |
|---|---|
| 指标 | `up{job="node-exporter"}` 与 `up{job="mysqld-exporter"}` |
| 采集来源 | 各节点 node-exporter / mysqld-exporter |
| 采集频率 | 同上 15s |
| 计算公式 | 月度窗口 `avg_over_time(up[30d]) * 100`，按 `instance` 分组 |
| 适用对象 | 4 个 node-exporter target + 3 个 mysqld-exporter target（清单以 `prometheus.yml.j2` 渲染结果为准） |

### 2.3 数据保护（Data Protection）

| 字段 | 内容 |
|---|---|
| RPO（Recovery Point Objective） | 当前最大值 24 小时（每日 02:00 全量逻辑备份）；bj-01 延迟从库为误操作场景额外提供约 1 小时回档窗口 |
| RTO（Recovery Time Objective） | 从 OSS 下载 → 临时容器初始化 → 导入 → 校验全过程，**v1.7 实测 34 秒**（不含人工决策；详见 `Docs/architecture/v1.7.md` "RPO / RTO 说明"） |
| 备份新鲜度 SLI | OSS 上 `docker-infra/mysql/ruoyi/` 前缀最新对象的 `LastModified` 时间距 `now()` 不超过 25 小时 |
| 备份完整性 SLI | 备份脚本 `set -Eeuo pipefail` 以 0 退出 + 本地 `gzip -t` 通过 |

> 备份新鲜度 SLI 当前**没有自动告警**，依赖人工查阅 `mysql_backup.log`。这是 `Docs/retrospectives/v1.7-retrospective.md` "遗留问题"已识别的缺口，待后续小任务补齐。

---

## 3. SLO 目标

### 3.1 入口可用性月度目标

按服务 Tier 差异化设置。**目标值是保守版本**，先保证可达后再逐月校准，避免初期目标过严反而失去参考意义。

| 服务 | Tier | 月度可用性目标 | 月度允许的"不可用预算" |
|---|---|---|---|
| 若依业务后端 | Tier-1 | 99.5% | 约 3.65 小时 / 月 |
| Jenkins | Tier-2 | 99.0% | 约 7.30 小时 / 月 |
| Grafana | Tier-2 | 99.0% | 约 7.30 小时 / 月 |
| Prometheus | Tier-2 | 99.0% | 约 7.30 小时 / 月 |
| Alertmanager | Tier-2 | 99.0% | 约 7.30 小时 / 月 |
| Diary | Tier-3 | 95.0% | 约 36 小时 / 月 |

**目标值的取值逻辑**：

- Tier-1 设 99.5% 而非 99.9%：当前 gz-01 是单点入口（v1.12 计划迁至 gz-02 + watchdog 闭环），单云厂商月级故障 + 计划维护窗口都会消耗预算，99.5% 是有诚意但有可达性的目标
- Tier-2 设 99.0%：运维平台 + 监控自身允许月度有更长维护窗口，9 小时容错够覆盖一次跨城演练或重启
- Tier-3 不设高目标：避免为 Diary 这种私人工具消耗运维注意力，95% 是诚实声明

### 3.2 数据保护目标

| 维度 | 目标 | 备注 |
|---|---|---|
| RPO 上限 | ≤ 24 小时 | 每日 02:00 备份；超期触发"备份新鲜度"违约 |
| RTO 上限 | ≤ 5 分钟 | 实测 34 秒，留充足余量给人工决策与不同时间窗口的 OSS 下载波动 |
| 月度备份成功率 | ≥ 30/31 次 | 每月最多允许 1 次备份失败（需当日内补偿执行） |
| 季度恢复演练 | ≥ 1 次 | 每季度至少完成 1 次完整恢复演练并产出 `Docs/drills/` 文档 |

---

## 4. SLI → 告警规则映射

每条 SLI 必须有对应告警规则保证违约可被发现。映射关系如下，所有阈值变量值以 `inventory/group_vars/all.yml` 当前定义为准：

| SLI | 对应告警规则 | 阈值变量 | 触发条件 |
|---|---|---|---|
| 入口可用性 | `BlackboxProbeFailed` | 硬编码 `for: 2m` | `probe_success{job="blackbox-http"} == 0` 持续 2 分钟 |
| 后端可达性（节点） | `NodeDown` | 硬编码 `for: 2m` | `up{job="node-exporter"} == 0` 持续 2 分钟 |
| 后端可达性（MySQL Master） | `MySQLMasterDown` | 硬编码 `for: 1m` | `mysql_up{instance="<master>"} == 0` 持续 1 分钟 |
| 后端可达性（MySQL 复制） | `MySQLReplicationDown` | 硬编码 `for: 1m` | IO/SQL 复制线程任一异常持续 1 分钟 |
| 实时从复制延迟 | `MySQLRealtimeReplicaLagHigh` | `alert_mysql_realtime_replica_lag_seconds_warning` | 延迟超过该值持续 2 分钟 |
| 延迟从复制状态 | `MySQLDelayReplicaLagAbnormal` | `alert_mysql_delay_replica_lag_seconds_warning` | 延迟超过该值（设计余量留给避免误报）持续 5 分钟 |
| 节点资源 | `NodeDiskSpaceLow` / `NodeInodeLow` / `NodeMemoryLow` / `NodeCPUHigh` | `alert_disk_free_percent_critical` 等 | 阈值超过持续 5–10 分钟 |
| 监控自身 | `PrometheusTargetDown` / `AlertmanagerDown` / `BlackboxExporterDown` | 硬编码 `for: 2m` | 探测/采集失败 |

**告警分发链路时延约束**（受 `alert_group_wait: 30s` / `alert_group_interval: 5m` / `alert_repeat_interval: 4h` 影响）：

- 首次触发 → 飞书到达：理论 ≤ `for` 时间 + `group_wait`（如 BlackboxProbeFailed: 2 分 30 秒）
- 同组重复事件：每 `group_interval` (5m) 发一次摘要
- 已发送但未恢复：每 `repeat_interval` (4h) 重复提醒

---

## 5. 实测样本（每月回填）

下表样本由 Prometheus 30 天窗口查询回填，**不允许凭印象估值**。每次回填都对应一个查询命令以便复核。

### 5.1 入口可用性 30 天实测

查询命令模板（在 `monitor.jjmstart.com` Grafana Explore 或 `100.117.7.75:9090` Prometheus UI 执行）：

```promql
avg_over_time(probe_success{job="blackbox-http"}[30d]) * 100
```

| 服务 | SLO 目标 | 30 天实测 | 是否达标 | 数据来源日期 |
|---|---|---|---|---|
| ruoyi | 99.5% | _待回填_ | _待回填_ | _待回填_ |
| jenkins | 99.0% | _待回填_ | _待回填_ | _待回填_ |
| grafana | 99.0% | _待回填_ | _待回填_ | _待回填_ |
| prometheus | 99.0% | _待回填_ | _待回填_ | _待回填_ |
| alertmanager | 99.0% | _待回填_ | _待回填_ | _待回填_ |
| diary | 95.0% | _待回填_ | _待回填_ | _待回填_ |

### 5.2 备份新鲜度近 30 天

查询方式：`ossutil ls oss://<bucket>/docker-infra/mysql/ruoyi/ -r --human-readable | tail -n 30`，统计每个工作日是否产生新备份对象。

| 月份 | 应备份次数 | 实际成功次数 | 失败原因记录 |
|---|---|---|---|
| _YYYY-MM_ | _待回填_ | _待回填_ | _待回填_ |

### 5.3 RTO 复测记录

每次执行 `Docs/drills/` 中的恢复演练时回填一行：

| 演练日期 | 演练文档 | 实测 RTO | 备份大小 | 备注 |
|---|---|---|---|---|
| 2026-05-06 | `Docs/architecture/v1.7.md` 首次实测 | 34 秒 | 7.4 MB | v1.7 验收期 |
| _待回填_ | _待回填_ | _待回填_ | _待回填_ | _待回填_ |

---

## 6. 违约时怎么办

SLO 不达标时，按违约严重度分级响应。**这一节不是制度文档，是给自己留的判断指南**。

| 严重度 | 触发条件 | 响应动作 |
|---|---|---|
| 黄灯 | 单月单服务可用性低于目标 0.5–1 个百分点 | 在 `Docs/retrospectives/` 当版本回顾里补一段说明，识别根因，不必单独立 review |
| 红灯 | 单月低于目标 1 个百分点以上，或连续两月不达标 | 触发一份 `Docs/reviews/v{触发版本}-{主题}.md`，按 reviews 子体系流程系统性盘点 |
| 备份未达标 | 单月备份失败 ≥ 2 次，或最新备份距今 > 25h | 立即手动补一次备份，定位失败根因，必要时升级到红灯流程 |
| 演练未达标 | 季度未完成至少 1 次演练 | 优先级提升至当前求职/架构演进副线之上，强制补做一次 |

---

## 7. 维护规则

- 本文件每月**月初**回填一次 §5 实测样本，回填时同步更新数据来源日期
- 每次执行 `Docs/drills/` 演练后回填 §5.3
- SLO 目标值原则上每季度评估一次，调整目标必须在 §3 留下"调整日期 + 原因"注脚
- 阈值变量值发生变化（`alert_*` 系列）时，本文件 §4 不需要改文本（已用变量名引用），但需 grep 确认没有遗漏的硬编码
- 本文件**不向 `arch-vX.Y` Git tag 强绑定**：SLO 是横向运营契约，跨架构版本稳定，调整不构成架构语义变化
- 触发红灯流程时必须开 review，触发黄灯时由月初回填一并记录

---

## 8. 与其他文档的边界

| 文件 | 回答的问题 |
|---|---|
| `Docs/architecture/vX.Y.md` | 集群当前是什么样（**State**） |
| `Docs/runbooks/vX.Y-to-vX.Z.md` | 怎么从 A 改到 B（**Transition**） |
| `Docs/retrospectives/vX.Y-retrospective.md` | 单次演进踩了什么坑（**Why**） |
| `Docs/reviews/v{触发版本}-{主题}.md` | 跨版本主题型盲点（**Why & Gap**） |
| `Docs/drills/v{触发版本}-{场景}.md` | 主动验证已有能力的演练记录（**Verification**） |
| **本文件** | 对外承诺什么、怎么衡量、违约怎么办（**Contract**） |
