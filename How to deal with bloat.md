# How to deal with bloat

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

## 什么是膨胀？

膨胀是页面内部的空闲空间，在`autovacuum` 删除大量元组时产生。

当更新表中的一行时，Postgres 不会覆盖旧数据，而是将旧的行版本 (元组) 标记为"dead"，并创建一个新的行版本。随着时间推移，越来越多的行被更新或删除，死元组占据的空间会积累起来。在某个时间点，`autovacuum` (或手动执行 `VACUUM`) 会删除死元组，留下页面内的空闲空间供重用。但是，如果死元组累积过多，可能会留下大量空闲空间 — 在最糟糕的情况下，它可能占据表或索引空间的 99% 或更多)

较低的膨胀率 (例如，低于 40%) 不应被视为问题，而较高的膨胀率会有问题，因为它们会导致严重的后果：

1. 磁盘使用量增加
2. 读写查询需要更多 IO
3. 缓存效率下降 (包括缓冲池和操作系统文件缓存)
4. 结果：查询性能变差

## 如何检查膨胀

应该定期检查索引和表的膨胀情况。注意，大多数常用的查询都是基于预估，容易出现误报 — 根据表的结构，有时会显示一些并不存在的膨胀 (我见过一些情况，即使是在刚创建的表中，也显示了高达 40% 的"幽灵膨胀")。但这些查询速度快，不需要安装额外的扩展。示例：

- [Estimated table bloat](https://github.com/NikolayS/postgres_dba/blob/master/sql/b1_table_estimation.sql)
- [Estimated btree index bloat](https://github.com/NikolayS/postgres_dba/blob/master/sql/b2_btree_estimation.sql)

监控系统中的建议：

- 没有必要频繁运行这些查询 (例如每分钟一次)，因为膨胀水平并不会迅速变化。
- 应向用户提供警告，告知结果是预估值。
- 对于大型数据库，查询执行时间可能较长，长达数秒，因此需要调整检查频率和 `statement_timeout`。

更精确确定膨胀水平的方式：

- 基于 `pgstattuple` 的查询 (需要安装扩展)。
- 在克隆的数据库上检查数据库对象大小，运行 `VACUUM FULL` (过重且会阻塞查询，不适用于生产) ，然后再次检查大小并进行前后对比。

定期检查膨胀水平是推荐的，以便在需要时及时做出反应。

## 缓解索引膨胀 (被动)

不幸的是，在经历大量 `UPDATE` 和 `DELETE` 的数据库中，索引健康状况会不可避免地会随着时间的推移而恶化。这意味着索引需要定期重建。

建议：

- 使用 `REINDEX CONCURRENTLY` 以非阻塞方式重建膨胀的索引。
- 请记住，`REINDEX CONCURRENTLY` 在运行时会持有 `xmin` 视界，这会影响 `autovacuum` 清理近期死元组的能力。这是使用分区的另一个原因；不要让表超过一定阈值 (例如，不超过 100 GiB)。
- 你可以使用第 15 天的监控方法监控重建索引的进度： [Day 15: How to monitor CREATE INDEX / REINDEX]().
- 优先使用 Postgres 14+ 版本，因为在 PG14 中，btree 索引经过了显著优化，在写入工作负载下性能下降得更慢。

## 缓解表膨胀 (被动)

某些水平的表膨胀可能不是坏事，因为它增加了优化 `UPDATE` 的机会 —`HOT` (Heap-Only Tuples) 更新。

但是，如果膨胀水平令人担忧，考虑使用 [pg_repack](https://github.com/reorg/pg_repack) 在不长时间持有排他锁的情况下重建表。`pg_repack` 的替代工具有：[pg_squeeze](https://github.com/cybertec-postgresql/pg_squeeze)

通常，这个过程不需要定期调度和完全自动化；通常只需在检测到高表膨胀时有控制地应用即可。

## 主动缓解膨胀

- 调整 `autovacuum`。
- 监控 `xmin` 视界，不要让其滞后太久 — [Day 45: How to monitor xmin horizon to prevent XID/MultiXID wraparound and high bloat]().
- 不要允许不必要的长事务 (例如超过 1 小时)，无论是在主节点还是在启用了 `hot_standby_feedback` 的备节点上。
- 如果使用 Postgres 13 或更早版本，请考虑升级到 14+ 以从 btree 索引优化中受益。
- 对大表 (100+ GiB) 进行分区。
- 对具有队列类型工作负载的表使用分区，即使它们很小，并使用 `TRUNCATE` 或删除分区来处理旧数据，而不是使用 `DELETE`；在这种情况下，不需要 `VACUUM`，膨胀也不再是问题。
- 不要使用大量的 `UPDATE` 和 `DELETE`，始终分批处理 (持续时间不超过 1-2 秒)；确保 `autovacuum` 及时清理死元组，或者在需要进行大量数据更改时手动执行 `VACUUM`。

## 推荐阅读材料

- Postgres 官方文档: [Routine Vacuuming](https://postgresql.org/docs/current/routine-vacuuming.html)
- [When autovacuum does not vacuum](https://2ndquadrant.com/en/blog/when-`autovacuum`-does-not-vacuum/)
- [How to Reduce Bloat in Large PostgreSQL Tables](https://timescale.com/learn/how-to-reduce-bloat-in-large-postgresql-tables/)