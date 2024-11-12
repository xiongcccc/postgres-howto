# How to rebuild many indexes using many backends avoiding deadlocks

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

有时我们需要重建多个索引，甚至全部索引，并希望加快速度。

例如，在从 Postgres 14 之前的版本升级到 14+ 后，为了从降低膨胀增长率这一优化项中受益，我们可能希望重建所有 B 树索引。

我们可以选择使用单一会话并设置较高的 [max_parallel_maintenance_workers](https://postgresqlco.nf/doc/en/param/max_parallel_maintenance_workers/) 值，一次处理一个索引。但如果我们有足够的资源 (大量的 vCPU 和快速的磁盘)，那么 `max_parallel_maintenance_workers` 最大值可能也不足以充分利用资源 (更改 `max_parallel_maintenance_workers` 不需要重启，但不能超过 `max_worker_processes` 的数量，且更改该值需要重启)。在这种情况下，使用 `REINDEX INDEX CONCURRENTLY`，并行处理多个索引可能更合适。

但在这种情况下，索引需要按正确的顺序处理。问题在于，如果你尝试并行重建属于同一张表的两个索引，那么会检测到死锁，并且其中一个会话将失败：

```sql
nik=# reindex index concurrently t1_hash_record_idx3;
ERROR:  deadlock detected
DETAIL:  Process 40 waits for ShareLock on virtual transaction 4/2506; blocked by process 1313.
Process 1313 waits for ShareUpdateExclusiveLock on relation 16634 of database 16401; blocked by process 40.
HINT:  See server log for query details.
```

为了解决这个问题，我们可以使用以下方式：

1. 决定要使用多少个重建会话，考虑 `max_parallel_maintenance_workers` 和预期资源利用率/饱和的风险 (CPU 和磁盘 IO)。
2. 假设我们想使用 N 个重建会话，构建索引的完整列表及其所属的表名称，并为每个表"分配"一个特定的重建会话，见下方的查询。
3. 使用此"分配"，将整个索引列表划分为 N 个独立的列表，以确保每个表的所有索引仅出现在一个独立列表中 — 然后我们可以使用这 N 个列表运行 N 个会话。

以下查询可以帮助完成步骤 2：

```sql
\set NUMBER_OF_SESSIONS 10

SELECT
  format('%I.%I', n.nspname, c.relname) AS table_fqn,
  format('%I.%I', n.nspname, i.relname) AS index_fqn,
  mod(
    hashtext(format('%I.%I', n.nspname, c.relname)) & 2147483647,
    :NUMBER_OF_SESSIONS
  ) AS session_id
FROM
  pg_index idx
  JOIN pg_class c ON idx.indrelid = c.oid
  JOIN pg_class i ON idx.indexrelid = i.oid
  JOIN pg_namespace n ON c.relnamespace = n.oid
WHERE
  n.nspname NOT IN ('pg_catalog', 'pg_toast', 'information_schema')
  -- and ... additional filters if needed
ORDER BY
  table_fqn, index_fqn;
```

## 我见

这个技巧挺不错：

- **NUMBER_OF_SESSIONS**：设置需要的重建会话数量。
- **session_id**：通过 `hashtext` 函数和模运算将表分配给不同的会话。
- **WHERE 子句**：排除系统模式 (`pg_catalog`、`pg_toast` 和 `information_schema`) 中的索引，避免不必要的重建。