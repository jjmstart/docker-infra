# retrospectives/ — 演进总结

本目录每个文件对应一次架构演进完成后的复盘文档，记录演进动机、技术决策原因、踩坑记录与操作心得，帮助回顾并加深对技术原理的理解。

## 总结索引

| 文件 | 对应版本 | 完成日期 | 一句话摘要 |
|------|----------|----------|-----------|
| [v1.1-retrospective.md](v1.1-retrospective.md) | V1.1 | 2025 年 | 监控栈从 bj-01 迁至 gz-01，实现监控独立，补齐四节点 Node Exporter 全量采集 |
| [v1.2-retrospective.md](v1.2-retrospective.md) | V1.2 | 2026 年 | MySQL 升级为一主两从 + 半同步 + GTID + ProxySQL 读写分离，故障演练 RTO 实测纯操作约 3 分钟 |
| [v1.4-retrospective.md](v1.4-retrospective.md) | V1.4 | 2026 年 | Ansible + Jenkins CI/CD 统一配置管理复盘，完成 git push 自动下发全节点配置 |
