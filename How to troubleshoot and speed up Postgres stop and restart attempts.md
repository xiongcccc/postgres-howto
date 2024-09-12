# How to troubleshoot and speed up Postgres stop and restart attempts

#Postgres马拉松第2天。让我们来讨论有关 PostgreSQL 关闭和重启的问题。在实际操作中，遇到长时间甚至失败的关闭/重启操作并不罕见。如果这些问题发生在故障排查期间，往往会引发情绪波动和额外的失误。

导致关闭操作时间延长的一些常见原因：

- 存在长时间运行的事务
- 大量缓冲区是脏的 (更改已在内存中应用，但尚未同步到磁盘，正在等待另一个检查点)，导致关闭时的检查点时间过长。
- WAL 归档 (`archive_command`) 滞后。

下面，我将讨论每个原因以及如何缓解。

## 原因 1: 长时间运行的事务

如果你怀疑第一个原因，请检查是否有长时间运行的事务：

```sql
select
  clock_timestamp(),
  clock_timestamp() - xact_start,
  *
from pg_stat_activity
where clock_timestamp() - xact_start > interval '10 second'
order by clock_timestamp() - xact_start desc
limit 10;
```

在这种情况下，通常建议使用 `SIGINT` (`pg_ctl stop -m fast`)，即所谓的"快速关闭"模式 (参见 [Postgres 文档](https://www.postgresql.org/docs/current/server-shutdown.html))。

## 原因 2: 长时间的关闭时检查点

第二个原因是缓冲池中有大量脏缓冲区，这个问题不太容易排查，但幸运的是，解决起来很简单。特别是在以下情况下容易出现：

- 缓冲池大小 (`shared_buffers`) 很大
- 进行了检查点调整，旨在减少大规模随机写入和减少全页写的开销 (通常意味着增大 `max_wal_size` 和 `checkpoint_timeout`)
- 最近的检查点迄今已过去太久，可以在启用 `log_checkpoint = on` 的情况下从 PG 日志中看到，建议在大多数情况下启用此选项)。

脏缓冲区的数量很容易观测，可以使用扩展模块 pg_buffercache (标准的 contrib 模块)并运行以下查询 (可能需要较长时间；详见[文档](https://postgresql.org/docs/current/pgbuffercache.html))：

```sql
select count(*), pg_size_pretty(count(*) * 8 * 1024)
from pg_buffercache
where isdirty;
```

如果数值较大 (例如几个 GiB)，在尝试关闭时，Postgres 将执行所谓的"关闭时检查点"，将脏缓冲区刷新到磁盘 ([源码](https://gitlab.com/postgres/postgres/blob/ebf76f2753a91615d45f113f1535a8443fa8d076/src/backend/access/transam/xlog.c#L6229))。在此期间，它不会处理查询，这会影响停机时间。解决方法很简单——在尝试关闭/重启之前显式执行一个 CHECKPOINT：

```sql
checkpoint;
```

这将有助于我们使得关闭时检查点非常轻量化，减少停机时间。

在某些情况下，可能需要在尝试关闭前连续执行两次显式 CHECKPOINT：如果第一个 CHECKPOINT 很重，可能会花费时间，而在此期间由于持续写入，新的脏缓冲区会积累——我们通过第二个 CHECKPOINT 来减轻这一问题，使关闭检查点保持轻量且快速。

要模拟此处描述的情况：

- 确保 `shared_buffers` 很大 (几 GiB；更改此项需要重启)
- 增大 `max_wal_size` 和 `checkpoint_timeout` (更改不需要重启)：例如 `'10GB'` 和 `'60min'` (确保 `pg_wal` 子目录中有足够的磁盘空间)
- 在一个大表 t1 上执行：`set statement_timeout = '60s'; begin; delete from t1;` (将被取消，但会产生大量脏缓冲区)
- 安装 pg_buffercache 并检查上述脏缓冲区的数量
- 持续监控 Postgres 日志中的检查点记录 (`log_checkpoint=on`)

然后测量 `pg_ctl stop -m fast` 的时间——它将花费较长时间。重复同样的步骤，但在尝试关闭之前显式执行 CHECKPOINT。

这个建议 (显式 CHECKPOINT) 在涉及关闭或重启的自动化中非常重要，特别是在需要可预测的时间时。(例如，即使我们不打算在重启 Postgres 时将停机时间最小化，我们也不希望它花费很多分钟，可能会触发我们自动化工具中的各种超时限制。)

## 原因 3: archive_command 失败/滞后

这是一个不幸的情况：`pg_ctl stop -m fast` 会断开正在进行的会话，但它给 WAL 归档进程一个归档待处理 WAL 的机会，这会影响关闭的时间 (逻辑很复杂，可以在这些文件中找到：postmaster.c，pgarch.c)。

WAL 归档滞后应该被视为一个严重事件，因为它会影响备份的健康状态以及 DR 的 RPO/RTO。这需要被监控，在出现显著滞后 (例如，几十个未归档的 WAL 文件) 时，应将其视为一次事件。此类监控应包括 `pg_stat_archiver`，最好还有当前 LSN 的信息，以计算滞后量 (以 WAL 文件中的字节为单位)。至少，监控/告警应涵盖 `last_failed_wal` 和 `last_failed_time`。 

有趣的是，归档命令慢/失败也会导致切换/故障转移时的停机时间变长——例如，在 Patroni 2.1.2 之前的版本中，这是一个问题，后续版本修复了这一行为，旨在加快故障转移速度 (["Release the leader lock when pg_controldata reports 'shut down'"](https://github.com/zalando/patroni/blob/master/docs/releases.rst#version-212)).

## 总结

- 如果不希望现有会话正常结束工作，请使用"快速模式" (`pg_ctl stop -m fast`)
- 在尝试关闭或重启前始终执行显式 CHECKPOINT
- 监控 `pg_stat_archiver` 并确保它无故障和滞后运行

就是这样。注意，我们在这里没有讨论各种超时 (例如 pg_ctl 的 `--timeout` 选项以及等待行为 `-w`, `-W`，详见 [Postgres 文档](https://postgresql.org/docs/current/app-pg-ctl.html))，而只是讨论了可能导致关闭/重启尝试延迟的原因。 希望这对你有所帮助——像往常一样，记得订阅、点赞、分享和评论！💙

# 我见

停止 PostgreSQL 包括三种模式

1. “Smart” mode disallows new connections, then waits for all existing clients to disconnect. If the server is in hot standby, recovery and streaming replication will be terminated once all clients have disconnected. 
2. “Fast” mode (the default) does not wait for clients to disconnect. All active transactions are rolled back and clients are forcibly disconnected, then the server is shut down. 
3. “Immediate” mode will abort all server processes immediately, without a clean shutdown. This choice will lead to a crash-recovery cycle during the next server start.

为什么停库可能变慢各位还可以参照 [PgSQL · 应用案例 · PG有standby的情况下为什么停库可能变慢？](http://mysql.taobao.org/monthly/2019/09/10/)