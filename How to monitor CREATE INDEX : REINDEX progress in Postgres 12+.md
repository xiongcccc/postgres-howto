# How to monitor CREATE INDEX / REINDEX progress in Postgres 12+

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

要观察长时间运行的索引创建或重建的进度，你可以使用以下查询：

~~~sql
select
   now(),
   query_start as started_at,
   now() - query_start as query_duration,
   format('[%s] %s', a.pid, a.query) as pid_and_query,
   index_relid::regclass as index_name,
   relid::regclass as table_name,
   (pg_size_pretty(pg_relation_size(relid))) as table_size,
   nullif(wait_event_type, '') || ': ' || wait_event as wait_type_and_event,
   phase,
   format(
           '%s (%s of %s)',
           coalesce((round(100 * blocks_done::numeric / nullif(blocks_total, 0), 2))::text || '%', 'N/A'),
           coalesce(blocks_done::text, '?'),
           coalesce(blocks_total::text, '?')
   ) as blocks_progress,
   format(
           '%s (%s of %s)',
           coalesce((round(100 * tuples_done::numeric / nullif(tuples_total, 0), 2))::text || '%', 'N/A'),
           coalesce(tuples_done::text, '?'),
           coalesce(tuples_total::text, '?')
   ) as tuples_progress,
   current_locker_pid,
   (select nullif(left(query, 150), '') || '...' from pg_stat_activity a where a.pid = current_locker_pid) as current_locker_query,
   format(
           '%s (%s of %s)',
           coalesce((round(100 * lockers_done::numeric / nullif(lockers_total, 0), 2))::text || '%', 'N/A'),
           coalesce(lockers_done::text, '?'),
           coalesce(lockers_total::text, '?')
   ) as lockers_progress,
   format(
           '%s (%s of %s)',
           coalesce((round(100 * partitions_done::numeric / nullif(partitions_total, 0), 2))::text || '%', 'N/A'),
           coalesce(partitions_done::text, '?'),
           coalesce(partitions_total::text, '?')
   ) as partitions_progress,
   (
      select
         format(
                 '%s (%s of %s)',
                 coalesce((round(100 * n_dead_tup::numeric / nullif(reltuples::numeric, 0), 2))::text || '%', 'N/A'),
                 coalesce(n_dead_tup::text, '?'),
                 coalesce(reltuples::int8::text, '?')
         )
      from pg_stat_all_tables t, pg_class tc
      where t.relid = p.relid and tc.oid = p.relid
   ) as table_dead_tuples
from pg_stat_progress_create_index p
        left join pg_stat_activity a on a.pid = p.pid
order by p.index_relid
; -- in psql, use "\watch 5" instead of semicolon
~~~

相同的查询，[采用更好的格式](https://gitlab.com/-/snippets/2138417)。

查询工作原理如下：

1. 此查询的核心依赖于 `pg_stat_progress_create_index`，该视图是在 PostgreSQL 12 中引入的。
2. 文档中还列出了创建索引所涉及的所有阶段。特别是像 `CREATE INDEX CONCURRENTLY `和 `REINDEX CONCURRENTLY` (即 CIC 和 RC) 这类高级变体，这些方法用于高负载的生产系统，采用非阻塞方式运行，但耗时更长，CIC 和 RC 在执行过程中引入了更多阶段。当前阶段可以在输出的 `phase` 列中查看。
3. 查询展示了索引名称 (在 CIC/RC 的情况下，是临时的) 和表名称 (使用了一个有用的技巧，通过 OID 转换成实际名称： `index_relid::regclass AS index_name`)。此外，表的大小对于形成总体持续时间的预期至关重要 — 表越大，创建索引的时间越久。
4. 查询还利用 `pg_stat_activity` (`pgsa`) 提供了大量额外的有用信息：
   - Postgres 后端的 PID
   - 实际使用的 SQL 查询
   - 查询开始的时间 (`query_start`)，用于计算已用时长 (`query_duration`)
   - `wait_event_type` 和 `wait_event`，帮助我们了解进程当前在等待什么
   - 当发生此类事件时，它还使用 (在单独的子查询中) 获取阻塞我们进程的会话的信息 (`current_locker_pid` 和`current_locker_query`)

5. `format(...)` 函数非常有用，可以将数据整合成便捷的形式，避免处理 `NULL` 值时出现问题。如果我们使用常规的字符串连接操作而不使用 `coalesce(...)`，就可能会遇到 `NULL` 值的问题。
6. 在某些情况下，我们使用 `coalesce(...)` 来填充特殊符号以处理缺失值 (即 `IS NULL`) — 例如 "?" 或 "N/A"。
7. 另一个有趣的技巧是结合使用 `coalesce(...)` 和 `nullif(...)`。后者可以避免除以零的错误 (通过将 `0` 替换为 `NULL`，使除法结果也为 `NULL`)，而前者再次用于将 `NULL` 替换为一些非空值 (在此示例中为 'N/A')。

在 `psql` 中执行时，使用 `\watch [seconds]` 命令可以方便地在循环中运行此查询，并实时观察进度。

![tracking the progress of index building/rebuilding](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0015_reindex.gif)