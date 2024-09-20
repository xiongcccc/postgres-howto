# How to find query examples for problematic pg_stat_statements records

// 我每天发布一篇新的 PostgreSQL "howto" 文章。和我一起踏上这段旅程 — 订阅、提供反馈、分享！

在几个小时后，我将在 DjangoCon US 上展示我的 "无缝 Postgres 查询优化"教程，所以现在是讨论从 `pg_stat_statements` (`pgss`) 过渡到 `EXPLAIN` 的好时机。

## 从 pgss 跳到 EXPLAIN 的问题

一旦确定了有问题的 `pgss` 记录 (这是我们在第 5-7 天讨论的查询宏观优化的主题)，首先要做的是理解优化的方向。

`pgss` 记录需要优化的两种最常见的基本情况：

1. 如果调用次数 (`calls`) 非常高 (大量 QPS，每秒查询次数)，主要的优化方法是减少这个数字 — 这需要在客户端 (应用程序) 侧完成。
2. 如果 `mean_exec_time + mean_plan_time` 很高 (对于 OLTP 环境 — 网络和移动应用 — `100ms` 应被视为[慢](https://postgres.ai/blog/20210909-what-is-a-slow-sql-query))，这时我们需要应用查询微观优化 — 使用 `EXPLAIN`。

当然，也常常会同时遇到这两种情况 — 查询频繁且延迟较差。

在本文中，我们不会讨论如何使用 `EXPLAIN` 和 `EXPLAIN (ANALYZE, BUFFERS)`。相反，我们将专注于为 EXPLAIN 找到适当的材料 — 需要研究和改进的特定查询示例。

值得记住的是，单个 `pgss` 记录可能与执行方式不同的单个查询相关联 — 使用不同的执行计划。以下是一个基本的例子，用来说明这一点：

```sql
nik=# create table t1 as select 1::int8 as c1;
SELECT 1
nik=# insert into t1 select 2
from generate_series(1, 1000000);
INSERT 0 1000000
nik=# create index on t1(c1);
CREATE INDEX
nik=# vacuum analyze t1;
VACUUM
nik=# explain select from t1 where c1 = 1;
                                   QUERY PLAN
-------------------------------------------------------------------------
 Index Only Scan using t1_c1_idx on t1  (cost=0.42..4.44 rows=1 width=0)
   Index Cond: (c1 = 1)
(2 行)

nik=# explain select from t1 where c1 = 2;
                             QUERY PLAN
------------------------------------------------------------
 Seq Scan on t1  (cost=0.00..16981.01 rows=1000001 width=0)
   Filter: (c1 = 2)
(2 行)
```

这里的两个查询在 pgss 中都会被记录为 `select * from t1 where c1 = $1`。但执行计划是不同的，因为对于 `c1 = 1`，选择性很高，而对于 `c1 = 2`，则非常差 (目标是表中除 1 以外的所有行)。

这意味着，仅仅查看显示较差查询延迟的 `pgss` 记录，我们无法快速跳转到使用 `EXPLAIN` — 我们需要找到特定的查询示例来处理。

下面，让我们来讨论解决这个问题的方式。

## 选项1：猜测

在某些情况下，猜测可能没问题。但是，我遇到过非常糟糕的情况，当我在猜测上犯错时浪费了很多时间。例如，在一个案例中，处理一个布尔列，我决定使用选择性非常差的值，并花了很多时间优化这种情况，才意识到应用程序代码永远不会使用它。

可能会尝试使用 `pg_statistic` 来改进猜测。但不幸的是，在一般情况下，这并不能很好地工作，因为缺乏多列统计信息 (除非显式创建) — 如果没有它，我们在很多情况下都会有不切实际的参数变体。

因此，这种方法有限，只能用于简单的情况。

## 选项2：从日志中获取示例

可以在 Postgres 日志中找到示例 — 当然，前提是它们已被记录 (通常通过参数 `log_min_duration_statement` 或 `auto_explain` 扩展)。要为给定的 `pgss` 记录找到示例，我们需要能够建立已记录查询和 `pgss` 记录之间的关联。有两个选项：

- 对于 PG14+，[compute_query_id](https://postgresqlco.nf/doc/en/param/compute_query_id/) 选项可以将与 pg_stat_statements 中使用相同 `queryid` 的值添加到日志条目中。
- 或者，我们可以使用优秀的库 [libpg_query](https://github.com/pganalyze/libpg_query) (也有 Ruby、Go、Python 等版本)。它可以同时应用于标准化的 (pgss 记录) 和单个查询，生成所谓的指纹，然后可用于找到我们需要的关系。

总的来说，使用 Postgres 日志来查找查询示例是一个好方法，但对于负载较重的系统，如果无法记录所有查询，它只能提供非常慢的示例 — 那些超过 `log_min_duration_statement` 的查询 (通常设置为相当高的值，例如 500ms )。

这种情况可以通过采样和降低慢查询的阈值来改善，甚至完全取消阈值。相关参数：

- [log_min_duration_sample](https://postgresqlco.nf/doc/en/param/log_min_duration_sample/) (PG13+)
- [log_statement_sample_rate](https://postgresqlco.nf/doc/en/param/log_statement_sample_rate/) (PG13+)
- [log_transaction_sample_rate](https://postgresqlco.nf/doc/en/param/log_transaction_sample_rate/) (PG12+)

或者，可以使用 `auto_explain`，它也支持采样 (`auto_explain.sample_rate`，目前所有受支持的版本)，并且它非常有吸引力，因为可以带来生产环境中的执行计划。安装 `auto_explain` 应该经过仔细的测试和对开销的分析。

## 选项3：从 pg_stat_activity 中采样查询

这种方法可能很有吸引力，因为它不需要我们打开过多查询的昂贵日志记录。即使是低延迟查询，如果它们足够频繁，最终也会被捕获，只要我们足够长时间地观察 `pg_stat_activity.query`。

然而，这里有两个重要的限制。

首先，`pg_stat_activity` 中的列 `query_id` (用于将 `pg_stat_activity` 的样本与 `pgss` 记录连接) 是最近才添加的，在 PG14 中。对于旧版本，我们最终需要使用一些正则表达式 (实现可能很麻烦且脆弱) 或 libpg_query 的指纹 (意味着我们需要采样所有的 `pg_stat_activity` 记录，然后对它们进行后处理)。因此，这种方法在 PG14+ 中使用更好。

其次，默认情况下，`pg_stat_activity.query` 被截断为 1024 个字符 — 这是由  [track_activity_query_size ](https://postgresqlco.nf/doc/en/param/track_activity_query_size/)定义的，默认值为 1024。建议将其大幅增加 — 例如，设置为 10k，以便能够采样和分析更大的查询。不幸的是，更改此设置需要重启。

## 选项4：eBPF

这个选项尚未完全开发，但有人认为它在未来可能是一个非常好的替代方案：使用 eBPF 来采样查询 (作为 `pg_stat_activity` 的替代)，甚至采样查询并标准化 (作为 `pg_stat_activity` 和 `pgss` 的替代)。

因此 — 待定。同时，查看这些有趣的资源：

- [pgtracer](https://github.com/Aiven-Open/pgtracer) (在 PostgresTV 上的演示：https://youtube.com/watch?v=tvJgMV-8nfU)
- [使用 perf 和 eBPF 分析 Postgres 性能问题](https://youtu.be/HghP4D72Noc?si=tFuQuDWKrScJ8w2i&t=1389) — Andres Freund 的演讲视频，描述了如何结合 `pg_stat_statements` 和 BPF (代码：https://github.com/anarazel/pg-bpftrace)

## 选项5：通用计划

待定：

- pg16 中 EXPLAIN 的新特性
- 低于 16 版本的技巧

## 总结

- 在 PG14+ 中，使用 `compute_query_id` 在 Postgres 日志和 `pg_stat_activity` 中都有 `query_id` 值。
- 增加 `track_activity_query_size` (需要重启)，以便在 `pg_stat_activity` 中跟踪更大的查询。
- 组织工作流程，将 `pg_stat_statements` 的记录与来自日志和 `pg_stat_activity` 的查询示例相结合，这样当涉及到查询优化时，你就有了可用于 `EXPLAIN (ANALYZE, BUFFERS)` 的良好示例。

提醒一下，一旦你有了好的示例，验证优化思路的最佳 (最快&代价最低)方法是使用经过适当调整的精简克隆 — 查看 [DBLab](https://twitter.com/Database_Lab)。