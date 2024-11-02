# How to understand what's blocking DDL

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

在 [day 60](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0060_how_to_add_a_column.md)，我们讨论了如何添加列，并提到在高负载下需要设置较低的 `lock_timeout` 以及进行重试 (参照：[Zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries))。

当时将 `max_attempts` 设置为 1000，可能有点过高，但有一个有趣的问题：如果在多次重试后 DDL 仍未成功，如何了解究竟是什么阻塞了它？

首先要做的是启用 `log_lock_waits`。这样，在 `deadlock_timeout` (默认为 1 秒，此设置用于定义死锁检测发生的时间) 之后，你会看到一些有关被阻塞会话的信息。

但是我们不希望将 `lock_timeout` 设置超过 1 秒 — 这侵入性过强。解决方案是在会话中设置较低的 `deadlock_timeout`。示例 (假设有另一个会话读取表 `t` 并保持事务处于开启的状态)：

```sql
postgres=# set lock_timeout to '100ms';
SET

postgres=# set deadlock_timeout to '50ms';
SET

postgres=# set log_lock_waits to 'on';
SET

postgres=# alter table t1 add column new_c int8;
ERROR:  canceling statement due to lock timeout
```

在日志中，我们有两条记录 — 一条是在约 50 毫秒 (`deadlock_timeout`) 之后，包含一些有关锁等待的详细信息，另一条是我们的语句在 100 毫秒之后失败的错误信息：

```bash
2023-12-06 19:12:21.823 UTC [197] LOG:  process 197 still waiting for AccessExclusiveLock on relation 16384 of database 13481 after 54.022 ms
2023-12-06 19:12:21.823 UTC [197] DETAIL:  Process holding the lock: 211. Wait queue: 197.
2023-12-06 19:12:21.823 UTC [197] STATEMENT:  alter table t1 add column new_c int8;
2023-12-06 19:12:21.874 UTC [197] ERROR:  canceling statement due to lock timeout
2023-12-06 19:12:21.874 UTC [197] STATEMENT:  alter table t1 add column new_c int8;
```

遗憾的是，当达到 `deadlock_timeout`  (在此例中为 50 毫秒) 时写入的日志消息不包含阻塞者的详细内容，只显示了 `PID` (`Process holding the lock: 211`)。如果幸运的话，我们可能会在 Postgres 日志中查看该会话在我们关注的时间戳周围的详细信息 — 我们应该查看错误发生时的时间戳附近的内容。可能会发现有在预防事务 ID 回卷的模式下运行着的 `autovacuum`、某些 DDL (如果通过 `log_statement='ddl'` 记录了 DDL)、或某个已达到 `log_min_duration_statement` 的长时间运行的查询 (建议将其设置为 1 秒或更低的值，以控制日志量，避免"观察者效应")。但在许多情况下，日志中没有其他信息，我们需要另一个解决方案。

在这种情况下，我们可以这样做：

1. 为执行 DDL 的会话设置自定义的 `application_name` 值，以便更容易在 `pg_stat_activity` 中进行区分 — 可以在 `PGAPPNAME `中设置，或直接通过 `SET` 设置：

   ```sql
   set application_name = 'ddl_runner';
   ```

2. 在另一个会话中执行观察查询。例如，使用以下包含 PL/pgSQL 代码的匿名 `DO` 块：

   ```sql
   do $$
   declare
     i int;
     rec record;
     wait_more boolean := false;
   begin
     for i in select generate_series(1, 2000) loop
       for rec in
         select
           pid,
           left(query, 50) as query_left50,
           *
         from pg_stat_activity
         where
           application_name = 'ddl_runner'
           and wait_event_type = 'Lock'
       loop
         raise log 'DDL session blocked. Session info: pid=%, query_left50=%, wait=%:%. Blockers: %',
           rec.pid,
           rec.query_left50,
           rec.wait_event_type,
           rec.wait_event,
           (
             select
               array_agg(json_build_object(
                 'pid', pid,
                 'state', state,
                 'wait', (wait_event_type || ':' || wait_event),
                 'query_l50', left(query, 50)
               ))
             from pg_stat_activity
             where array[pid] <@ pg_blocking_pids(rec.pid)
           );
   
           wait_more := true;
       end loop;
   
       if wait_more then
         perform pg_sleep(1);
   
         wait_more := false;
       else
         perform pg_sleep(0.05);
       end if;
     end loop;
   end $$;
   ```

此观察代码将在 DDL 会话等待超过 50 毫秒时报告类似信息：

```bash
2023-12-06T19:37:35.746363053Z 2023-12-06 19:37:35.746 UTC [237] LOG:  DDL session blocked. Session info: pid=197, query_left50=alter table t1 add column new_c int8;, wait=Lock
:relation. Blockers: {"{"pid" : 211, "state" : "idle in transaction", "wait" : "Client:ClientRead", "query_l50" : "select from t1 limit 0;"}"}
```

这应该足以进行分析为何试图执行 DDL 却失败了。

注意事项：

- 最好将 PL/pgSQL 代码封装至一个函数中，以替代匿名块，这样日志消息的 `CONTEXT` 和 `STATEMENT` 部分不会占用太多空间。
- 此函数可以通过 `pg_cron` 调用，从而提供一个长期的观察工具。但需要注意的是，这需要一个完整的后台进程来运行，因此可能不适合长期运行 — 可以在当我们知道要执行 DDL 时，才不定期地唤醒。