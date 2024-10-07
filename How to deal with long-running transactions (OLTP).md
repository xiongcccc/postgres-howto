# How to deal with long-running transactions (OLTP)

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

## 为什么长时间运行的事务会成为问题

在 OLTP (例如移动和 Web 应用程序) 环境中，长时间运行的事务通常会带来两个主要问题：

1. **锁定问题**。锁一旦被获取，只有在事务结束时才会释放锁。这可能会阻止其他事务。而且，有时即使是最"弱"的锁 — `AccessShareLock`，如果持有太久，也可能成为一个大问题；[even a simple open transaction that has read from a table, can cause big troubles](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries#problem-demonstration).
2. **对 autovacuum 活动的负面影响**。如果我们有一个打开的事务，并且持有事务 ID — 例如 `xid1`，所有 `xmax > xid1` 的死元组 (换句话说，事务开始后变成死元组的那些元组 — 由 `XID > xid1` 的其他事务产生的元组) 在事务结束前无法被 `autovacuum` 删除。这可能导致膨胀和性能下降。

"长时间运行"是一个相对的概念，具体含义取决于具体情况。通常，在负载较重的系统中 — 例如，每秒约10^5 次 TPS (包括只读查询) 和每秒约 10^3 次 XID 消耗 TPS (写操作) — 我们将运行时间超过 30-60 秒的事务视为长时间运行的事务。在最坏的情况下，这相当于在 30-60 秒内积累了 30-60 k 个死元组 — 假设所有事务在这段时间内每个都产生 1 个死元组。当然，这只是一个非常粗略的假设，但这可以提供关于规模的想法，帮助我们定义"长时间运行事务"这一术语的含义。

## 如何预防长时间运行的事务

在某些情况下，我们可能希望在全局层面防止长时间运行的事务，以避免上述负面影响。在这种情况下，我们认为中断一个长时间运行的事务并向用户发送错误信息，要好过于让用户受到负面影响。

如何完全预防长时间运行的事务发生呢？简短的回答是：仅使用 Postgres 设置是无法实现的。

截至 PG16 / 2023，Postgres 并未提供限制事务持续时间的方法 (尽管已经有一个实现[transaction_timeout](https://commitfest.postgresql.org/45/4040/) 的补丁提出，如果你能帮助测试和改进它会更好)。

不过，有两个设置可以帮助减少长时间运行事务的几率，但无法完全消除风险：

- [statement_timeout](https://postgresqlco.nf/doc/en/param/statement_timeout/) — 限制单个查询的最大持续时间。对于 Web /移动应用程序，将其设置为较低值，例如 30 秒或 15 秒。

  你可以在 Postgres 文档中找到，这是"不推荐的"，但是该建议并不实用并且我认为它没有效率。为了保护网络和移动应用程序，我们确实需要全局限制 statement_timeout：应用程序代码通常无论如何都是受限制的，当应用程序达到 30 秒这样的超时时间时，情况就不妙了，但 Postgres 仍在处理孤立查询。而且用户通常不会等待超过几秒钟 (参照: [What is a slow query?](https://postgres.ai/blog/20210909-what-is-a-slow-sql-query))。那些确实需要更高甚至无限制 statement_timeout 值的连接，可以在会话中使用简单的 `SET` 来设置 (例如，生成一些报告、执行 `pg_dump` 或创建索引的连接)。全局设置低值并在需要时覆盖是一种更安全的方法。

- [idle_in_transaction_session_timeout](https://postgresqlco.nf/doc/en/param/idle_in_transaction_session_timeout/) — 设置在事务中允许的最大空闲时间。类似建议：将其设置为较低的值，15-30 秒。对于确实需要长时间运行的事务，可以覆盖全局值。

如果同时将这两个选项设置为较低值，依旧无法完全阻止长时间运行的事务。例如，如果将两者都设置为 30 秒，我们可能仍然会有一个事务运行数小时：

- begin;
- a query lasting <  30s
- brief delay (< 30s)
- another query lasting < 30s
- ...

在这种情况下，两个阈值都未达到，但事务可能会运行数小时甚至数天。

虽然还没有`transaction_timeout`这样的设置，但我们可以考虑其他替代方法来完全阻止长时间运行的事务：

1. 使用定时任务 (或 `pg_cron` 或 `pg_timetable`) 记录运行"终结者"查询，检测所有持续时间超过 N 秒的事务并终止它们。

   示例查询：

   ~~~sql
   select clock_timestamp(), pid, query, pg_terminate_backend(pid)
   from pg_stat_activity
   where clock_timestamp() - xact_start > interval '5 minute';
   ~~~

   需要提前考虑如何处理例外情况 — 例如，排除 `pg_dump` 或创建索引的会话。可以通过 `pg_stat_activity.application_name `来排除这些会话 (通过 `PGAPPNAME `设置 `application_name` 是一个很好的实践)。

2. 在应用端限制。根据使用的语言和库，实现这一点的难易程度不同。还需要处理那些确实需要长时间运行事务的会话。

## 如何分析长时间运行的事务

获取所有长时间运行事务的列表非常简单：

```sql
select clock_timestamp() - xact_start, *
from pg_stat_activity
where clock_timestamp() - xact_start > interval '1 minute'
order by clock_timestamp() - xact_start desc;
```

但对于上述情况 — `statement_timeout` 和 `idle_in_transaction_session_timeout` 设置很低，仍然有长时间运行的事务 — 我们通常希望对具有长时间运行事务的会话的状态进行采样，以了解它由哪些查询组成。否则，我们将缺乏数据来源 (查询通常很快，低于 `log_min_duration_statement` 的阈值，因此不会记录在日志中)。

在这种情况下，我们可以应用 #PostgresMarathon 中描述的方法：[Day 11: Ad-hoc monitoring]()，每秒对长事务 (> 1 分钟) 进行采样 (可能值得增加频率)：

```bash
while sleep 1; do
  psql -XAtc "
      copy (
        with samples as (
          select
            clock_timestamp(),
            clock_timestamp() - xact_start as xact_duration,
            *
          from pg_stat_activity
        )
        select *
        from samples
        where xact_duration > interval '1 minute'
        order by xact_duration desc
      ) to stdout delimiter ',' csv
    " 2>&1 \
  | tee -a long_tx_$(date +%Y%m%d).log.csv
done
```

# 我见

17 中一个好消息是正式支持了 transaction_timeout，用于限制一个事务的执行时间

>Specify the maximum transaction execution time in milliseconds. If a timeout occurs, the session is forcibly disconnected. The default value is 0, which means no timeout will occur.

~~~sql
postgres=> SET transaction_timeout = '10s' ; 
SET
postgres=> BEGIN ; 
BEGIN
postgres=*> SELECT pg_sleep(10) ;
FATAL: terminating connection due to transaction timeout
server closed the connection unexpectedly
 This probably means the server terminated abnormally
 before or while processing the request.
The connection to the server was lost. Attempting reset: Succeeded.
~~~

读者阿沐分享了 transaction_timeout 内核实现，具体可以参照"PostgreSQL学徒"公众号。

