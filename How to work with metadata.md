# **How to work with **metadata

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

当处理元数据 (关于数据的数据) 时，Postgres 中的这些参考文档非常有用：

- [System Catalogs](https://postgresql.org/docs/current/catalogs.html)
- [System Views](https://postgresql.org/docs/current/views.html)
- [The Cumulative Statistics System](https://postgresql.org/docs/current/monitoring-stats.html)
- [The Information Schema](https://postgresql.org/docs/current/information-schema.html)

没有必要重复文档中的内容。相反，我们专注于一些技巧和原则，使你的工作更高效。我们将讨论以下主题：

- `::oid` 和 `::regclass`
- `\?` 和 `ECHO_HIDDEN`
- 性能
- `INFORMATION_SCHEMA`
- `pg_stat_activity` 并不是一张表

------

## ::oid,::regclass

在 Postgres 的术语中，表、索引、视图、物化视图均被称为"关系"。它们的元数据信息可以通过多种方式查看，但"核心"位置是 [pg_class system catalog](https://postgresql.org/docs/current/catalog-pg-class.html)。换句话说，这是一个存储所有表、索引等信息的表。它有两个主键：

- PK：oid — 一个数字 ([OID, object identifier](https://postgresql.org/docs/current/datatype-oid.html))
- UK：一对列 (relname, relnamespace)，即关系名和模式 OID。

一个值得记住的技巧是：OID 可以快速转化为关系名，反之亦然，使用类型转换为 oid 和 regclass 数据类型。

以下是名为 `t1` 的表的简单示例：

```sql
nik=# select 't1'::regclass;
 regclass
----------
 t1
(1 row)

nik=# select 't1'::regclass::oid;
  oid
-------
 74298
(1 row)

nik=# select 74298::regclass;
 regclass
----------
 t1
(1 row)
```

因此，不需要执行 `select oid from pg_class where relname = ...` — 只需记住 `::regclass` 和 `::oid` 即可。

## ? 和 ECHO_HIDDEN

`psql` 的 `\?` 命令非常重要 — 通过它可以找到所有命令的相关描述。例如：

```sql
\d[S+]                 list tables, views, and sequences
```

这些"描述"命令会隐式生成一些SQL语句 — "窥探"它们可能很有帮助。为此，我们首先需要开启 `ECHO_HIDDEN`：

```sql
nik=# \set ECHO_HIDDEN on
```

或者在启动 `psql` 时使用 `-E` 选项。然后我们便可以开始窥探：

```sql
nik=# \d t1
/********* QUERY **********/
SELECT c.oid,
 n.nspname,
 c.relname
FROM pg_catalog.pg_class c
    LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
WHERE c.relname OPERATOR(pg_catalog.~) '^(t1)$' COLLATE pg_catalog.default
 AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 2, 3;
/**************************/

[... + more queries to get info about "t1" ...]
```

查看这些查询可以帮助你构建各种处理元数据的工具。

## 性能

在某些情况下，元数据查询可能比较重且慢。如果出现这种情况，请考虑以下方法：

- 考虑缓存以减少元数据查询的频率和必要性。
- 检查系统目录膨胀情况。例如，由于频繁的 DDL 操作、临时表的使用等原因，`pg_class` 可能会膨胀。在这种情况下，不幸的是，可能需要执行 `VACUUM FULL` (`pg_repack` 不能处理系统目录)。如果需要，不要忘记 Postgres 中零停机 DDL 的黄金法则 — [use low lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries)

## INFORMATION_SCHEMA

系统目录和视图是查询表和索引元数据的"本地"方式 — 但这并不是标准的方式。标准方法是 `INFORMATION_SCHEMA`，Postgres 根据 SQL 标准支持它：[Docs](https://postgresql.org/docs/current/information-schema.html)。如何使用：

- 对于简单的、跨数据库兼容的元数据查询，使用 `information_schema`。
- 对于更复杂的、特定于 Postgres 的查询，或者当需要详细的内部信息时，使用本地系统目录。

## pg_stat_activity 不是表

必须记住的是，在查询元数据时，你可能正在处理一些看起来像但行为不像普通表的东西。

例如，当你从 `pg_stat_activity` 中读取记录时，你并不是在处理一致的表数据快照：读取第一行和理论上的最后一行是在不同时刻产生的，你可能会看到并非同时运行的查询。

这种现象也解释了为什么 `select now() - query_start from pg_stat_activity;` 可能会给出负值：函数 `now()` 在事务开始时执行一次，并且在事务内部不会改变其值，无论调用它多少次。

要获取精确的时间间隔，请使用 `clock_timestamp()` 代替：

```sql
select clock_timestamp() - query_start from pg_stat_activity;
```