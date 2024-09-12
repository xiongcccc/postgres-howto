# EXPLAIN ANALYZE 还是 EXPLAIN (ANALYZE, BUFFERS)?

在分析 PostgreSQL 的查询执行计划时，我总是推荐使用 `BUFFERS` 选项：

```sql
explain (analyze, buffers) <query>;
```

## 示例

```sql
test=# explain (analyze, buffers) select * from t1 where num > 10000 order by num limit 1000;
QUERY PLAN
----------------------------------------------------------
Limit  (cost=312472.59..312589.27 rows=1000 width=16) (actual time=314.798..316.400 rows=1000 loops=1)
Buffers: shared hit=54173
...
Rows Removed by Filter: 333161
Buffers: shared hit=54055
Planning Time: 0.212 ms
Execution Time: 316.461 ms
(18 rows)
```

如果 `EXPLAIN ANALYZE` 未使用 `BUFFERS`，分析将缺少缓冲池 IO 的相关信息。

**推荐使用 `EXPLAIN (ANALYZE, BUFFERS)` 而不仅仅是 `EXPLAIN ANALYZE` 的原因**

1. 每个计划节点的缓冲池 IO 操作都可见。
2. 可以了解涉及的数据量 (注意：buffer hits 可能涉及多次"命中"同一个缓冲区)。
3. 如果分析的重点是 IO 数字，那么即使在较弱的硬件上 (较少的内存，较慢的磁盘)，仍然可以获得可靠的查询优化数据。

为了更好地理解，建议将缓冲区数量转换为字节。在大多数系统中，1 个缓冲区为 8 KiB。因此，10 次缓冲区读取即为 80 KiB。 然而，要注意可能的混淆：需要记住的是，`EXPLAIN (ANALYZE, BUFFERS)` 提供的数字并不是数据量，而是 IO 次数，即完成的 IO 工作量。例如，针对内存中的单个缓冲区，可能会有 10 次命中——在这种情况下，我们的缓冲池中并没有 80 KiB 的数据，**而是处理了 80 KiB**，多次处理了同一个缓冲区。实际上，命名不太准确：它显示为 `Buffers: shared hit=5`，但这个数字其实表示的是 `buffer hits`，而不是`buffers hit`——它代表操作次数，而不是数据大小。

## 总结

始终使用 `EXPLAIN (ANALYZE, BUFFERS)`，而不仅仅是 `EXPLAIN ANALYZE`——这样你可以看到 Postgres 在执行查询时实际完成的 IO 工作量。

这有助于更好地了解所涉及的数据量。更好的是，你可以开始将缓冲区数量转换为字节——只需将它们乘以块的大小 (大多数情况下是 8 KiB)。

在优化过程中不要过于关注时间数字——这可能听起来违反直觉，但这可以让你忘记环境之间的差异。而且这也允许你使用轻量克隆——看看 Database Lab Engine 以及其他公司在这方面做了什么。

最后，当你成功减少 BUFFERS 数字时，这意味着 Postgres 在执行查询时将需要更少的缓冲区，减少了 IO，降低了争用的风险，并为其他任务留出了更多的缓冲区空间。遵循这一方法，最终可能对数据库的整体性能产生积极影响。

关于这一主题的博客文章：[EXPLAIN (ANALYZE) needs BUFFERS to improve the Postgres query optimization process](https://postgres.ai/blog/20220106-explain-analyze-needs-buffers-to-improve-the-postgres-query-optimization-process)





