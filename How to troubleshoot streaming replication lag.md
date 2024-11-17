# How to troubleshoot streaming replication lag

PostgreSQL 流复制允许从主服务器到备服务器进行持续的数据复制，以确保高可用并平衡只读负载。然而，复制延迟可能会发生，从而导致数据同步延迟。本文提供了排查和缓解复制延迟的步骤。

## 识别延迟位置

在分析之前，我们需要了解实际存在延迟的位置，以及位于复制的哪个阶段：

- 通过 `walsender` 发送 WAL 流到备服务器的网络传输
- 通过 `walreceiver` 从网络接收 WAL 流到备服务器
- 备服务器上通过 `walreceiver` 写入 WAL 到磁盘
- 作为恢复过程的 WAL 应用 (回放)

因此，流复制延迟可以分为三类：

1. **写延迟**：事务在主服务器提交后，数据写入备服务器的 WAL 之间的延迟。
2. **刷新延迟**：事务写入备服务器的 WAL 后，刷新到磁盘之间的延迟。
3. **应用延迟**：事务从刷新到磁盘后，应用到备服务器数据库的延迟。

## 分析查询

要识别延迟，请使用以下 SQL 查询：

```sql
select
  pid,
  client_addr,
  application_name,
  state,
  coalesce(pg_current_wal_lsn() - sent_lsn, 0) AS sent_lag_bytes,
  coalesce(sent_lsn - write_lsn, 0) AS write_lag_bytes,
  coalesce(write_lsn - flush_lsn, 0) AS flush_lag_bytes,
  coalesce(flush_lsn - replay_lsn, 0) AS replay_lag_bytes,
  coalesce(pg_current_wal_lsn() - replay_lsn, 0) AS total_lag_bytes
from pg_stat_replication;
```

## 示例

查询结果可能如下所示：

```sql
postgres=# select
  pid,
  client_addr,
  application_name,
  state,
  coalesce(pg_current_wal_lsn() - sent_lsn, 0) AS sent_lag_bytes,
  coalesce(sent_lsn - write_lsn, 0) AS write_lag_bytes,
  coalesce(write_lsn - flush_lsn, 0) AS flush_lag_bytes,
  coalesce(flush_lsn - replay_lsn, 0) AS replay_lag_bytes,
  coalesce(pg_current_wal_lsn() - replay_lsn, 0) AS total_lag_bytes
from pg_stat_replication;

   pid   |  client_addr   | application_name | state     | sent_lag_bytes | write_lag_bytes | flush_lag_bytes | replay_lag_bytes | total_lag_bytes
---------+----------------+------------------+-----------+----------------+-----------------+-----------------+------------------+-----------------
 3602908 | 10.122.224.101 | backupmachine1   | streaming |              0 |       728949184 |               0 |                0 |               0
 2490863 | 10.122.224.102 | backupmachine1   | streaming |              0 |       519580176 |               0 |                0 |               0
 2814582 | 10.122.224.103 | replica1         | streaming |              0 |          743384 |               0 |          1087208 |         1830592
 3596177 | 10.122.224.104 | replica2         | streaming |              0 |         2426856 |               0 |          4271952 |         6698808
  319473 | 10.122.224.105 | replica3         | streaming |              0 |          125080 |          162040 |          4186920 |         4474040
```

## 如何解读结果

`_lsn` 的含义：

- `sent_lsn`：已经通过网络发送的 WAL (LSN 位置)。
- `write_lsn`：已经发送到操作系统的 WAL (LSN 位置)，还未刷至磁盘。
- `flush_lsn`：已经刷新到磁盘的 WAL (LSN 位置)，已写入磁盘。
- `replay_lsn`：已经应用到数据库的 WAL (LSN 位置)，对查询可见。

因此，延迟是 `pg_current_wal_lsn` 与 `replay_lsn` 之间的差值 (`total_lag_bytes`，并且将其添加到监控中是一个好主意)。但为了排查问题，我们需要监控所有四种延迟。

- `sent_lag_bytes` 的延迟意味着我们在发送数据时遇到问题，例如主服务器上的 `WALsender` CPU 饱和或网络负载过重。
- `write_lag_bytes` 的延迟意味着我们在接收数据时遇到问题，例如备服务器上的 `WALreceiver` CPU 饱和或网络负载过重。
- `flush_lag_bytes` 的延迟意味着我们在备服务器上将数据写入磁盘时遇到问题，例如 `WALreceiver` 的CPU 饱和或 I/O 瓶颈。
- `replay_lag_bytes` 的延迟意味着我们在应用 WAL 时遇到问题，通常是 CPU 饱和或 Postgres 进程的 IO 争用

一旦定位了问题，我们需要在操作系统级别排查相关进程，以找出瓶颈所在。

## 可能的瓶颈

待定

## 额外资源

- [Streaming replication](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION) (官方文档)
- [pg_stat_replication view](https://www.postgresql.org/docs/current/monitoring-stats.html#PG-STAT-REPLICATION-VIEW) (官方文档)
- [Replication configuration parameters](https://www.postgresql.org/docs/current/runtime-config-replication.html) (官方文档)

## 作者/维护者

- Dmitry Fomin
- Sadeq Dousti
- Nikolay Samokhvalov