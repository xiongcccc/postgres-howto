# How to work with arrays, part 1

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

今天我们来聊一聊数组：[Docs](https://postgresql.org/docs/current/arrays.html)。

Postgres 中的数组非常强大，它们在 JSON 加入之前就已经存在。了解如何使用数组可以让你实现更强大的表达能力 (不仅仅是操作 — 你还可以存储数组，表的列也可以是数组类型，甚至是多维数组)。

如果你只记得关于 Postgres 数组的一点，那就是：Postgres 与其他系统不同，数组的索引是从 1 开始的，而不是 0。而且，当试图使用索引 0 时也不会发出警告：

```sql
nik=# \pset null '(null)'
Null display is "(null)".

nik=# select (array['one', 'two'])[0];
 array
--------
 (null)
(1 row)

nik=# select (array['one', 'two'])[1];
 array
-------
 one
(1 row)
```

## 创建数组

### 方法1：将数组值定义为字符串并转换为数组类型

```sql
nik=# select '{"one", "two"}'::text[];
   text
-----------
 {one,two}
(1 row)
```

### 方法2：使用 `array[...]`

```sql
nik=# select array['one', 'two'];
   array
-----------
 {one,two}
(1 row)
```

### 方法3：使用 `select array( <subquery> );`

```sql
nik=# select array( values('one'), ('two') );
   array
-----------
 {one,two}
(1 row)
```

这种方法可以将多个单列行打包成一个数组值：

```sql
nik=# select array(select pid from pg_stat_activity);
                    array
---------------------------------------------
 {54757,54759,26135,54751,54758,54750,54756}
(1 row)
```

有趣的是，你可以将整个表打包成一个单一值 — 更准确地说，是一个单列单行的表：

```sql
nik=# select n from pg_namespace as n \gx
-[ RECORD 1 ]-------------------------------------------------------------------------
n | (99,pg_toast,10,)
-[ RECORD 2 ]-------------------------------------------------------------------------
n | (11,pg_catalog,10,"{nik=UC/nik,=U/nik}")
-[ RECORD 3 ]-------------------------------------------------------------------------
n | (2200,public,6171,"{pg_database_owner=UC/pg_database_owner,=U/pg_database_owner}")
-[ RECORD 4 ]-------------------------------------------------------------------------
n | (13471,information_schema,10,"{nik=UC/nik,=U/nik}")
-[ RECORD 5 ]-------------------------------------------------------------------------
n | (41022,pgss_pgsa,10,)

nik=# select array(select n from pg_namespace as n) \gx
-[ RECORD 1 ]------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
array | {"(99,pg_toast,10,)","(11,pg_catalog,10,\"{nik=UC/nik,=U/nik}\")","(2200,public,6171,\"{pg_database_owner=UC/pg_database_owner,=U/pg_database_owner}\")","(13471,information_schema,10,\"{nik=UC/nik,=U/nik}\")","(41022,pgss_pgsa,10,)"}
```

这个值保留了列名，你可以使用 `unnest(..)` 和 `(t).*` 将它还原回来：

```sql
nik=# with packed(val) as (
  select array(select n from pg_namespace as n)
)
select unnest(val)
from packed;
                                       unnest
------------------------------------------------------------------------------------
 (99,pg_toast,10,)
 (11,pg_catalog,10,"{nik=UC/nik,=U/nik}")
 (2200,public,6171,"{pg_database_owner=UC/pg_database_owner,=U/pg_database_owner}")
 (13471,information_schema,10,"{nik=UC/nik,=U/nik}")
 (41022,pgss_pgsa,10,)
(5 rows)

nik=# with packed(val) as (
  select array(select n from pg_namespace as n)
)
select (unnest(val)).*
from packed;
  oid  |      nspname       | nspowner |                            nspacl
-------+--------------------+----------+---------------------------------------------------------------
    99 | pg_toast           |       10 | (null)
    11 | pg_catalog         |       10 | {nik=UC/nik,=U/nik}
  2200 | public             |     6171 | {pg_database_owner=UC/pg_database_owner,=U/pg_database_owner}
 13471 | information_schema |       10 | {nik=UC/nik,=U/nik}
 41022 | pgss_pgsa          |       10 | (null)
(5 rows)
```

简直是魔法！这可以帮助你编写非常复杂的 CTE (建议对每一步都进行注释 — Postgres 支持 SQL 标准的 `--` 注释，以及`/* ... */`注释)。

### 方法4：聚合函数

```sql
nik=# select array_agg(i) from generate_series(1, 3) as i;
 array_agg
-----------
 {1,2,3}
(1 row)
```

这种方法还可以将所有单列行打包成一个数组值：

```sql
nik=# select array_agg(pid) from pg_stat_activity;
                  array_agg
---------------------------------------------
 {54757,54759,26135,54751,54758,54750,54756}
(1 row)
```

## 如何访问数组元素

访问数组成员非常简单：

```sql
nik=# select (array_agg(pid))[1] from pg_stat_activity;
 array_agg
-----------
     54757
(1 row)
```

记住，索引是从 1 开始的，访问不存在的成员并不会报错，而是返回 NULL：

```sql
nik=# select (array_agg(pid))[100000] from pg_stat_activity;
 array_agg
-----------
    (null)
(1 row)
```

你可以使用 `arr[N:M]` 获取数组的一个"切片"。你还可以省略一个数字 — 例如，只保留前 3 个成员，使用 `arr[:3]`：

```sql
nik=# select (array_agg(pid))[:3] from pg_stat_activity;
 array_agg
---------------------
 {54757,54759,26135}
(1 row)
```

// 未完待续