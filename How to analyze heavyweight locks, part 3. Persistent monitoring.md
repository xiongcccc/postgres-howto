# How to analyze heavyweight locks, part 3. Persistent monitoring

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

如果活跃会话 (`pg_stat_activity`，`state='active'`) 的峰值伴随着独占锁 (`pg_locks`，`mode ~* 'exclusive'`) 的峰值发生，我们需要分析被阻塞和阻塞的会话，以了解根本原因。

之前讨论过临时的分析方法：

- [Day 22, How to analyze heavyweight locks, part 1](https://xiongcccc.github.io/postgres-howtos/#/./docs/22)
- [Day 42, How to analyze heavyweight locks, part 2: Lock trees (a.k.a. "lock queues", "wait queues", "blocking chains")](https://xiongcccc.github.io/postgres-howtos/#/./docs/42)
- 特别是对于 DDL 被阻塞的情况： [Day 71, How to understand what's blocking DDL](https://xiongcccc.github.io/postgres-howtos/#/./docs/71)

然而，如果事件已经结束，临时的分析方法也无济于事。部分原因是，即使设置了 `log_lock_waits = 'on'`，Postgres 日志也只报告"受害者" (被阻塞的会话) 的查询文本 (日志条目的 `STATEMENT` 部分)，关于阻塞会话，只能获取到 `PID` 的信息。

要对过去的事件进行排障并了解趋势，我们需要在监控中实现锁分析，提供关于被阻塞和阻塞会话的详细信息。

以下是  [@VKukharik](https://twitter.com/VKukharik) 为 [pgwatch2 - Postgres.ai Edition](https://hub.docker.com/r/postgresai/pgwatch2) 开发的查询。它没有使用函 数 `pg_blocking_pids()`，因为由于可能的观察者效应，它并不安全。

```sql
with sa_snapshot as (
  select *
  from pg_stat_activity
  where
    datname = current_database()
    and pid <> pg_backend_pid()
    and state <> 'idle'
)
select
  (extract(epoch from now()) * 1e9)::bigint as epoch_ns,
  waiting.pid as waiting_pid,
  waiting_stm.usename::text as tag_waiting_user,
  waiting_stm.application_name::text as tag_waiting_appname,
  waiting.mode as waiting_mode,
  waiting.locktype as waiting_locktype,
  waiting.relation::regclass::text as tag_waiting_table,
  waiting_stm.query as waiting_query,
  (extract(epoch from (now() - waiting_stm.state_change)) * 1000)::bigint as waiting_ms,
  blocker.pid as blocker_pid,
  blocker_stm.usename::text as tag_blocker_user,
  blocker_stm.application_name::text as tag_blocker_appname,
  blocker.mode as blocker_mode,
  blocker.locktype as blocker_locktype,
  blocker.relation::regclass::text as tag_blocker_table,
  blocker_stm.query as blocker_query,
  (extract(epoch from (now() - blocker_stm.xact_start)) * 1000)::bigint as blocker_tx_ms
from pg_catalog.pg_locks as waiting
join sa_snapshot as waiting_stm on waiting_stm.pid = waiting.pid
join pg_catalog.pg_locks as blocker on
  waiting.pid <> blocker.pid
  and blocker.granted
  and waiting.database = blocker.database
  and (
    waiting.relation  = blocker.relation
    or waiting.transactionid = blocker.transactionid
  )
join sa_snapshot as blocker_stm on blocker_stm.pid = blocker.pid
where not waiting.granted;
```

将此查询添加到监控中，当发生涉及锁争用峰值的事件时，可以观察到如下内容：

![img](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0073_01.jpg)

它提供了被阻塞和阻塞会话的所有详细信息，为排障和分析根本原因提供支持。

![img](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0073_02.jpg)

## 我见

维基百科：**观测者效应** (Observer effect)，是指“观测”这种行为对被观测对象造成一定影响的效应。

不止一篇文中提到了观测者效应，pg_blocking_pids 便是其中之一。

>Frequent calls to this function could have some impact on database performance, because it needs exclusive access to the lock manager's shared state for a short time.