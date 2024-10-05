# How to set application_name without extra queries

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

`application_name` 非常有用，可以控制你和他人在 `pg_stat_activity` 中看到的内容 (该视图中有一个同名的列)，以及使用此系统视图的各种工具。此外，当 `log_line_prefix` 包含 `%a` 时，它还会出现在 Postgres 的日志中。

文档：[application_name](https://postgresql.org/docs/current/runtime-config-logging.html#GUC-APPLICATION-NAME)

设置 `application_name` 是一个很好的实践 — 例如，在事件发生期间或之后进行根本原因分析时非常有帮助。

以下方法也可以用于设置其他设置 (包括常规的 Postgres 参数，如 `statement_timeout` 或 `work_mem`)，但此处我们重点关注 `application_name`。

通常，`application_name` 是通过 `SET` 设置的 (抱歉使用了同源词)：

```sql
nik=# show application_name;
 application_name
------------------
 psql
(1 row)

nik=# set application_name = 'human_here';
SET

nik=# select application_name, pid from pg_stat_activity where pid = pg_backend_pid() \gx
-[ RECORD 1 ]----+-----------
application_name | human_here
pid              | 93285
```

然而，即使是一个非常快的查询，也意味着额外的RTT ([round-trip time](https://en.wikipedia.org/wiki/Round-trip_delay))，影响延迟，尤其是在与远程服务器进行通信时。

为了避免这种情况，可以使用 `libpq` 的选项。

## 方法1：通过环境变量

```bash
❯ PGAPPNAME=myapp1 psql \
  -Xc "show application_name"
 application_name
------------------
 myapp1
(1 row)
```

(`-X` 表示忽略 `.psqlrc`，这是在自动化脚本中使用 `psql` 时的良好实践。)

## 方法2：通过连接URI

```bash
❯ psql \
  "postgresql://?application_name=myapp2" \
  -Xc "show application_name"
 application_name
------------------
 myapp2
(1 row)
```

URI 方法优先于 `PGAPPNAME`。

## 在应用程序代码中

所描述的方法不仅可以与 psql 一起使用。以 Node.js 为例：

```bash
❯ node -e "
 const { Client } = require('pg');

 const client = new Client({
   connectionString: 'postgresql://?application_name=mynodeapp'
 });

 client.connect()
       .then(() => client.query('show application_name'))
       .then(res => {
           console.log(res.rows[0].application_name);
           client.end();
       });
 "
 mynodeapp
```