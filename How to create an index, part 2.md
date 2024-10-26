# How to create an index, part 2

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

参照 [Part 1](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0061_how_to_create_an_index_part_1.md).

在第一部分中，我们讨论了创建索引的基础知识。今天，我们将探讨与索引创建相关的并行化和分区方面的内容。这两部分不涉及如何选择索引类型、何时使用部分索引、表达式索引或多列索引 — 这些将在单独的指南中讨论。

## 长时间索引创建及其加速方法

如前所述，索引创建时间过长，例如数小时 — 不仅不方便，还会阻止 `autovacuum` 处理表，并在整个操作期间持有 `xmin` 视界 (这意味着 `autovacuum` 无法删除数据库中所有表新生成的死元组)。

因此，值得优化单个索引的创建时间。以下是一些常见方法：

1. 配置调优

- 提高 `maintenance_work_mem` ，此前已经描述过。

  > 🎯 **TODO**: 通过实验展示如何调整此参数。

- **检查点调优**：临时提高 `max_wal_size` 和 `checkpoint_timeout` (无需重启) 以减少检查点频率，这可能会提高创建的时间。

  > 🎯 **TODO**: 进行实验来验证其效果。

2. 并行化 — 使用多个后台进程加快整个操作。

3. 分区 — 将表拆分为多个物理表可以减少创建单个索引所需的时间。

## 并行索引创建

选项 `max_parallel_maintenance_workers` (PG11+，参见[文档](https://postgresqlco.nf/doc/en/param/max_parallel_maintenance_workers/)) 定义了 `CREATE INDEX` 的最大并行工作进程数。目前 (截至 PG16)，它仅适用于创建 B-tree 索引。

默认 `max_parallel_maintenance_workers` 为 `2`，可以增加，不需要重启，可以在会话中动态调整。其最大值取决于两个设置：

- **`max_parallel_workers`**，默认为 8，同样可以在不重启的情况下更改。
- **`max_worker_processes`**，默认为 8，更改需要重启。

增加 `max_parallel_maintenance_workers` 可以显著减少索引构建时间，但应在适当分析 CPU 和磁盘 IO 利用率后进行。

> 🎯 **TODO**: 进行实验来验证效果。

## 分区表上的索引

如之前多个指南中讨论的那样，大表 (例如超过100 GiB；这不是一个硬性规定) 应当进行分区。如果没有分区，处理上 TB 的表时，索引创建时间会非常长，在此期间，`autovacuum` 无法处理该表。这会导致更高的膨胀度。

可以为各个分区创建索引。然而，考虑为所有分区使用统一的索引方法，并在分区表本身上定义索引是有意义的。

使用 `CONCURRENTLY` 选项无法在分区表 (父表) 上创建索引，只能在单个分区上使用。然而，这个问题可以通过以下方式解决 (PG11+)：

1. 在所有分区上分别创建索引，使用 `CONCURRENTLY`：

   ```sql
   create index concurrently i_p_123 on partition_123 ...;
   ```
   
2. 然后在分区表 (父表) 上创建索引，不使用 `CONCURRENTLY`，并使用关键字 `ONLY` — 这很快，因为物理上这不是一个大表，但在完全执行完下一步之前，它会被标记为 `INVALID`：

   ```sql
   create index i_p_main on only partitioned_table ...;
   ```
   
3. 然后，将每个分区上的索引标记为"attached"到"主"索引，使用这种稍微奇怪的语法 (注意，这里使用的是索引名称，而不是表名)：

   ```sql
   alter index i_p_main attach partition i_p_123;
   ```

相关文档：[Partition Maintenance](https://postgresql.org/docs/current/ddl-partitioning.html#DDL-PARTITIONING-DECLARATIVE-MAINTENANCE).