# How to get into trouble using some Postgres features

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

今天我们有一些相当有趣的素材，但了解 (并避免) 这些问题可以节省您的时间和精力。

## NULLs

`NULL` 值虽然很常见，但在 SQL 中是导致麻烦的主要原因，Postgres 也不例外。

例如，有人可能会忘记连接操作符 (||)、算术运算符 (*、/、+、-)、传统比较运算符 (=、<、>、<=、>=、<>) 都不是 NULL-safe 的操作，结果可能会丢失。

尤其是当你建立一家初创公司并且一些重要的业务逻辑依赖于它时，查询不能正确处理 NULL，导致用户群或金钱或时间 (或所有) 的损失：

```sql
nik=# \pset null ∅
Null display is "∅".

nik=# select null + 1;
	?column?
 ----------

         ∅

(1 row)
```

`NULL` 值真的很危险，即使是经验丰富的工程师在处理它们时也常常会遇到问题。以下是一些可以帮助你的素材：

- [NULLs: the good, the bad, the ugly, and the unknown](https://postgres.fm/episodes/nulls-the-good-the-bad-the-ugly-and-the-unknown) (podcast)
- [What is the deal with NULLs?](http://thoughts.davisjeff.com/2009/08/02/what-is-the-deal-with-nulls/)

一些技巧 — 如何使代码成为 NULL-safe 的：

- 考虑使用表达式如 `COALESCE(val, 0)` 将 `NULL` 值替换为某个值 (通常是 `0` 或 `''`)。
- 在比较时，使用 `IS [NOT] DISTINCT FROM` 代替 `=` 或 `<>`  (不过请查看 `EXPLAIN` 执行计划)。
- 进行连接时使用 `format('%s %s', var1, var2)`。
- 不要使用 `WHERE NOT IN (SELECT ...) `— 使用 `NOT EXISTS `代替 (参照 [JOOQ blog post](https://jooq.org/doc/latest/manual/reference/dont-do-this/dont-do-this-sql-not-in/))。
- 要小心，`NULL` 值是狡猾的。

## 在高负载下使用子事务

如果你希望达到数十万或数百万 TPS 并遇到各种问题，请使用子事务。你可能会隐式使用到它们 — 例如，如果使用了 Django、Rails 或 PL/pgSQL 中的`BEGIN/EXCEPTION`块。

为什么要完全摆脱子事务的原因：[PostgreSQL Subtransactions Considered Harmful](https://postgres.ai/blog/20210831-postgresql-subtransactions-considered-harmful)

## int4 主键

如果表有 10 亿行，要想将 `int4` 主键 (也称为 int、integer) 零停机时间转换为 `int8` 主键时，所需的工作量极其之大。由于对齐填充，表 (`id int4, created_at timestamptz`) 将占用与 (`id int8, created_at timestamptz`) 相同的磁盘空间。

## (特殊的) SELECT INTO并不像你想的那样

有一天我在调试 PL/pgSQL 函数时，复制粘贴了一条查询，像这样，运行在 psql 中：

```sql
nik=# select * into var from t1 limit 1;
SELECT 1
```

它工作了！这真是一个巨大的惊喜 — 在 SQL 上下文中，[SELECT INTO](https://postgresql.org/docs/current/sql-selectinto.html) 是一个 DDL 命令，它会创建一个表并将数据插入其中 (难道这不应该被弃用吗？)。

## 认为"事务性DDL"很简单

是的，Postgres 具有"事务性 DDL"，你可以从中获益良多 — 直到您无法这样做。在高负载下，你无法依赖它 — 而是需要开始使用零停机时间的方法，避免错误 (阅读：[常见的数据库模式更改错误](https://postgres.ai/blog/20220525-common-db-schema-change-mistakes)，并依赖于"非事务性" DDL，比如 `CREATE INDEX CONCURRENTLY`，某些操作可能会失败，之后需要清理再重试)。

在高负载下进行 DDL 部署的大问题是，默认情况下，你可能会在尝试部署一个非常轻的模式更改时遇到停机 — 除非实现了带有低 `lock_timeout` 和重试的逻辑 (参照 [Zero-downtime Postgres schema migrations need this: lock timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries))。

## 一次删除大量行

这是一个陷入麻烦的好方法：发出 `DELETE` 数百万行的命令并等待。如果检查点没有调优 (`max_wal_size = 1GB`)，如果元组通过 IndexScan 被删除 (意味着使页面变脏的过程相当"随机")，并且磁盘 IO 受到限制，这可能会让你的系统崩溃。即使它幸存下来，你也会碰到：

- 锁定问题的风险 (`DELETE` 阻止其他用户发出的写入)
- 生成了大量死元组，这些死元组稍后会被 `autovacuum` 转换为膨胀。

解决方法：

- 拆分为批次，
- 如果大量写入不可避免，请考虑临时提高 `max_wal_size`，这不需要重启 (不过：如果服务器在此过程中崩溃，可能会增加恢复时间)。

阅读 [common db schema change mistakes](https://postgres.ai/blog/20220525-common-db-schema-change-mistakes#case-4-unlimited-massive-change).

## 其他"不要做"文章

- [Depesz: Don’t do these things in PostgreSQL](https://depesz.com/2020/01/28/dont-do-these-things-in-postgresql/)
- [PostgreSQL Wiki: Don't Do This](https://wiki.postgresql.org/wiki/Don't_Do_This)
- [JOOQ: Don't do this](https://jooq.org/doc/latest/manual/reference/dont-do-this/)