# How to NOT get screwed as a DBA (DBRE)

以下规则非常简单 (但由于组织原因，实施起来往往不那么简单)。

这对在快速成长的初创公司中负责数据库的人会有所帮助，其中一些建议也适用于大公司的 DBA/DBRE。

## 1) 确保备份系统可靠

1. 不要使用 `pg_dump` 作为备份工具 — 而是使用支持 PITR (例如 `pgBackRest`，`WAL-G`) 的系统。如果使用的是托管解决方案，请详细了解备份系统是如何组织的 — 不要盲目相信。

2. 了解 RPO 和 RTO，测量实际值并定义目标值。使用监控系统覆盖这些值 (例如，`archive_command` 滞后应被视为最高优先级的事件)。
3. 测试备份。这是最关键的部分。未经测试的备份是"薛定谔的备份" — 任何备份的状态在尝试恢复之前都是未知的。自动化测试。
4. 局部恢复的自动化和加速：在某些情况下，需要恢复手动删除的数据，而不是整个数据库。在这种情况下，考虑以下选项以加快恢复速度：(a) 特殊的延迟备库；(b) 频繁的云快照 + PITR；(c) 使用 DBLab，每小时快照 + 克隆上的 PITR。
5. 其他需要涵盖的主题：适当的加密选项、保留策略，将老的备份移动到"冷存储"以节省成本、次要存储位置，位于不同云中，并限制访问。

毫无疑问，备份是数据库管理中最重要的话题。在这个方面遇到麻烦是任何 DBA 的噩梦。请对备份保持高度关注，学习他人的经验教训，而不是自己的。

可靠的备份系统也许是一些组织选择托管 Postgres 服务的最大原因之一。但仍然：不要盲目相信 — 仔细研究所有细节，并自行测试。

## 2) 数据损坏控制

1. 启用[数据校验和](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0037_how_to_enable_data_checksums_without_downtime.md)。
2. 谨慎进行操作系统或 `glibc` 升级 — 避免索引损坏。
3. 使用 `amcheck` 来检测索引、堆和序列。
4. 设置警报，快速响应 Postgres 日志中的错误代码 `XX000`、`XX001`、`XX002` ([PostgreSQL Error Codes](https://postgresql.org/docs/current/errcodes-appendix.html))
5. 数据损坏有很多种类型 — 覆盖基础部分，然后继续学习并实施更好的控制和预防措施。(关于该主题的一组好材料：[PostgreSQL Data Corruption and Bugs – Runbook](https://docs.google.com/spreadsheets/u/1/d/1zUH7IYOv46CVSmc-72CD7ROnMA6skJSQZjnm4yxvX9A/edit#gid=0))

## 3) 高可用

- 设置备节点。
- 考虑使用 `synchronous_commit=remote_write` (或根据情况选择 `remote_apply`) 和 `synchronous_standby_names` ([Multiple Synchronous Standbys](https://postgresql.org/docs/current/warm-standby.html#SYNCHRONOUS-REPLICATION-MULTIPLE-STANDBYS))。
- 如果是自托管，使用 **Patroni**。否则，研究你的提供商提供的所有高可用选项并使用它们。

## 4) 性能

- 拥有良好的监控系统 ([my PGCon slide deck on monitoring](https://twitter.com/samokhvalov/status/1664686535562625034))。
- 设置高级查询分析工具：`pg_stat_statements`、`pg_stat_kcache`、`pg_wait_sampling` 或 `pgsentinel`、`auto_explain`。
- 构建可扩展的"实验室"环境和实验流程 — DBA 不应该成为工程师们实验的瓶颈 ([@Database_Lab](https://twitter.com/Database_Lab) 可以解决这个问题)。
- 实施容量规划，确保有足够的增长空间，进行主动的基准测试。
- 架构设计：微服务和分片都是很棒的想法，值得考虑。但主要问题是：何时开始？从初创公司成立之初，还是之后？答案是"视情况而定"。请记住，凭借当前 Postgres 的性能和任何云提供的硬件，你可以轻松扩展到几十 TB 或数百 TB 的数据量，以及每秒成千上万的 TPS。选择你的优先事项 — 有许多成功的案例可以证明任何方法都是可行的。
- 不要犹豫寻求帮助 — 无论是社区帮助还是付费咨询。Postgres 支撑着大量的项目，有丰富的经验可以借鉴。

## 5) 学习他人的错误

- Postgres 仍然使用 32 位事务ID。确保不要触发事务 ID (以及组事务 ID) 的回卷问题 — [Sentry](https://blog.sentry.io/transaction-id-wraparound-in-postgres/) 和[Mailchimp](https://mailchimp.com/what-we-learned-from-the-recent-mandrill-outage/) 的案例是很好的教训。
- 子事务 — 我个人认为在负载较重的系统中使用它们是危险的，建议[避免](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0035_how_to_use_subtransactions_in_postgres.md)。
- 调优 `autovacuum` — 不要让 Postgres 积累大量的死元组或膨胀 (是的，这两者是不同的)。关闭 `autovacuum` 是让你的服务器宕机的好方法。
- 在 OLTP 环境中，避免长时间运行的事务和未使用/滞后的复制槽。
- 学习如何无停机地部署架构变更。有用的文章：
  - [Zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries)
  - [Common DB schema change mistakes](https://postgres.ai/blog/20220525-common-db-schema-change-mistakes)