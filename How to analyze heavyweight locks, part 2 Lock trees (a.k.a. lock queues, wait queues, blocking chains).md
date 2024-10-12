# How to analyze heavyweight locks, part 2: Lock trees (a.k.a. "lock queues", "wait queues", "blocking chains")

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

参考[第一部分](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0022_how_to_analyze_heavyweight_locks_part_1.md)。

以下是一些值得参考的资源：

- [13.3. Explicit Locking](https://postgresql.org/docs/current/explicit-locking.html) — 官方文档 (尽管标题如此，但实际上仅涉及显式锁定)。
- [PostgreSQL rocks, except when it blocks: Understanding locks](https://www.citusdata.com/blog/2018/02/15/when-postgresql-blocks/) — @marcoslot 的博文。
- Egor Rogov 的书籍 [PostgreSQL 14 Internals](https://postgrespro.com/community/books/internals)，Part III "Locks".
- [PostgreSQL Lock Conflicts](https://postgres-locks.husseinnasser.com) — @hnasr 提供的一个参考类工具，用于研究各种锁类型之间的关系以及各种 SQL 命令获取的锁类型。

当出现锁问题时，我们通常需要：

1. 理解问题的性质和规模。
2. 考虑终止最初"有问题"的会话 — 即锁的根节点 — 以便尽快停止问题 (通常使用 `select pg_terminate_backend(<pid>);`)。

以下是一个高级查询，通常会显示"锁树森林" (因为可能有多个"根"会话，从中生长出多棵"树")：

```sql
\timing on
set statement_timeout to '100ms';

with recursive activity as (
  select
    pg_blocking_pids(pid) blocked_by,
    *,
    age(clock_timestamp(), xact_start)::interval(0) as tx_age,
    -- "pg_locks.waitstart" – PG14+ only; for older versions:  age(clock_timestamp(), state_change) as wait_age
    age(clock_timestamp(), (select max(l.waitstart) from pg_locks l where a.pid = l.pid))::interval(0) as wait_age
  from pg_stat_activity a
  where state is distinct from 'idle'
), blockers as (
  select
    array_agg(distinct c order by c) as pids
  from (
    select unnest(blocked_by)
    from activity
  ) as dt(c)
), tree as (
  select
    activity.*,
    1 as level,
    activity.pid as top_blocker_pid,
    array[activity.pid] as path,
    array[activity.pid]::int[] as all_blockers_above
  from activity, blockers
  where
    array[pid] <@ blockers.pids
    and blocked_by = '{}'::int[]
  union all
  select
    activity.*,
    tree.level + 1 as level,
    tree.top_blocker_pid,
    path || array[activity.pid] as path,
    tree.all_blockers_above || array_agg(activity.pid) over () as all_blockers_above
  from activity, tree
  where
    not array[activity.pid] <@ tree.all_blockers_above
    and activity.blocked_by <> '{}'::int[]
    and activity.blocked_by <@ tree.all_blockers_above
)
select
  pid,
  blocked_by,
  case when wait_event_type <> 'Lock' then replace(state, 'idle in transaction', 'idletx') else 'waiting' end as state,
  wait_event_type || ':' || wait_event as wait,
  wait_age,
  tx_age,
  to_char(age(backend_xid), 'FM999,999,999,990') as xid_age,
  to_char(2147483647 - age(backend_xmin), 'FM999,999,999,990') as xmin_ttf,
  datname,
  usename,
  (select count(distinct t1.pid) from tree t1 where array[tree.pid] <@ t1.path and t1.pid <> tree.pid) as blkd,
  format(
    '%s %s%s',
    lpad('[' || pid::text || ']', 9, ' '),
    repeat('.', level - 1) || case when level > 1 then ' ' end,
    left(query, 1000)
  ) as query
from tree
order by top_blocker_pid, level, pid
; -- to run query in a loop, use this instead of ";":   \watch 10
```

------

注意事项：

1. 此查询可以直接在 `psql` 中执行。对于其他客户端，移除反斜杠命令；在非交互模式中使用时，用 `;` 代替 `\watch`。

2. 根据文档，函数 `pg_blocking_pids(...)` 应谨慎使用：

   >Frequent calls to this function could have some impact on database performance, because it needs exclusive access to the lock manager's shared state for a short time.

不建议以自动化的方式使用它 (例如，放入监控中)。因此，我们在查询中设置了较低的 `statement_timeout` 以作为保护措施。

示例输出：

![Example output that shows the "forest of lock trees"](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0042_example_output.jpeg)

注意以下几点：

- 有两个锁树，分别由 PIDs 46015 和 46081 的根会话生成。
- 两者都在等待客户端 (`wait_event_type:wait_event` 显示为 `Client:ClientRead`)，并获取了一些锁(会话 46015 中最后一个查询是 `UPDATE`，会话 46081 中是 `DROP TABLE`)，并且正在持有这些锁。
- 第一个锁树 (根会话为 46015)更大，阻塞了 11 个会话，且树高为 4 层 (或称为深度，取决于使用的术语)。这种情况正是由于 `ALTER TABLE` 操作试图修改某个表，但被另一个会话阻塞，从而导致任何尝试操作该表的会话 (即使是 `SELECT` 查询) 也被阻塞的情况 (这一问题在 [Zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries) 进程了讨论) 
- 在分析期间，情况可能迅速变化，因此可能有必要添加时间戳或间隔 (例如，基于 `xact_start` 和 `pg_stat_activity` 中的 `state_change`)。还要注意，由于结果可能存在不一致 — 当我们从 `pg_stat_statements` 读取时，我们处理的是一些动态数据，而不是快照，因此如果会话状态快速变化，结果中存在一些偏差是正常的。通常，在得出结论和决定之前，分析查询的几个样本结果是有意义的。(译者著：此处应该是 `pg_stat_activity`，而不是 `pg_stat_statements`)