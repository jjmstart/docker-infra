# runbooks/ — 操作手册

本目录每个文件对应一次架构演进的完整操作手册，记录演进目标、前置条件、分步操作命令、预期输出、验证方法与回滚方式。将任意版本手册提供给 AI，即可立即进入操作协助模式。

## 手册索引

| 文件 | 演进内容 | 涉及节点 |
|------|----------|----------|
| [v1.0-to-v1.1.md](v1.0-to-v1.1.md) | 监控栈（Prometheus/Grafana）从 bj-01 迁至 gz-01，补齐四节点 Node Exporter | gz-01（迁入目标）、gz-02/gz-03（新增 exporter）、bj-01（迁出源，保留 exporter） |
| [v1.1-to-v1.2.md](v1.1-to-v1.2.md) | MySQL 单点升级为一主两从 + 半同步复制 + GTID + ProxySQL 读写分离 + bj-01 延迟复制 + mysqld_exporter 监控 + 故障演练 | gz-03（主库 + ProxySQL）、gz-02（Slave1）、bj-01（Slave2 延迟复制）、gz-01（Prometheus 采集） |
| [v1.2-to-v1.3.md](v1.2-to-v1.3.md) | `/opt/docker` 纳入 Git 版本控制，推送至 GitHub Private 仓库，建立 .env.example 模板（Phase 6 未执行，已由 v1.4 取代） | gz-03（Git 初始化 + 推送） |
| [v1.3-to-v1.4.md](v1.3-to-v1.4.md) | 引入 Ansible 统一配置管理，仓库重构为 Ansible 目录结构，bj-01 作为控制节点，ansible-vault 加密密码，Jenkins CI/CD 全自动部署 | gz-03（仓库重构）、bj-01（Ansible 控制节点）、gz-01/gz-02/gz-03/bj-01（全节点 Ansible 接管） |
| [v1.4-to-v1.5.md](v1.4-to-v1.5.md) | 新增 Prometheus alert rules、blackbox-exporter、Alertmanager 和飞书通知，建立服务异常主动告警闭环 | gz-01（Prometheus/Grafana/blackbox-exporter）、bj-01（Alertmanager/飞书 adapter/Ansible 控制节点）、gz-02/gz-03（监控目标） |
| [v1.5-to-v1.6.md](v1.5-to-v1.6.md) | 新增私有 Docker Registry、CI 镜像构建推送 Pipeline、参数化 CD 部署、Smoke Test 和飞书部署通知，建立完整应用交付流水线 | bj-01（Registry/Jenkins CI+CD/GC Job）、gz-02/gz-03（Docker daemon 配置 + ruoyi 镜像地址变更）、gz-01（Smoke Test 数据源） |
| [v1.6-to-v1.7.md](v1.6-to-v1.7.md) | 新增 MySQL 逻辑备份、OSS 上传和 bj-01 临时库恢复演练，建立备份恢复最小闭环 | gz-03（MySQL Master + 备份执行）、bj-01（Ansible 控制 + 恢复演练） |
