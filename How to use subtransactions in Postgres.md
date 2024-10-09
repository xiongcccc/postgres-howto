# How to use subtransactions in Postgres

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

## TL;DR

如无必要，否则不要使用子事务。

## 什么是子事务？

子事务，也称为"嵌套事务"，是指在已启动的事务范围内通过指令启动的事务 (来源：[Wikipedia](https://en.wikipedia.org/wiki/Nested_transaction))。这一特性允许用户部分回滚事务，这在许多情况下很有帮助：如果发生错误，重新执行的步骤会减少。

SQL 标准定义了描述这种机制的两个基本指令：`SAVEPOINT` 和扩展的 `ROLLBACK` 语句 — `ROLLBACK TO SAVEPOINT`。PostgreSQL 实现了这些指令，但允许在语法上稍有不同，例如可以在 `RELEASE` 和 `ROLLBACK` 语句中省略 `SAVEPOINT` 关键字。

你可能已经在使用子事务，例如：

- 在 Django 中，使用嵌套的 ["atomic()" blocks](https://docs.djangoproject.com/en/5.0/topics/db/transactions/#savepoints)
- 隐式使用：在 PL/pgSQL 函数中使用 `BEGIN / EXCEPTION WHEN ... / END` 块。

## 如何使用 (如果你真的想用)

语法：

- `SAVEPOINT savepoint_name` ([SAVEPOINT](https://postgresql.org/docs/current/sql-savepoint.html))
- `ROLLBACK [ WORK | TRANSACTION ] TO [ SAVEPOINT ] savepoint_name` ([ROLLBACK TO](https://postgresql.org/docs/current/sql-rollback-to.html))
- `RELEASE [ SAVEPOINT ] savepoint_name` ([RELEASE SAVEPOINT](https://postgresql.org/docs/current/sql-release-savepoint.html))

一个示例：

![Rolled-back subtransaction example](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0035_rolled_back_subtransaction_example.jpg)

## 推荐

对于任何计划增长的 OLTP 类型负载 (如 Web 和移动应用)，我唯一的实际建议是：

> 尽可能避免子事务

如果你不想有一天发生以下情况：

![Performance drop for more than 64 subtransactions in a transaction](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0035_performance_drop_too_many_subtx.jpg)

或者类似情况：

![Performance drop of standby server with primary running a long transaction and many subtransactions](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0035_standby_server_killed.jpg)

你可以在这篇文章中找到对子事务四大危险的详细分析：[PostgreSQL Subtransactions Considered Harmful](https://postgres.ai/blog/20210831-postgresql-subtransactions-considered-harmful)

截至 2023 年 (PG16 版本)，这些问题尚未解决，尽管一些优化工作正在进行中：

- [More scalable multixacts buffers and locking](https://commitfest.postgresql.org/45/2627/)
- [suboverflowed subtransactions concurrency performance optimize](https://postgresql.org/message-id/flat/003201d79d7b%24189141f0%2449b3c5d0%24@tju.edu.cn) (不幸的是，补丁已被撤回)

底线是

- 如果可以，不要使用子事务。
- 关注与子事务相关的 pgsql-hackers 邮件讨论，并参与测试和改进。
- 如果绝对需要使用子事务，请参考  [Problem 3: unexpected use of Multixact IDs](https://postgres.ai/blog/20210831-postgresql-subtransactions-considered-harmful#problem-3-unexpected-use-of-multixact-ids) 以及：
  - 仅在低 TPS 系统中使用它们
  - 避免深度嵌套
  - 在包含子事务的事务中谨慎使用 `SELECT ... FOR UPDATE`。
  - 监控 `pg_stat_slru` 数值 (PG13+，[Monitoring stats](https://postgresql.org/docs/current/monitoring-stats.html))，以便迅速发现并解决 SLRU 溢出问题。

# 我见

在 17 版本中，已经针对 SLRU 的问题进行了极大优化，并且可以配置 SLRU 相关参数 — multixact_member_buffers、multixact_offset_buffers 和 subtransaction_buffers。

SLRU optimization

- Make the cache size configurable
- Divide SLRU cache into small associative banks
- Implement bankwise lock to remove the centralized control lock contention
- Remove centralize LRU counter

Performance

- 2-3 x performance gain when there is huge load on SLRU