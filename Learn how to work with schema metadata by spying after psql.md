# Learn how to work with schema metadata by spying after psql

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

## 不确定时，从何开始

[psql 文档](https://postgresql.org/docs/current/app-psql.html)

`psql` 有内置帮助文档 — 命令`\?` ；在使用 psql 时值得记住并将其作为起点。

当你不确定某个函数的确切名称时，可以使用搜索命令 `\df *关键字*`。例如：

```sql
nik=# \df *clock*
                                  List of functions
   Schema   |      Name       |     Result data type     | Argument data types | Type
------------+-----------------+--------------------------+---------------------+------
 pg_catalog | clock_timestamp | timestamp with time zone |                     | func
(1 row)
```

## 如何查看 psql 正在做什么 – ECHO_HIDDEN

假设我们想查看表 `t1` 的大小，为此，我们可以构建一个返回表大小的查询 (或者只是在某处找到其大小或询问 LLM)。但是当使用 `psql` 时，我们只需使用 `\dt+ t1`：

```sql
nik=# \dt+ t1
                                  List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+------+-------+----------+-------------+---------------+-------+-------------
 public | t1   | table | postgres | permanent   | heap          | 25 MB |
(1 row)
```

我们可能想要循环执行，以观察表大小是如何增长的。为此，psql 支持 `\watch` — 但是，它不适用于其他反斜杠命令。

解决方案 —  打开 `ECHO_HIDDEN` 并查看 `\dt+` 后面的 SQL 查询 (或者，你可以在启动 `psql` 时使用选项 `--echo-hidden`)：

```sql
nik=# \set ECHO_HIDDEN 1
nik=#
nik=# \dt+ t1
********* QUERY **********
SELECT n.nspname as "Schema",
  c.relname as "Name",
  CASE c.relkind WHEN 'r' THEN 'table' WHEN 'v' THEN 'view' WHEN 'm' THEN 'materialized view' WHEN 'i' THEN 'index' WHEN 'S' THEN 'sequence' WHEN 't' THEN 'TOAST table' WHEN 'f' THEN 'foreign table' WHEN 'p' THEN 'partitioned table' WHEN 'I' THEN 'partitioned index' END as "Type",
  pg_catalog.pg_get_userbyid(c.relowner) as "Owner",
  CASE c.relpersistence WHEN 'p' THEN 'permanent' WHEN 't' THEN 'temporary' WHEN 'u' THEN 'unlogged' END as "Persistence",
  am.amname as "Access method",
  pg_catalog.pg_size_pretty(pg_catalog.pg_table_size(c.oid)) as "Size",
  pg_catalog.obj_description(c.oid, 'pg_class') as "Description"
FROM pg_catalog.pg_class c
     LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
     LEFT JOIN pg_catalog.pg_am am ON am.oid = c.relam
WHERE c.relkind IN ('r','p','t','s','')
  AND c.relname OPERATOR(pg_catalog.~) '^(t1)$' COLLATE pg_catalog.default
  AND pg_catalog.pg_table_is_visible(c.oid)
ORDER BY 1,2;
**************************

                                  List of relations
 Schema | Name | Type  |  Owner   | Persistence | Access method | Size  | Description
--------+------+-------+----------+-------------+---------------+-------+-------------
 public | t1   | table | postgres | permanent   | heap          | 72 MB |
(1 row)
```

现在我们得到了 SQL 查询，便可以使用 `\watch 5` 每隔 5 秒查看一次表大小 (并且可以省略不需要的字段)：

```sql
nik=# SELECT n.nspname as "Schema",
    c.relname as "Name",
    pg_catalog.pg_size_pretty(pg_catalog.pg_table_size(c.oid)) as "Size"
  FROM pg_catalog.pg_class c
       LEFT JOIN pg_catalog.pg_namespace n ON n.oid = c.relnamespace
       LEFT JOIN pg_catalog.pg_am am ON am.oid = c.relam
  WHERE c.relkind IN ('r','p','t','s','')
    AND c.relname OPERATOR(pg_catalog.~) '^(t1)$' COLLATE pg_catalog.default
    AND pg_catalog.pg_table_is_visible(c.oid)
  ORDER BY 1,2 \watch 5
Thu 16 Nov 2023 08:00:10 AM UTC (every 5s)

 Schema | Name | Size
--------+------+-------
 public | t1   | 73 MB
(1 row)

Thu 16 Nov 2023 08:00:15 AM UTC (every 5s)

 Schema | Name | Size
--------+------+-------
 public | t1   | 75 MB
(1 row)

Thu 16 Nov 2023 08:00:20 AM UTC (every 5s)

 Schema | Name | Size
--------+------+-------
 public | t1   | 77 MB
(1 row)
```