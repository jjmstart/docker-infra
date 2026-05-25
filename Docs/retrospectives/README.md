# retrospectives/ — 演进总结

本目录每个文件对应一次架构演进完成后的复盘文档，记录演进动机、技术决策原因、踩坑记录与操作心得，帮助回顾并加深对技术原理的理解。

## 总结索引

| 文件 | 对应版本 | 完成日期 | 一句话摘要 |
|------|----------|----------|-----------|
| [v1.1-retrospective.md](v1.1-retrospective.md) | V1.1 | 2025 年 | 监控栈从 bj-01 迁至 gz-01，实现监控独立，补齐四节点 Node Exporter 全量采集 |
| [v1.2-retrospective.md](v1.2-retrospective.md) | V1.2 | 2026 年 | MySQL 升级为一主两从 + 半同步 + GTID + ProxySQL 读写分离，故障演练 RTO 实测纯操作约 3 分钟 |
| [v1.3-retrospective.md](v1.3-retrospective.md) | V1.3 | 2026 年（事后补写） | `/opt/docker` 纳入 Git + GitHub Private + `.env.example` 模板；同步方案 Phase 6 未落地即被 V1.4 取代，复盘"过渡版本"性质与四条设计盲点 |
| [v1.4-retrospective.md](v1.4-retrospective.md) | V1.4 | 2026 年 | Ansible + Jenkins CI/CD 统一配置管理复盘，完成 git push 自动下发全节点配置 |
| [v1.5-retrospective.md](v1.5-retrospective.md) | V1.5 | 2026 年 | 告警闭环复盘：新增 Prometheus 规则、blackbox-exporter、Alertmanager 和 prometheus-alert 飞书通知组件，Phase 1-8 全部通过，arch-v1.5 tag 已打 |
| [v1.6-retrospective.md](v1.6-retrospective.md) | V1.6 | 2026 年 | 应用交付链路复盘：私有 Docker Registry、ruoyi CI/CD 全链路打通、参数化部署回滚、Smoke Test、飞书职责分离通知、Registry GC Pipeline，全量验收通过，arch-v1.6 tag 已打 |
| [v1.7-retrospective.md](v1.7-retrospective.md) | V1.7 | 2026 年 | 备份恢复闭环复盘：mysqldump + OSS 上传 + bj-01 恢复演练，实测 RTO 34 秒，含 IDE 未保存/RAM 授权/vault 密码文件等 6 条踩坑记录 |
