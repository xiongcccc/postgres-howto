# How to troubleshoot a growing pg_wal directory

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

`$PGDATA/pg_wal` 目录包含 WAL 文件。WAL (预写式日志) 是用于备份、恢复以及物理复制和逻辑复制的核心机制。

在某些情况下，`pg_wal` 目录大小会持续增长，这可能会引发担忧，因为它会增加磁盘空间耗尽的风险。

以下是排查增长中的`pg_wal`目录问题的步骤。

## 步骤1：检查复制槽

未使用或滞后的复制槽会阻止 WAL 文件被回收，因此 `pg_wal` 目录的大小会增加。在主库上检查：

```sql
select
  pg_size_pretty(pg_current_wal_lsn() - restart_lsn) as lag,
  slot_name,
  wal_status,
  active
from pg_replication_slots
order by 1 desc;
```

参考文档： [The view pg_replication_slots](https://postgresql.org/docs/current/view-pg-replication-slots.html)

- 如果有非活跃状态的复制槽，考虑删除它们以防止磁盘空间达到 100%。一旦删除了有问题的槽，Postgres 将会移除旧的 WAL 文件。
- 另外，考虑使用 `max_slot_wal_keep_size` (PG13+)。

## 步骤2：检查`archive_command`是否正常工作

如果配置了 `archive_mode` 和 `archive_command` 来归档 WAL 文件 (例如用于备份)，但 `archive_command` 失败 (返回了一个非零的退出代码) 或滞后 (WAL 的生成速率超过归档速度)，这可能是 `pg_wal` 增长的另一个原因。

如何监控和排查：

- 检查 [pg_stat_archiver](https://postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ARCHIVER-VIEW)
- 检查 Postgres 日志 (例如，查看`archive command failed with exit code 1`)

一旦确定了问题，`archive_command` 需要修复或提升速度 (例如，启用 `wal_compression = on`，增加 `max_wal_size` 以减少 WAL 数据的生成；同时，在归档工具中使用较轻的压缩方式 — 具体取决于 `archive_command `中使用的工具。例如，WAL-G 支持多种压缩选项，使用的 CPU 强度不同)。

接下来的两个步骤是附加的，因为它们对 `pg_wal` 大小增长的影响有限 — 它们只会导致一定数量的额外 WAL 文件保存在 `pg_wal` 中 (不像我们前面讨论的两种原因，它们可能会导致更大的影响)。

## 步骤3：检查`wal_keep_size`

在某些情况下，`wal_keep_size` ([PG13+ docs](https://postgresqlco.nf/doc/en/param/wal_keep_size/)；在 PG12 及更早版本中，请参照`wal_keep_segments`)被设置为较大的值。当使用复制槽时，通常不需要使用这个机制 — 这是在复制槽机制出现之前的旧方法，用于避免某些 WAL 文件被删除，导致滞后的副本无法跟上。

## 步骤4：检查`max_wal_size`和`checkpoint_timeout`

当成功的检查点发生时，Postgres 可以删除旧的WAL文件。在某些情况下，如果调整了检查点设置以减少检查点的频率，这可能会导致存储在 `pg_wal` 中的 WAL 文件要多于预期。如果这对磁盘空间造成问题 (特别是在小型服务器上)，可以考虑将 `max_wal_size` 和 `checkpoint_timeout` 设置为较低的值。在某些情况下，手动执行 `CHECKPOINT` 命令也是合理的，以便 Postgres 立即清理一些旧文件。

## 总结

最重要的检查项：

- 检查未使用或滞后的复制槽
- 检查失败或滞后的 `archive_command`

此外：

- 检查 `wal_keep_size`
- 检查 `max_wal_size` 和 `checkpoint_timeout`