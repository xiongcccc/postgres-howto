# How to troubleshoot long Postgres startup

#Postgres马拉松第 3 天。在上一篇文章中，我们讨论了如何快速停止或重启 PostgreSQL。现在是时候讨论当你尝试启动服务器但看到以下错误时该如何应对了：

```sql
FATAL: the database system is not yet accepting connections
DETAIL: Consistent recovery state has not been yet reached.
```

或者：

```sql
FATAL: the database system is starting up
```

或者，从客户端的角度看：

```
pq: the database system is starting up
```

不该做的事情 (非专家常见的做法)：

- 不明所以就开始担心或等待很长时间
- 多次尝试停止/重新启动

该做的事情：

- 保持冷静
- 了解你的配置和工作负载
- 了解并观察 REDO 进程

接下来详细讨论每个步骤。

## 1. 保持冷静

这是最难的一部分 (如果是生产故障，时间压力很大等)，但这是必要的。为了帮助保持冷静，我们需要了解正在发生的事情。

上面的消息表明 Postgres 已启动，但尚未完成 REDO——即从最近一次成功的检查点时刻开始回放 WAL 记录。

日志示例，表明 REDO 确实已经启动：

```sql
2023-09-28 10:17:06.864 PDT [83002] LOG: database system was interrupted; last known up at 2023-09-27 14:49:18 PDT
2023-09-28 10:17:07.126 PDT [83002] LOG: database system was not properly shut down; automatic recovery in progress
2023-09-28 10:17:07.130 PDT [83002] LOG: redo starts at 26/6504A218
```

最大的挫败感之一是无法理解当前发生的事情，无法回答一个简单的问题："它正在做什么吗？"下面，我们将处理这些问题。

## 2. 了解你的配置和工作负载

### 2a. 检查 max_wal_size 和 checkpoint_timeout

这里最重要的配置与检查点的调整有关。如果 `max_wal_size` 和 `checkpoint_timeout` 进行了调整以使检查点发生得更少，那么如果关机不是干净的 (译者注：即 immediate 模式)，没有成功的关机时检查点 (例如，虚拟机重启)，或者如果你正在从备份中恢复，Postgres 需要更多时间才能达到一致性点。要了解更多信息，请参考：

- [官方文档](https://www.postgresql.org/docs/current/runtime-config-wal.html)
- [WAL 和检查点调优](https://postgres.fm/) (Postgres.fm podcast)

换句话说，如果你发现启动时间较长，这可能是因为服务器经过调优，在正常情况下减少了对 WAL 的写入以及缓冲区的同步频率，这样的调优会导致更长的启动时间，而这便是你正在处理的问题。

### 2b. 了解实际的检查点行为

建议启用 `log_checkpoint = on`。在 Postgres 14 及更早的版本中默认是关闭的，而在 PG15+ 中是开启的。

启用 `log_checkpoint` 后，你可以在日志中看到检查点的信息，了解检查点的频率以及写入了多少数据。以下是一个示例：

```sql
2023-09-28 10:17:17.798 PDT [83000] LOG: checkpoint starting: end-of-recovery immediate wait
2023-09-28 10:17:46.479 PDT [83000] LOG: checkpoint complete: wrote 465883 buffers (88.9%); 0 WAL file(s) added, 0 removed, 0 recycled; write=28.667 s, sync=0.001 s, total=28.681 s; sync files=6, longest=0.001 s, average=0.001 s; distance=5241900 kB, estimate=5241900 kB
```

在 Postgres 16+ 中，你还可以在日志中看到检查点和 REDO LSNs 的信息，这对 REDO 进程的监控非常有帮助 (见下文)。

如果你不了解 LSN，请阅读以下内容：

- [pg_lsn 类型](https://www.postgresql.org/docs/current/datatype-pg-lsn.html) (官方文档)
- Postgres WAL 文件和序列号
- [WAL, LSN 和文件名](https://fluca1978.github.io/2020/05/28/PostgreSQLWalNames) 

### 2c. 了解工作负载

WAL 文件存储在 `$PGDATA/pg_wal` 中，通常大小为 16 MiB (可以调整——例如 RDS 已将其改为 64 MiB)。负载很大的系统每秒可以创建多个 WAL 文件，每天产生多个 TiB 的 WAL 数据。

1. 检查 PGDATA 目录中的 `pg_wal` 子目录可以帮助了解每分钟创建了多少 WAL 文件。
2. 如果监控中存在当前的 LSN 及其增长率，这也会很有帮助——如果你在两个时间点获取了两个 LSN 值，你可以将它们相减，得到以字节为单位的"距离"，例如：

```sql
nik=# select pg_size_pretty('39/EF000000'::pg_lsn - '32/AA000000'::pg_lsn);
 pg_size_pretty
----------------
 29 GB
(1 row)
```

同样，你也可以获取由备份工具创建的备份对应的 LSN，并计算它们之间的差异。

## 了解并观察 REDO 进程

要了解进度，我们需要获取以下几个值：

- 当前 REDO 进程的位置 (当前正在回放的 LSN)
- 目标 LSN——一致性点，即 REDO 进程的终点
- (可选) 起始 LSN——REDO 进程的起始点

不幸的是，获取这些值并不容易——Postgres 并没有在日志中报告这些信息 (对于潜在的 hackers：这是个不错的改进点)。我们无法使用 SQL 获取任何信息，因为 `FATAL:  the database system is not yet accepting connections。`

如果你自己管理 Postgres，可以通过以下方式来确定这些值，理解并监控进程。

首先，查看当前状态：

```sql
❯ ps ax | grep 'startup recovering'
98786   ??  Us     0:15.81 postgres: startup recovering 000000010000004100000018
```

稍后：

```sql
❯ ps ax | grep 'startup recovering' | grep -v grep
99887   ??  Us     0:02.29 postgres: startup recovering 000000010000004500000058
```

——可以看到，位置在变化，这表明 REDO 进程正在进行。这已经很有帮助，应该能缓解一些压力。

现在，虽然无法使用 SQL，但我们可以使用 `pg_controldata` 来查看集群的元信息 (需要指定 PGDATA 位置，使用 `-D`):

```
❯ /opt/homebrew/opt/postgresql@15/bin/pg_controldata -D /opt/homebrew/var/postgresql@15 | grep Latest | grep -e location -e WAL
Latest checkpoint location:           48/E10B8B50
Latest checkpoint's REDO location:    45/1772EA98
Latest checkpoint's REDO WAL file:    000000010000004500000017
```

在 Postgres 16 中，你还可以看到 REDO 进程的起始位置和时间 (如果 `log_checkpoint=on`，推荐开启)。示例：

```
2023-09-28 01:23:32.613 UTC [81] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.003 s, sync=0.002 s, total=0.025 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=97 kB; lsn=0/1274EF8, redo lsn=0/1274EC1
```

如果我们正在处理一个副本或者从备份中恢复，可以使用这个值来判断 REDO 进程何时结束：

```sql
❯ /opt/homebrew/opt/postgresql@15/bin/pg_controldata -D /opt/homebrew/var/postgresql@15 | grep 'Minimum recovery ending location'
Minimum recovery ending location:     4A/E0FFE518
```

做一些计算（可以参考这篇文章的详细说明：[PostgreSQL WAL 和 LSN](https://fluca1978.github.io/2020/05/28/PostgreSQLWalNames)）：

- 当前位置 `000000010000004500000058` —— LSN `45/58000000`
- 目标位置 `4A/E0FFE518`
- 起始位置 `45/1772EA98`

现在在任何可用的 Postgres 中执行：

```
nik=# select pg_size_pretty(pg_lsn '4A/E0FFE518' - pg_lsn '45/58000000');
 pg_size_pretty
----------------
 22 GB
(1 row)

nik=# select pg_size_pretty(pg_lsn '45/58000000' - '45/1772EA98');
 pg_size_pretty
----------------
 1033 MB
(1 row)
```

第一个值是剩余的量，第二个值是已经处理的量。这里我们看到大约已经回放了 1 GiB，剩余 22 GiB。通过分析 Postgres 日志中的时间戳和当前时间，并假设 REDO 以恒定速度进行 (这一假设虽然粗略，但对于粗略估计已经足够，尤其是当我们多次重新估计并观察进程时)，我们可以估算出到达一致性点并允许 Postgres 接受连接还需要等待多久。

如果我们正在处理一个崩溃的 Postgres，通常 `pg_controldata` 不会提供 `Minimum recovery ending location` (显示为 0/0)。在这种情况下，我们可以检查 `$PGDATA/pg_wal` 来了解还剩多少要回放的 WAL 文件，按创建时间排序文件。这基于一个假设：当 Postgres 崩溃时，pg_wal 中需要回放的 WAL 文件的"尾部"就是我们所需要的。例如：

```
❯ ls -la /opt/homebrew/var/postgresql@15/pg_wal | grep 0000 | tail  -3
-rw-------     1 nik  admin  16777216 Sep 28 10:55 000000010000004A000000DF
-rw-------     1 nik  admin  16777216 Sep 28 10:55 000000010000004A000000E0
-rw-------     1 nik  admin  16777216 Sep 28 11:03 000000010000004A000000E1
```

最新的文件是 `000000010000004A000000E1`，因此我们可以粗略估算 REDO 进程将在 LSN `4A/E100000` 完成。

---

附加内容：如何模拟长时间启动/REDO 时间：

- 增加检查点之间的距离，调高 `max_wal_size` 和 `checkpoint_timeout` (例如 '100GB' 和 '60min')
- 创建一个大表 `t1`(例如，10-100M 行)：`create table t1 as select i, random() from generate_series(1, 100000000) i;`
- 执行一个对 `t1` 数据的长事务 (不必完成)：`begin; delete from t1;`
- 使用扩展模块 `pg_buffercache` 观察脏缓冲区的数量：
  - create extension `pg_buffercache`;
  - `select isdirty, count(*), pg_size_pretty(count(\*) * 8 * 1024) from pg_buffercache group by 1 \watch`
- 当脏缓冲区总大小达到几 GiB 时，通过向任何 Postgres 后端进程发送 `kill -9 <pid>` 来有意使服务器崩溃。
- 确保 Postgres 已关闭：`ps ax | grep postgres` 
- 启动 Postgres (例如，`pg_ctl -D <PGDATA> start`)
- 检查 Postgres 日志，确保 REDO 正在进行。

---

就是这样！希望这能帮助你保持冷静，让你在等待 Postgres 开启查询之门时更有意识。希望减少人们在启动时间过长时的压力。

请注意，这篇文章是一个操作指南的草稿；我写得很快，可能包含一些不准确之处。我将很快把所有操作指南草稿上传到一个 Git 仓库中，以保持其公开并欢迎修复和改进。敬请关注！ 

和往常一样，如果你觉得这篇文章有帮助，请订阅、点赞、分享和评论！💛

**更新 2024-05-07**：也请参见 PG15 中引入的 `log_startup_progress_interval`。