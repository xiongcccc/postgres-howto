# How to use variables in psql scripts

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

`psql` 是 PostgreSQL 的本地终端客户端，非常强大，适用于多种平台，通常与 Postgres 一起安装 (例如，在 Ubuntu/Debian 上可以通过 `apt install postgresql-client-16` 安装)。

`psql` 支持高级脚本编写，`psql` 脚本可以看作是 Postgres SQL 方言的扩展。例如，它支持诸如 `\set`、`\if`、`\watch` 等命令。通常我会使用 `.psql` 扩展名来保存需要通过 `psql` 执行的脚本。

在 `psql` 中可以使用两种类型的变量：

1. 客户端变量 (`psql` 的变量) — 使用 `\set` 设置，并通过冒号前缀的名称访问。
2. 服务端变量 (用户自定义的 GUC) — 使用 SQL 查询中的 `SET` 和 `SHOW` 来设置和查看。

## 客户端变量

数字变量：

```sql
nik=# \set var1 1.23

nik=# select :var1 as result;
 result
--------
   1.23
(1 row)
```

注意，`\set` 是一个客户端命令，不需要使用分号结束。

字符串变量：

```sql
nik=# \set str1 'Hello, world'

nik=# select :'str1' as hi;
      hi
--------------
 Hello, world
(1 row)
```

注意这里使用了奇怪的语法 `:'str1'`，需要一些时间来记住。

另一种设置客户端变量的有趣方式是使用 `\gset` 代替结束分号：

```sql
nik=# select now() as ts, current_user as usr \gset

nik=# select :'ts', :'usr';
           ?column?            | ?column?
-------------------------------+----------
 2023-11-14 00:27:53.615579-08 | nik
(1 row)
```

## 服务端变量

最常见的设置用户自定义 GUC 的方法是 `SET`：

```sql
nik=# set myvars.v1 to 1.23;
SET

nik=# show myvars.v1;
 myvars.v1
-----------
 1.23
(1 row)
```

注意：

- 这些是 SQL 查询，需要使用分号结束，它们也可以在其他客户端中执行，不仅仅是 `psql`。
- 自定义 GUC 应该带有"命名空间" (例如 `set v1 = 1.23;` 不会起作用 — 未加前缀的参数会被视为标准 GUC，如 `shared_buffers`)。
- 使用字符串很简单 (`set myvars.v1 to 'hello';`)。

通过 `SET` 定义的值不会持久化 — 它们仅在当前会话中有效 (或者如果使用 `SET LOCAL`，仅在当前事务中有效)。要使它们持久化，可以使用以下方法：

1. 集群范围内持久化：

```sql
nik=# alter system set myvars.v1 to 2;
ALTER SYSTEM

nik=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

nik=# \c
You are now connected to database "nik" as user "nik".

nik=# show myvars.v1;
 myvars.v1
-----------
 2
(1 row)
```

注意，使用 `pg_reload_conf()` 重新加载配置并重新连接。

2. 数据库级别持久化：

```sql
nik=# alter database nik set myvars.v1 to 3;
ALTER DATABASE

nik=# \c
You are now connected to database "nik" as user "nik".

nik=# show myvars.v1;
 myvars.v1
-----------
 3
(1 row)
```

3. 用户级别持久化：

```sql
nik=# alter user nik set myvars.v1 to 4;
ALTER ROLE

nik=# \c
You are now connected to database "nik" as user "nik".

nik=# show myvars.v1;
 myvars.v1
-----------
 4
(1 row)
```

## 如何将服务端变量与 SQL 结合使用

虽然 `SET/SHOW` 语法很常见，但它们不便于与其他 SQL 查询比如 `SELECT` 结合使用。为了解决这个问题，可以使用 `set_config(...)` 和 `current_setting(...)` 方法 ([docs](https://postgresql.org/docs/current/functions-admin.html#FUNCTIONS-ADMIN-SET))。

使用 `set_config(...)` 代替 `SET`：

```sql
nik=# select set_config('myvars.v1', '5', false);
 set_config
------------
 5
(1 row)

nik=# show myvars.v1;
 myvars.v1
-----------
 5
(1 row)
```

注意，此处的值只能是文本，因此对于数字和其他数据类型，可能需要进行后续转换。

同样，使用 `current_setting(...)` 代替 `SHOW`：

```sql
nik=# select set_config('myvars.v1', '6', false);
 set_config
------------
 6
(1 row)

nik=# select current_setting('myvars.v1', true)::int;
 current_setting
-----------------
               6
(1 row)
```

## 在匿名 DO 块中使用变量

💡👉 灵感源自 **[passing parameters from command line to DO statement](https://postgres.cz/wiki/PostgreSQL_SQL_Tricks#Passing_parameters_from_command_line_to_DO_statement)**.

匿名 DO 块不支持客户端变量，因此我们需要先将它们传递到服务端：

```sql
nik=# \set loops 5

nik=# select set_config('myvars.loops', (:loops)::text, false);
 set_config
------------
 5
(1 row)

nik=# do $$
begin
  for i in 1..current_setting('myvars.loops', true)::int loop
    raise notice 'Iteration %', i;
  end loop;
end $$;
NOTICE:  Iteration 1
NOTICE:  Iteration 2
NOTICE:  Iteration 3
NOTICE:  Iteration 4
NOTICE:  Iteration 5
DO
```

## 将变量传递给 .psql 脚本

假设我们有一个名为 `largest_tables.psql` 的脚本：

```bash
❯ cat largest_tables.psql

  select
    relname,
    pg_total_relation_size(oid::regclass),
    pg_size_pretty(pg_total_relation_size(oid::regclass))
  from pg_class
  order by pg_total_relation_size(oid::regclass) desc
  limit :limit;
```

现在，我们可以通过动态设置客户端变量 `limit` 来调用它：

```sql
❯ psql -X -f largest_tables.psql -v limit=2
     relname      | pg_total_relation_size | pg_size_pretty
------------------+------------------------+----------------
 pgbench_accounts |              164732928 | 157 MB
 t13              |               36741120 | 35 MB
(2 rows)

❯ PGAPPNAME=mypsql psql -X \
  -f largest_tables.psql \
  -v limit=3 \
  --csv
relname,pg_total_relation_size,pg_size_pretty
pgbench_accounts,164732928,157 MB
t13,36741120,35 MB
tttttt,36741120,35 MB
```