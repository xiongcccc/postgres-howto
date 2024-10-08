# How to speed up bulk load

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

如果你需要加载大量数据，以下技巧可以帮助你提升加载速度：

## 1) 使用 COPY

使用 `COPY` 加载数据，它针对批量加载进行了优化。

## 2) 减少检查点频率

考虑暂时增加 `max_wal_size` 和 `checkpoint_timeout` 的值。

调整这些参数不需要重启。更大的值意味着发生故障时恢复时间会更长，但好处是检查点发生的频率减少，因此：

- 对磁盘的压力减小，

- 由于相同页面的全页写次数减少，写入的 WAL 数据也减少 (特别是在存在索引的情况下加载数据)。

## 3) 增加缓冲池

如果可能，增加 `shared_buffers` 的大小。

## 4) 减少 (或移除) 索引

如果数据加载到一个新表中，可以在数据加载完成后再创建索引。如果是加载到一个现有表中，[避免过多的索引。](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0018_over_indexing.md)

每个额外的索引都会显著减慢数据加载的速度。

## 5) 减少 (或移除) 外键和触发器

与索引类似，外键约束和触发器也可能会显著减慢数据加载速度。可以考虑在批量加载后再 (重新) 创建它们。

触发器可以通过 `ALTER TABLE ... DISABLE TRIGGERS ALL` 禁用 — 但如果触发器支持某些一致性检查，需要确保这些检查没有被违反 (例如，数据加载后运行额外的检查)。外键通过隐式触发器实现，`ALTER TABLE ... DISABLE TRIGGERS ALL `也会禁用它们 — 在这种情况下加载数据时要特别小心。

## 6) 避免WAL写入

如果这是一个新表，可以考虑在数据加载期间完全避免 WAL 写入。有两种选择 (都有限制，需要理解如果发生崩溃，数据可能会丢失)：

- 使用无日志表：`CREATE UNLOGGED TABLE ...`。无日志表不会归档，也不会复制，且不是持久化的 (虽然它们在正常重启时仍然存在)。不过，将无日志表转换为普通表可能需要一些时间（通常需要很长时间 — 值得测试），因为数据需要写入WAL。有关未记录表的更多信息，请参阅这篇文章；另请参阅此 [StackOverflow](https://dba.stackexchange.com/questions/195780/set-postgresql-table-to-logged-after-data-loading/195829#195829) 讨论。
- 使用 `wal_level='minimal'` 的 `COPY`。`COPY` 必须在建表的事务内执行。由于`wal_level='minimal'`，`COPY` 写入的数据不会记录在WAL中（对于 PG16，只有非分区表才能使用此选项)。此外，可以考虑使用 `COPY (FREEZE)`，这一方法的好处是数据加载完成后所有元组都会被冻结。设置 `wal_level='minimal'` 需要重启 Postgres，还需要其他更改 (如 `archive_mode = 'off'`，`max_wal_senders = 0`)。这种方法通常不适合生产环境，但对于单服务器配置可以很好地工作。有关 wal_level='minimal' + COPY (FREEZE) 方法的详细信息，请参阅[此帖](https://www.cybertec-postgresql.com/en/loading-data-in-the-most-efficient-way/)。

## 7) 并行化

考虑并行化。这可能会加速过程，但也取决于单线程进程的瓶颈 (例如，如果单线程负载使磁盘 IO 饱和，则并行化将无济于事)。有两种选择：

- 分区表和多进程加载多个分区。可以使用多个工作进程并行加载 ([Day 20: pg_restore tips]())。
- 非分区表和大块数据加载。这需要事先准备大块数据 — 可以将 CSV 文件拆分成多个部分，或使用多个同步的 `REPEATABLE READ` 事务导出表数据的范围 (通过 `SET TRANSACTION SNAPSHOT `在同一个快照下工作)。参照：[Day 8: How to speed up pg_dump]().

如果你使用 TimescaleDB，考虑使用 [timescaledb-parallel-copy ](https://github.com/timescale/timescaledb-parallel-copy)工具。

最后但同样重要的是：在进行大规模数据加载后，不要忘记运行 `ANALYZE`。

# 我见

过多索引，我写了一篇文章进行分析：[慢工出细活，久久方为功](https://mp.weixin.qq.com/s?__biz=MzUyOTAyMzMyNg==&mid=2247492180&idx=1&sn=a9385efacbcf77564fbb59924ba3a1fe&chksm=fa65ca65cd1243733ce53b24026780cca8bca70ba5b1025a046c423df26bfacd978b3baef2e0&token=1396136569&lang=zh_CN#rd)，另外关于 minimal + copy 的技巧，也值得写一篇，PostgreSQL 14 internals 一书中也有介绍，敬请期待。