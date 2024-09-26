# Over-indexing

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享

众所周知，每添加一个额外的索引，都会使写操作 (`INSERT`，`UPDATE` — 除非是 `HOT`)变慢。但请耐心读下去，今天的内容一定会让你大吃一惊。

------

## 索引与写入开销

显然，如果你创建了一个新索引，那么每当有产生新的行版本的语句 (`INSERT`，`UPDATE`)时，Postgres 也需要更新这个索引 — 这意味着写入语句的执行时间会更长。

对于 `UPDATE`，有一种特殊的优化机制 — HOT 更新，Heap-Only-Tuple updates — 它可以使 Postgres 避免这种开销。如果满足以下两个条件，这种优化便会生效：

- `UPDATE` 不涉及参与索引定义的列 (考虑所有索引；有一个例外：得益于一项特殊的优化，Postgres 16+ 版本中的 BRIN 索引在此检查中会被跳过)，并且
- 更新的元组所在的页面有空闲空间。

文档：[Heap-Only Tuples (HOT)](https://postgresql.org/docs/current/storage-hot.html)

在本文中，我将讨论一个不太明显的情况，在这种情况下，试图优化索引集时，我们完全丧失了 HOT (这意味着 `UPDATE` 变慢了)：

- [How partial, covering, and multicolumn indexes may slow down UPDATEs in PostgreSQL](https://postgres.ai/blog/20211029-how-partial-and-covering-indexes-affect-update-performance-in-postgresql)

抛开 HOT 更新不谈，结论非常简单：索引越多，写入越慢。

该怎么做：避免过度使用索引，明智地添加新索引，定期进行索引维护，删除未使用和冗余的索引。

## 索引与规划时间

---

现在，让我们看看一些不那么直观的情况。

以下是一个[基准测试](https://gitlab.com/postgres-ai/postgresql-consulting/tests-and-benchmarks/-/issues/41)，其中 @VKukharik 表明，每增加一个额外的索引，一个简单的 `SELECT` (主键查找) 的规划时间显著变慢 (感谢 @Adrien_nayrat 提供了关于规划时间的宝贵建议)：

![Degradation of query plan time with increase in number of indexes](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0018_degradation_with_indexes.png)

与执行时间相比：

![Mean plan / execution time vs. # indexes](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0018_plan_exec_time_vs_index.png)

拥有 1 个索引的表在规划时间上表现要好 4 倍以上，并且在这种特定情况下，规划时间显著高于执行时间。

这是因为对于每个索引，Postgres 的规划器需要分析更多的执行选项。下面是相同查询在具有 32 个索引的相同表上的火焰图 ([source](https://gitlab.com/postgres-ai/postgresql-consulting/tests-and-benchmarks/-/issues/41#note_1602558372)):

![Flame graph](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0018_flame_graph.png)

对于简单 SELECT 查询，总体延迟的最显著负面影响可以观察到，执行时间很低，而当有很多索引时，执行时间可能远远低于规划时间。该怎么做：

对于简单的 `SELECT`，可以观察到对整体延迟最显着的负面影响，其中执行时间很短，并且有大量索引，可能比计划时间短得多。

- 减少索引数量。
- 考虑使用预处理语句。如果你使用 pgBouncer 并且运行在 `pool_mode=transaction` 模式下，历史上由于 pgBouncer 不支持该模式，无法使用预处理语句。然而，2023 年 10 月，这个功能得到了[实现](https://github.com/pgbouncer/pgbouncer/pull/845) 🎉  (并在 [PgBouncer 1.21.0 中发布 — "带有预处理语句的版本"](https://github.com/pgbouncer/pgbouncer/releases/tag/pgbouncer_1_21_0))。其他一些连接池工具 (如 `odyssey`, `pgcat`) 也支持此功能，或计划支持 (supavisor)。

------

## 索引与fastpath=false (LWLock:LockManager contention)

你可能注意到，上面的图表显示，当索引数量达到 15 时，曲线的性质发生了变化，表现出比之前的线性趋势更糟糕的退化。

让我们理解为什么会发生这种情况，以及如何应对。

对于单个表的主键查找，当执行查询时 (规划时也是 — 运行一个简单的 EXPLAIN 而不执行也能观察到) Postgres 会在表及其所有索引上获取 AccessShareLock。这可能会让人感到惊讶。相关的源代码：`plancat.c`。

如何检查 — 假设我们有一个表 `t1`：

- 在一个会话中，运行：

```sql
select pg_backend_pid(); -- remember PID
begin;
explain select from t1 where id = 1; -- and keep tx open
```

- 在另一个会话中，运行：

```sql
select * from pg_locks where pid = {{PID}}; -- 使用上面的PID
```

你会看到，对于 `t1` 上的 N 个索引，已经获取了 N+1 个 `AccessShareLock` 的关系级锁。

对于前 16 个锁，你会在 `pg_locks.fastpath` 一列中看到为 `true`。从第 17 个开始，它将变为 `false`。这个阈值是硬编码的常量 [FP_LOCK_SLOTS_PER_BACKEND](https://gitlab.com/postgres/postgres/blob/22655aa23132a0645fdcdce4b233a1fff0c0cf8f/src/include/storage/proc.h#L85)。当 `fastpath=false` 时，Postgres 的锁管理器使用了一种较慢但更全面的方法来获取锁。详细信息可以参考[这里](https://gitlab.com/postgres/postgres/blob/22655aa23132a0645fdcdce4b233a1fff0c0cf8f/src/backend/storage/lmgr/README#L70)。

在高并发环境中，如果我们有 `fastpath=false` 的锁，可能会开始观察到 `LWLock` 争用，有时是严重的争用 — 很多活跃会话在 `pg_stat_activity` 中显示 `wait_event='LockManager'`  (在 PG13 或更早版本中为 `lock_manager` )。

这种情况可能会发生在主节点或备节点上，当两个条件满足时：

1. 高 QPS — 例如，对于观察的查询，每秒 100 次或更多 (取决于工作负载和硬件资源)。
2. `fastpath=false` 锁由于涉及的关系数超过了 16 个 (在 Postgres 中，表和索引都被视为"关系") — 这可能是过度索引，或者查询中涉及太多表 (例如分区表，且缺少分区修剪)，或是这些因素的组合。

正如前面提到的，`FP_LOCK_SLOTS_PER_BACKEND=16` 是硬编码的。2023年10月，-hackers邮件列表中正在讨论审查这个常量 (感谢 @fuzzycz 起草的 [patches](https://postgresql.org/message-id/flat/116ef01e-942a-22c1-a2af-35bf69c1b07b@enterprisedb.com#b19340c248755be70b805404becd43ad))。

------

该怎么做？

- 首先，避免过度使用索引。特别是对于那些涉及 (或计划涉及) 高频查询 (高 QPS) 的表，尽量保持索引数量尽可能少。
- 如果你对表进行了分区，再次确保你没有过度使用索引，并且查询计划仅涉及特定分区 (分区修剪)。
- 如果必须拥有大量索引，尝试降低相应查询的 QPS。
- 另一个减少争用的选项是增强集群性能：使用更快的机器，并将只读查询卸载到其他副本上。

关于这个话题的几篇好文章：

- Useful docs RDS Postgres (useful for non-RDS users too): [LWLock: lockmanager](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/wait-event.lw-lock-manager.html)
- [Postgres Partition Pains - LockManager Waits](https://kylehailey.com/post/postgres-partition-pains-lockmanager-waits) by [@kylelf_](https://twitter.com/kylelf_)