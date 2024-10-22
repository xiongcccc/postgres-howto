# How to make the non-production Postgres planner behave like in production

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

为了实现查询优化的目标 — 运行 `EXPLAIN (ANALYZE, BUFFERS)` 并验证各种优化想法，比如新索引，确保 Postgres 规划器的行为与生产数据库的规划器完全相同或者相似至关重要。幸运的是，无论非生产机器上可用的资源 (CPU、RAM、磁盘 IO) 如何，这都是可以实现的。

此处描述的技术已用于 DBLab Engine [@Database_Lab](https://twitter.com/Database_Lab)，支持数据库克隆/分支，以进行查询优化和 CI/CD 中的数据库测试。

要实现规划器的生产/非生产行为一致性，需要两个组件：

1. 相匹配的数据库设置
2. 相同或非常相似的统计信息 (`pg_statistic` 中的内容)

## 相匹配的数据库设置

在生产数据库上使用以下查询获取会影响规划器行为的非默认设置列表：

```sql
select
  format(e'%s = %s', name, setting) as configs
from
  pg_settings
where
  source <> 'default'
  and (
    name ~ '(work_mem$|^enable_|_cost$|scan_size$|effective_cache_size|^jit)'
    or name ~ '(^geqo|default_statistics_target|constraint_exclusion|cursor_tuple_fraction)'
    or name ~ '(collapse_limit$|parallel|plan_cache_mode)'
  );
```

注意：

- 规划器的行为不依赖于实际可用的资源，如 CPU 或 RAM，也不依赖于操作系统、文件系统或其设置。
- `shared_buffers `的值无关紧要(!) — 它只会影响执行器的行为和缓冲池的命中/读取比率。对规划器重要的是 `effective_cache_size`，你可以将其设置为显著超过实际可用的 RAM，以"愚弄"规划器 (好的出发点)，从而实现与生产环境规划器行为一致的目标。例如，在生产环境中有 1 TiB 的 RAM 和 `shared_buffers = '250GB'`，并且 `effective_cache_size = '750GB'`，你可以在 `shared_buffers = '2GB'` 和 `effective_cache_size = '750GB'` 的 8 GiB 机器上有效地分析和优化查询，规划器会假设你有大量的 RAM，从而选择最优的执行计划。
- 同样，你可以在生产环境中使用快速的 SSD，设置`random_page_cost = 1.1`，但在测试环境中使用廉价低速的磁盘。如果在测试环境中同样使用 `random_page_cost = 1.1`，规划器会认为随机访问并不昂贵。尽管执行仍会很慢，但是所选择的计划和数据量 (实际行数、缓冲区数量) 将与生产环境非常接近。

## 相匹配的统计信息

一个有趣的想法是将生产环境中的 `pg_statistic` 内容转储并恢复到更小的测试环境中，而无需实际数据。尽管这种方法不常见，但可能可以使用 [pg_dbms_stats](https://github.com/ossc-db/pg_dbms_stats/blob/master/doc/pg_dbms_stats-en.md) 来实现。

> 🎯 TODO：测试 pg_dbms_stats 并查看其效果。

一个简单的思路是使用相同大小的数据库。这里有两个选项：

- **选项1：物理拷贝**。可以使用 `pg_basebackup`，通过 `rsync` 或其他工具复制 PGDATA，或从物理备份 (比如 `WAL-G` 或 `pgBackRest`) 恢复。
- **选项2：逻辑拷贝**。可以通过转储/恢复，或创建逻辑副本来完成。

注意：

- 物理拷贝的方式效果最佳：不仅"实际行数"匹配，运行 `EXPLAIN (ANALYZE, BUFFERS)` 时提供的缓冲区数量也匹配，因为页面数量 (`relpages`) 相同，膨胀保留等。这是进行优化和微调查询的最佳方法。
- 逻辑方式的行数相匹配，但表和索引的大小会不同 — `relpages` 在新节点上更小，膨胀没有保留，元组通常以不同的顺序存储 (这种膨胀可以称为"好的膨胀"，我们希望在测试环境中保留它以匹配生产环境的状态)。这种方法仍然可以实现相当高效的查询优化工作流程，此外，生产环境中保持低膨胀率变得更为重要。
- 在执行转储/恢复后，你必须显式运行 `ANALYZE` (或使用 `vacuumdb --analyze -j <number of workers>`)，以在 `pg_statistic` 中收集统计信息，因为 `pg_restore` (或 `psql`) 不会自动运行。
- 如果需要修改数据库内容以移除敏感数据，这几乎肯定会影响规划器的行为。对于某些查询，影响可能较小，但对于其他查询，影响可能非常大，导致查询优化几乎变得不可能。这种负面影响通常比转储/恢复丢失膨胀造成的影响更严重，因为：
  - 转储/恢复会影响 `relpages` (丢失膨胀) 和元组顺序，但不会影响 `pg_statistic` 中的内容。
  - 移除敏感数据不仅会重新排列元组，产生无关的膨胀 ("坏膨胀")，还会丢失生产环境中 `pg_statistic` 的重要部分。

## 总结

为了让非生产环境中的 Postgres 规划器与生产环境中一致，执行以下两步：

1. 调整非生产环境中的某些 Postgres 设置 (与规划器相关，如`work_mem`)，使其与生产环境匹配。
2. 尽可能从生产环境复制数据库，优先选择物理方式的拷贝，并且可能的话，尽量避免数据修改。对于超快速交付克隆，考虑使用 DBLab Engine [@Database_Lab](https://twitter.com/Database_Lab)。