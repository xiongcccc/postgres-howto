# Index maintenance

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

在大型项目中，索引维护是不可避免的。越早建立维护流程，性能表现会越好。

## 分析索引健康状况

索引健康分析包括：

- 识别无效索引
- 膨胀分析
- 查找未使用的索引
- 查找冗余索引
- 检查索引损坏情况

## 无效索引

查找无效索引非常简单：

```sql
nik=# select indexrelid, indexrelid::regclass, indrelid::regclass
from pg_index
where not indisvalid;
 indexrelid | indexrelid | indrelid
------------+------------+----------
      49193 | t_id_idx   | t1
(1 row)
```

在 [Postgres DBA](https://github.com/NikolayS/postgres_dba/) 中可以找到更全面详细的查询。

在分析这个列表时，请记住，如果索引正在通过 `CREATE INDEX CONCURRENTLY` 或 `REINDEX CONCURRENTLY` 创建或重建，无效索引可能是正常情况，因此还值得检查 `pg_stat_activity `以识别此类进程。

其他无效索引必须重建 (`REINDEX CONCURRENTLY`) 或删除 (`DROP INDEX CONCURRENTLY`)。

## 膨胀索引

如何分析索引膨胀：参照 [Day 46: How to deal with bloat](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0046_how_to_deal_with_bloat.md)

膨胀程度较高的索引 (真实的或预估的) 超过 50%，必须通过 `REINDEX CONCURRENTLY `进行重建。

由于随着时间的推移，索引健康状况会因更新而下降，因此需要重新索引。以下方法可以帮助减缓这种下降：

- 升级到 Postgres 14 或更新的版本 (以利用 btree 优化)。
- 优化工作负载和架构，以使用更多的 HOT ([Heap Only Tuple](https://postgresql.org/docs/current/storage-hot.html))。
- 调整 autovacuum，使其更积极地执行清理 (但请记住，`VACUUM` 不会重新平衡 btree 索引)。

因此，你需要定期对已知膨胀的索引使用 `REINDEX CONCURRENTLY`。理想情况下，这个过程应该自动化。例如：[pg_auto_reindexer](https://github.com/vitabaks/pg_auto_reindexer)。

## 未使用的索引

可以在 [pg_stat_user_indexes ](https://postgresql.org/docs/current/monitoring-stats.html)中找到使用信息。[Postgres DBA](https://github.com/NikolayS/postgres_dba/) 中提供了一个用于分析的查询示例。

在查找未使用的索引时，请保持谨慎，避免错误：

- 确保统计信息已经足够长时间没有重置。例如，如果统计信息几天前才被重置，你认为某个索引未被使用，这可能是一个错误 — 如果需要这个索引来支持下个月 1 号的一些报告。
- 不要忘记收集承接工作负载的所有节点的统计信息 — 包括主节点和所有副本。在主节点上看似未使用的索引，可能在副本上是必需的。
- 如果有多个系统安装，请确保分析了所有系统或至少一部分具有代表性的系统。

一旦可靠地识别出未使用的索引，那么应该使用 `DROP INDEX CONCURRENTLY` 进行删除。

我们能否软删除索引 (将其"隐藏"以确保计划器行为不变，如果没有问题，再执行真正的删除，否则快速恢复至原始状态)？不幸的是，这里没有简单的答案：

- [HypoPG 1.4.0](https://github.com/HypoPG/hypopg/releases/tag/1.4.0) 有一个"隐藏"索引的功能 — 这非常有用，但需要安装它。更重要的是，在整个工作负载中使用它可能具有挑战性，因为你需要调用 `hypopg_hide_index(oid)`。

- 有些人使用将 `indisvalid` 设置为 false 的技巧来隐藏索引 — 但有可靠的观点认为这不是一个安全的做法；请参见 [Peter Geoghegan's Tweet](https://twitter.com/petervgeoghegan/status/1599191964045672449):

  > It's unsafe, basically. Though hard to say just how likely it is to break. Here is one hazard that I know of: in general such an update might break a concurrent `pg_index.indcheckxmin = true` check. It will effectively "change the xmin" of your affected row, confusing the check.
  >
  > 基本上，这是不安全的。尽管很难说它破坏的可能性有多大。我知道的一个风险是：通常这种更新可能会破坏并发的 `pg_index.indcheckxmin = true` 检查。它实际上会"改变受影响行的 xmin"，使检查混淆。

## 冗余索引

如果索引 B 可以有效地支持与索引 A 相同 (甚至更多) 的查询集，那么索引 A 相对于索引 B 是冗余的。几个例子：

- 列 (a) 上的索引相对于列 (a, b) 上的索引是冗余的。
- 列 (a) 上的索引相对于列 (b, a) 上的索引不是冗余的。

具有完全相同定义的索引是彼此冗余的 (即重复索引)。

可以在 [Postgres DBA](https://github.com/NikolayS/postgres_dba/) 或 [Postgres Checkup](https://gitlab.com/postgres-ai/postgres-checkup) 中找到用于识别冗余索引的查询示例。

冗余索引通常可以安全删除 (在手动仔细检查列表后)，即使要删除的索引当前正在使用。

## 损坏

使用 `pg_amcheck` 识别 btree 索引中的损坏情况 (详细内容将在另一篇操作指南中介绍)。

截至 2023 年/PG16，`pg_amcheck` 还不支持以下功能，但未来有计划添加：

- [检查 GIN 和 GIST 索引](https://commitfest.postgresql.org/45/3733/)
- 检查唯一键 (已经推送，可能在 PG17 中发布)