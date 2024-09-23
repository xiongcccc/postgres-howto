# How to decide when a query is too slow and needs optimization

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

"慢"是一个相对概念。在某些情况下，我们可能对查询延迟为 1 分钟并不会感到不满意 (或者不是？)，而在其他场景下，即使是 1 毫秒可能也显得过于缓慢。

何时应用优化技术的决策对效率来说至关重要 — 正如 Donald Knuth 在《计算机程序设计艺术》中所说的那样：

> 真正的问题在于，程序员花费了太多时间在错误的地方和错误的时间担心效率；在编程中，过早的优化是万恶之源 (或至少是大多数问题的根源)。

在下文中，我们假设我们正在处理 OLTP 或混合型工作负载，需要决定某个查询是否太慢并需要优化。

## 如何判断查询太慢

1. 你的用例是 OLTP 型的还是分析型的，又或者是混合型的？对于 OLTP 用例，要求更为严格，受到人类感知的影响 (参照:  [What is a slow SQL query?](https://postgres.ai/blog/20210909-what-is-a-slow-sql-query))， 而对于分析需求，我们通常可以等待一两分钟 — 除非它也是面向用户的。如果是这样的话，可能我们会认为 1 分钟太慢了。在这种情况下，请考虑使用列存储数据库系统 (Postgres 生态系统中有一个新的产品 [@hydradatabase](https://twitter.com/hydradatabase))。对于 OLTP，大多数面向用户的查询应当低于 100 毫秒 — 理想情况下低于 10 毫秒 — 这样用户向后端发出的完整请求不会超过 100-200 毫秒 (每个请求可能会发出多个 SQL 查询，具体情况而定)。当然，非面向用户的查询，例如来自后台任务、`pg_dump` 等，可以持续更长时间 — 前提是满足以下原则。
2. 在 OLTP 的情况下，第二个问题应当是：该查询是"只读"的吗，还是更改了数据 (无论是 DDL 还是简单的写入 DML — INSERT/UPDATE/DELETE)？在这种情况下，OLTP 中不应允许它运行超过一两秒，除非我们 100% 确定该查询不会长时间阻塞其他查询。对于大量写入操作，考虑将它们分批处理，以确保每个批次的持续时间不超过 1-2 秒。对于 DDL，务必小心锁的获取和锁链（阅读这些文章 [Common DB schema change mistakes](https://postgres.ai/blog/20220525-common-db-schema-change-mistakes#case-5-acquire-an-exclusive-lock--wait-in-transaction) 和 [Useful queries to analyze PostgreSQL lock trees (a.k.a. lock queues)](https://postgres.ai/blog/20211018-postgresql-lock-trees)).
3. 如果你处理的是只读查询，确保它不会运行太久 — 长时间运行的事务会导致 Postgres 长时间保留旧的死元组 ("xmin视界"未能前进)，从而导致 autovacuum 无法删除在事务开始后成为死元组的条目。避免事务持续时间超过一两小时 (如果你确实需要这么长的事务，建议在活动较少的时间段运行，当 XID 推进较慢时进行，并且尽量不要频繁地运行它们)。
4. 最后，即使查询相对较快 — 例如 10 毫秒——如果其频率较高，仍可能被视为过慢。例如，10 毫秒的查询每秒运行 1,000 次 (可以通过 `pg_stat_statements.calls` 查看)，则 Postgres 每秒需要花费 10 秒来处理这组查询。 在这种情况下，如果降低频率较难，则应将该查询视为慢查询，并尝试进行优化，以减少资源消耗 (目标是减少 `pg_stat_statements.total_exec_time `— 详见之前的帖子)。

## 总结

1. 所有超过 100-200 毫秒的查询，如果面向用户，应被视为慢查询。好的查询应该低于 10 毫秒。
2. 后台处理查询可以持续更长时间。如果它们修改了数据并可能阻塞面向用户的查询，则不应超过 1-2 秒。
3. DDL 应谨慎操作 — 确保它们不会导致大量写入 (如果会，则应分批处理)，并使用较低的 `lock_timeout` 和重试机制，以避免形成阻塞链。
4. 不要允许长时间运行的事务。确保 xmin 视界在推进，autovacuum 可以及时删除死元组 — 避免事务持续时间过长 (>1-2 小时)。
5. 即使是快的查询 (<100 毫秒)，如果 pg_stat_statements.calls 和 pg_stat_statements.total_exec_time 较高，也需要进行优化。