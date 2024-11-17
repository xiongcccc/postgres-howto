# How to use lib_pgquery in shell to normalize and match queries from various sources

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

## 为什么使用 lib_pgquery

在 [Day 12: How to find query examples for problematic pg_stat_statements records](https://postgres-howto.cn/#/./docs/12) 中提到，可以使用 [lib_pgquery](https://github.com/pganalyze/libpg_query) 标准化查询。这个库可以创建查询文本的树形表示，并计算所谓的"指纹" — 查询的标准化哈希 (移除所有参数后的查询文本)。

这对于以下几种情况特别有用：

- 如果你需要将 `pg_stat_statements` 中的标准化查询与来自 `pg_stat_activity` 或 Postgres 日志的单个查询文本相匹配，并且使用的是 Postgres 14 之前的版本，其中 [compute_query_id](https://postgresqlco.nf/doc/en/param/compute_query_id/) 功能尚未实现来解决此问题。
- 如果你使用的是新版本，但 `compute_query_id` 是 `off`。
- 如果你使用来自不同来源的查询文本，并且不确定标准的 `query_id` (也称为 "queryid" — 这个命名在不同的表中并不统一) 是否可以作为可靠的匹配方式。

`lib_pgquery` 大多数是用 C 编写的，并且用于各种语言的库中：

- [Ruby](https://github.com/pganalyze/pg_query)
- [Python](https://pypi.org/project/pglast/)
- [Go](https://github.com/pganalyze/pg_query_go)
- [Node](https://github.com/pyramation/pgsql-parser)

## 用于 CLI 的 Docker 版本

为了方便使用，我的同事们将 Go 版本封装成了 Docker 镜像，允许通过 CLI 风格在 Shell 中使用：

```bash
❯ docker run --rm postgresai/pg-query-normalize \
    'select c1 from t1 where c2 = 123;'
{
  "query": "select c1 from t1 where c2 = 123;",
  "normalizedQuery": "select c1 from t1 where c2 = $1;",
  "fingerprint": "0212acd45326d8972d886d4b3669a90be9dd4a9853",
  "tree": [...]
  ]
}
```

该命令会返回一个 JSON 格式的结果。

为了提取标准化查询，可以使用 `jq`工 具：

```bash
❯ docker run --rm postgresai/pg-query-normalize \
      'select c1 from t1 where c2 = 123;' \
    | jq -r '.normalizedQuery'
select c1 from t1 where c2 = $1;
```

仅提取文本形式的指纹：

```bash
❯ docker run --rm postgresai/pg-query-normalize \
    'select c1 from t1 where c2 = 123;' \
  | jq -r '.fingerprint'
0212acd45326d8972d886d4b3669a90be9dd4a9853
```

如果我们使用不同的参数，指纹并不会改变 — 例如，将 123 替换为 0：

```bash
❯ docker run --rm postgresai/pg-query-normalize \
    'select c1 from t1 where c2 = 0;' \
  | jq -r '.fingerprint'
0212acd45326d8972d886d4b3669a90be9dd4a9853
```

## 标准化查询作为输入

这是 lib_pgquery 一个非常好的特性！如果查询已经标准化，并且我们有占位符 (如 `$1`、`$2` 等) 代替实际参数，那么查询的指纹将保持不变：

```bash
❯ docker run --rm postgresai/pg-query-normalize \
    'select c1 from t1 where c2 = $1;' \
  | jq -r '.fingerprint'
0212acd45326d8972d886d4b3669a90be9dd4a9853
```

但是，如果查询不同，比如 `SELECT` 子句发生了变化，例如使用 `select *` 代替 `select c1`，则会得到新的指纹值：

```bash
❯ docker run --rm postgresai/pg-query-normalize \
    'select * from t1 where c2 = $1;' \
  | jq -r '.fingerprint'
0293106c74feb862c398e267f188f071ffe85a30dd
```

## 处理 IN 列表

与内置的 `queryid` 不同，lib_pgquery 会忽略 `IN` 子句中参数的数量 — 这种行为更适合查询的标准化和匹配。比如，比较以下两条查询：

```bash
❯ docker run --rm postgresai/pg-query-normalize \
    'select * from t1 where c2 in (1, 2, 3);' \
  | jq -r '.fingerprint'
022fad3ad8fab1b289076f4cfd6ef0a21a15d01386

❯ docker run --rm postgresai/pg-query-normalize \
  'select * from t1 where c2 in (1000);' \
| jq -r '.fingerprint'
022fad3ad8fab1b289076f4cfd6ef0a21a15d01386
```

## 我见

在 Oracle 中，有 sql_id，在 14 版本以后，PostgreSQL 也正式支持了 query_id。

> 1. 查看等待事件，根据 sql_id 反查会话期间执行的 sql_text；
> 2. 或者根据 sql_id 查看某段时间内，对应 SQL 的执行计划是否发生变化、何时发生变化，然后通过对执行计划的好坏分析进行 SQL 执行计划的固化

另外有个值得注意的细节：**PostgreSQL does not generate query_id in the pg_stat_activity until the SQL is parsed.** 

~~~sql
/* sesion 1 :: Take an exclusive lock on the table */

postgres=> BEGIN;
BEGIN
postgres=*> alter table test add column n2 int;
ALTER TABLE

/* sesion 2 :: Query is blocked due to exclusive lock taken by session 1 */

postgres=> select * from test;

<<waiting on session1>>

/* session 3 :: query_id column is blank */

select query_id,query from pg_stat_activity where query like '%test%';
       query_id       |                                    query
----------------------+-----------------------------------------------------
                      | select * from test;
~~~

推荐阅读：

- [PostgreSQL中的SQL_ID介绍及使用 ](https://mp.weixin.qq.com/s/pu6b4wS7yv6XOnJwzP3ymA)
- [Peek into Query Hash (query_id) in PostgreSQL](https://virender-cse.medium.com/query-hash-in-postgresql-4522e91b5623)

