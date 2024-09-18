# How to speed up pg_dump when dumping large databases

在讨论了一系列较为复杂的话题之后，让我们放松一下，讨论一些简单的话题。

今天，我们将讨论在处理大数据量时，如何提升 `pg_dump` 的速度。

此处讨论了几种加速方法：

- 压缩
- 不保存到磁盘的转储/恢复
- 并行化的 `pg_dump`
- 自定义的高级并行化

## 选项1：考虑使用压缩

在磁盘或网络较弱的情况下，使用压缩是有意义的。如果使用的是文本格式 (`plain`，默认格式)，那么可以直接使用 `pg_dump ... | gzip`。请注意，如果磁盘 IO 和网络 (如有使用) 没有达到饱和状态 (可以使用 `iostat`、`top` 等工具查看资源使用情况)，这种方法并不能提升导出速度。

## 选项2：避免转储到磁盘，即时恢复

使用目录 (`directory`) 格式 (选项 `-Fd`，最灵活，我通常使用这种格式，除非有特殊情况) 时，默认会使用压缩 (默认是 `gzip`，也可以使用 `lz4` 和 `zstd`)。

还可以通过 `pg_dump -h ... | pg_restore` 以避免将转储写入磁盘，而是直接"实时"恢复数据。不幸的是，这只能在 pg_dump 创建纯文本转储时使用 — 对于 `directory` 的格式，这种方法不起作用。为了解决这个问题，有一个第三方工具叫做 [pgcopydb](https://github.com/dimitri/pgcopydb)。

## 选项3：pg_dump -j$N

对于拥有大量 CPU 的服务器，当处理多个表并使用目录格式创建转储时，并行化 (选项 `-j$N`) 非常有帮助。即使是单个但已分区的表，其行为也类似于多个表 — 因为物理上，转储将应用至多个表 (分区)。

考虑一个标准表 `pgbench_accounts`，由 [pgbench](https://postgresql.org/docs/current/pgbench.html) 创建，分为 30 个分区，且数据量较大，拥有 1 亿行数据 — 这些数据 (不包括索引) 的大小约为 12 GiB：

```shell
❯ pgbench -i -s1000 --partitions=30 test
dropping old tables...
creating tables...
creating 30 partitions...
generating data (client-side)...
100000000 of 100000000 tuples (100%) done (elapsed 53.64 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 146.30 s (drop tables 0.15 s, create tables 0.22 s, client-side generate 54.02 s, vacuum 25.19 s, primary keys 66.71 s).
```

`test` 数据库中的数据大小现在约为 12 GiB，其中大部分位于共享表 `pgbench_accounts` 中。

最灵活的转储格式是 `directory` 格式 — 选项 `-Fd`。测量几次时间以确保缓存已预热 (macbook m1，16 GiB RAM，PG15，`shared_buffers='4GB'`)：

```shell
❯ time pg_dump -Fd -f ./test_dump test
pg_dump -Fd -f ./test_dump test  45.94s user 2.65s system 77% cpu 1:02.50 total
❯ rm -rf ./test_dump
❯ time pg_dump -Fd -f ./test_dump test
pg_dump -Fd -f ./test_dump test  45.83s user 2.69s system 79% cpu 1:01.06 total
```

大约 61 秒。 

通过指定 `-j8` 启动 8 个并行 `pg_dump` 进程来加速：

```shell
❯ time pg_dump -Fd -j8 -f ./test_dump test
pg_dump -Fd -j8 -f ./test_dump test  57.29s user 6.02s system 259% cpu 24.363 total
❯ rm -rf ./test_dump
❯ time pg_dump -Fd -j8 -f ./test_dump test
pg_dump -Fd -j8 -f ./test_dump test  57.59s user 6.06s system 261% cpu 24.327 total
```

大约 24 秒 (相比之前的 61 秒)。

## 选项4：针对大型非分区表的高级并行化

当转储一个数据库时，如果其中有一个未分区的表比其他表大很多 (例如包含历史数据的巨大"日志"表)，标准的 pg_dump 并行化方式并不起作用。使用与上面相同的例子，但不分区：

```shell
❯ pgbench -i -s1000 test
dropping old tables...
creating tables...
generating data (client-side)...
100000000 of 100000000 tuples (100%) done (elapsed 51.71 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 116.23 s (drop tables 0.73 s, create tables 0.03 s, client-side generate 51.93 s, vacuum 4.41 s, primary keys 59.14 s).
❯ rm -rf ./test_dump
❯ time pg_dump -Fd -j8 -f ./test_dump test
pg_dump -Fd -j8 -f ./test_dump test  48.24s user 3.25s system 83% cpu 1:01.83 total
```

多个工作进程并没有多大帮助，因为大部分时间，只有一个进程在工作 (转储最大的非分区表)。

这是因为 `pg_dump` 的并行化是在表级别进行的，无法并行转储单个表。

要并行转储单个大表，需要使用自定义解决方案。为此，我们需要使用多个 SQL 客户端，如 psql，每个客户端在 `REPEATABLE READ` 隔离级别下工作 (`pg_dump` 也是在此隔离级别下工作的，参见[文档](https://www.postgresql.org/docs/current/transaction-iso.html))，且 (十分重要) 所有转储事务需要使用相同的快照。

流程如下：

1. 在一个连接中 (例如在一个 `psql` 会话中)，以 `REPEATABLE READ` 级别启动一个事务：

```sql
test=# start transaction isolation level repeatable read;
START TRANSACTION
```

2. 此事务必须保持开启直至整个过程结束 — 确保其保持开启状态。
3. 在同一会话中，使用函数 `pg_export_snapshot()` 获取快照 ID：

```sql
test=*# select pg_export_snapshot();
 pg_export_snapshot
---------------------
 00000004-000BF714-1
(1 row)
```

4. 在其他会话中，同样开启 `REPEATABLE READ` 事务，并设置为使用完全相同的快照 (当然，我们是并行运行这些会话的，这正是加速的重点)：

```sql
test=# start transaction isolation level repeatable read;
START TRANSACTION
test=*# set transaction snapshot '00000004-000BF714-1';
SET
```

5. 然后在每个会话中，分别转储大表的一部分，并确保访问方式是高效的 (使用 `Index Scan`；如果没有适当的索引，也可以使用隐藏列 `ctid` 的范围，并使用 [TID Scan](https://www.pgmustard.com/docs/explain/tid-scan)，避免 `Seq Scan`)。例如，转储 `pgbench_accounts` 表的第 1 部分：

```sql
test=# start transaction isolation level repeatable read;
START TRANSACTION
test=*# set transaction snapshot '00000004-000BF714-1';
SET
test=*# copy (select * from pgbench_accounts where aid <= 12500000) to stdout;
```

6. 对于其他较小的表，可以使用 `pg_dump` — 它也支持使用特定快照，通过选项 `--snapshot=...`。在这种情况下，我们需要使用 `--exclude-table-data=...` 排除大表数据，因为我们已单独处理它。在这种情况下，我们还可以使用并行导出。例如：

```shell
❯ pg_dump \
  -Fd \
  -j2 \
  -f ./everything_but_pgba_data.dump \
  --snapshot="00000004-000BF714-1" \
  --exclude-table-data="pgbench_accounts" \
  test
```

7. 在完成所有操作后，不要忘记关闭第一个事务 — 长时间运行的事务对 OLTP 工作负载有害。
8. 在恢复时，我们需要遵循通常的 pg_dump 顺序：先定义对象的 DDL (不包括索引)，然后加载数据，最后进行约束验证和索引创建。为此，我们可以利用 `directory` 格式转储，并使用 `pg_restore` 的选项 `-l` 和 `-L` 列出转储中的对象并分别筛选它们进行恢复。

一篇关于在进行数据库转储时如何处理快照的优秀文章："[Postgres 9.5 feature highlight - pg_dump and external snapshots](https://paquier.xyz/postgresql-2/postgres-9-5-feature-highlight-pg-dump-snapshots/)"。该文章中提到的一个非常有趣的额外考虑是与转储的一个特殊情况有关：逻辑副本的初始化。可以使用与逻辑复制槽位置同步的自定义转储方法，但这种复制槽的创建必须通过复制协议完成(`CREATE_REPLICATION_SLOT foo3 LOGICAL test_decoding;`)，而不是通过 SQL (`select * from pg_create_logical_replication_slot(...);`)。

---

今天的分享就到这里。祝大家在生产环境中能达到出色的转储速度 (2+ TiB/h)！