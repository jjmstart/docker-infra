# architecture/ — 集群架构快照

本目录每个文件对应一个架构版本的静态快照，记录该版本集群的完整状态（节点角色、服务部署、网络拓扑、技术决策）。将任意版本文件提供给 AI，即可在无需额外上下文的情况下完整理解当时的集群状态。

## 版本索引

| 文件 | 版本 | 一句话说明 | 状态 |
|------|------|------------|------|
| [v1.0.md](v1.0.md) | V1.0 | 初始拓扑：四节点 + Redis 哨兵，监控栈（Prometheus/Grafana）位于 bj-01 | 历史归档 |
| [v1.1.md](v1.1.md) | V1.1 | 监控栈迁至 gz-01，补齐四节点 Node Exporter 全量采集 | 历史归档 |
| [v1.2.md](v1.2.md) | V1.2 | MySQL 升级为一主两从 + 半同步复制 + GTID + ProxySQL 读写分离 + mysqld_exporter 监控 | 历史归档 |
| [v1.3.md](v1.3.md) | V1.3 | `/opt/docker` 纳入 Git + GitHub Private 管理，建立配置即代码基础管理层 | 历史归档 |
| [v1.4.md](v1.4.md) | V1.4 | 引入 Ansible + Jenkins CI/CD，git push 全自动下发配置到各节点 | 历史归档 |
| [v1.5.md](v1.5.md) | V1.5 | 告警闭环：新增 Prometheus 规则、blackbox-exporter、Alertmanager 和飞书 webhook 通知 | 历史归档 |
| [v1.6.md](v1.6.md) | V1.6 | 完整应用交付链路：私有 Docker Registry、ruoyi CI 自动触发、参数化 CD、Smoke Test、Registry GC、飞书双机器人 | **当前最新** ✓ |
