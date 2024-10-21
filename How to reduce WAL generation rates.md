# How to reduce WAL generation rates

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

在一个快速增长的项目中，其中一个重要优化方向便是减少生成的 WAL (预写日志) 的数量。

## 为什么这很重要

WAL 是 PostgreSQL 的核心机制，用于在发生故障时进行恢复、备份和复制。

每秒生成的 WAL 数据越多，意味着每秒需要复制和备份的数据量也越大，因此各种风险也可能会增加：
复制延迟、WAL 归档延迟以及发生故障后，恢复时间也会增加。

## 如何衡量 WAL 生成速率

当新事务创建一条 WAL 记录时，它会分配一个 LSN (日志序列号)。监控当前 LSN 的位置非常简单：

```sql
select pg_current_wal_lsn();
```

在任何 PostgreSQL 监控中都应该包含这个指标。

两个 LSN 之间的差值就是这段时间内生成的字节数，Postgres 可以执行相关的计算 — 使用 `pg_lsn` 数据类型：

```sql
nik=# select pg_size_pretty('3/ED5F1E0'::pg_lsn - '0/110A1E0');
 pg_size_pretty
----------------
 12 GB
(1 row)
```

如果监控没有此功能，你可以通过查看以下内容了解每小时或每天生成了多少 WAL 数据：

-  `pg_wal` 目录中的 WAL 文件名
- 检查备份 (例如，检查 WAL-G 创建的两个完整备份的名称：`wal-g backup-list --detail`)

这两种方法都应该可以帮助你获取与两个遥远时间点相对应的两个 LSN 值。

要了解更多细节，参照 [Day 9: How to understand the LSN values and WAL file name](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0009_lsn_values_and_wal_filenames.md)

## 查询分析中的 WAL 指标

PostgreSQL 13 及以后的版本中，`pg_stat_statements` 和 `EXPLAIN` 都能提供与 WAL 相关的指标：

1. `pg_stat_statements`：`wal_records`、`wal_fpi`、`wal_bytes` 等指标([docs](https://postgresql.org/docs/current/pgstatstatements.html))。一个简单的分析例子：

```sql
with time_period(delta_sec) as (
  select extract(epoch from now() - stats_reset)
  from pg_stat_statements_info
)
select
  now(),
  delta_sec,
  round(wal_bytes / delta_sec) as wal_bytes_per_sec,
  round(wal_bytes / calls) as wal_bytes_per_call,
  queryid
from
  pg_stat_statements,
  time_period
order by wal_bytes desc
limit 25;
```

2. `EXPLAIN`：使用 `explain (analyze, buffers, wal)` 来查看执行计划中的 WAL 指标：

```sql
nik=# explain (analyze, buffers, wal) insert into t select i from generate_series(1, 100000) as i;
                                                             QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------
 Insert on t  (cost=0.00..1000.00 rows=0 width=0) (actual time=159.378..159.378 rows=0 loops=1)
   Buffers: shared hit=100895 dirtied=442 written=442
   WAL: records=100000 fpi=1 bytes=5900343
   ->  Function Scan on generate_series i  (cost=0.00..1000.00 rows=100000 width=4) (actual time=26.179..30.696 rows=100000 loops=1)
 Planning Time: 1.945 ms
 Execution Time: 160.483 ms
(6 rows)
```

## 全页写

pg_stat_statements 和 EXPLAIN 结果中的 "fpi" 指标表示发生了多少次全页镜像 (全页写)。

如果配置参数 `full_page_write` 为 on (默认情况下也是如此；通过 `show full_page_writes;` 检查)，则在每个检查点之后，页面中的第一次更改会导致整个页面被写入 WAL 中。默认情况下，页面大小为 8 KiB，大多数 Postgres 安装中都是如此 (通过 `show block_size` 检查)。这意味着，如果只有很小一部分页面发生更改，在检查点之后仍需要写入整个页面。同一页面中的后续写入将正常进行 (只有更改会记录到 WAL 中)，但是一旦发生新的检查点，那么需要再次先进行新的全页写。

更多信息可以参考以下文档：

- Hironobu Suzuki 的 "The Internals of PostgreSQL"。第 9 章 "Write Ahead Logging – WAL"，[9.1.3. Full-Page Writes](https://interdb.jp/pg/pgsql09.html#_9.1.3)
- Egor Rogov 的 "PostgreSQL 14 internals"，"10.4 Recovery" 章节
- Postgres wiki：[Full page writes](https://wiki.postgresql.org/wiki/Full_page_writes)
- Tomas Vondra, [On the impact of full-page writes (2016)](https://2ndquadrant.com/en/blog/on-the-impact-of-full-page-writes/)

## 优化思想

以下是一些减少 WAL 生成量的优化建议：

1. **检查点优化：增加检查点间隔**

   增加检查点之间的间隔有两个好处，特别是当工作负载包含许多随机写入 (不是连续的，比如在使用 `COPY` 进行大量数据加载时)时：

   - 更少的全页写
   - 减少对同一缓冲区的重复刷新 (刷新后，可能会由于新的写入而很快再次变脏)。

   为了增加间隔，我们只需要增加 `max_wal_size` (默认 `1GB`) 和 `checkpoint_timeout` (默认 5 分钟)。但这需要权衡利弊：检查点之间的间隔越大，意味着在各种情况下需要重放更多的 WAL 才能达到一致性点：

   - 崩溃后的恢复时间更长

   - 从备份中配置新节点的时间更长。

   不过，这种方法对于较大的规格来说是必不可少的，因为它可以带来实质性的改进。

   调整 `max_wal_size` 和 `checkpoint_timeout` 不需要重新启动。

2. **检查点优化：开启 WAL 压缩**

   考虑 `wal_compression` — 可以压缩全页写，大多数情况下，这样做是值得的 (尽管有些报告称这会导致更高的 CPU 使用率，并决定回退更改)。

   修改此参数不需要重启。

3. **优化查询**

   使用 `pg_stat_statements` 和 `EXPLAIN` 来定位生成大量 WAL 的查询，并进行优化。

   优化写入的方法之一是鼓励 Postgres 使用更多的 `HOT UPDATE` (为此，我们需要确保页面有可用空间 —  令人惊讶的是，些许膨胀在此处是有益的 — 并且不会对表进行过度索引，因此我们正在更改的列不参与索引定义)。

4. **删除未使用和冗余的索引**

   在查询优化期间，请记住，对于非 `HOT UPDATE` 和 `INSERT`，生成的 WAL 的量取决于表的索引数量。索引清理是一种非常有用的方法，可以减少此类写入产生的 WAL 量。

5. **分区**

   对大 (100+ GiB) 表进行分区可以提高写入的数据局部性 — 例如，如果表未分区，那么一堆行的更新可能会分散在许多页面中，而使用定义了旧分区 (几乎没有接收写入) 和包含新数据分区的分区模式，大多数写入将集中在新分区中，这有助于降低 WAL 生成率。