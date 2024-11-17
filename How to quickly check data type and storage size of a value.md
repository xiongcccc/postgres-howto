# How to quickly check data type and storage size of a value

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

以下是无需查阅文档便可快速检查值的数据类型和大小的方法。

## 如何检查值的数据类型

使用 `pg_typeof(...)`：

```sql
nik=# select pg_typeof(1);
 pg_typeof
-----------
 integer
(1 row)

nik=# select pg_typeof(1::int8);
 pg_typeof
-----------
 bigint
(1 row)
```

## 如何检查值的存储大小

使用 `pg_column_size(...)` — 即使没有实际的列：

```sql
nik=# select pg_column_size(1);
 pg_column_size
----------------
              4
(1 row)

nik=# select pg_column_size(1::int8);
 pg_column_size
----------------
              8
(1 row)
```

👉 `int4` (即 `int` 或 `integer`) 占用 4 字节，而 `int8`  (`bigint`) 占用 8 字节。

对于 `VARLENA` 类型，比如 `varchar`、`text`、`json`、`jsonb`、数组，长度是可变的 (因此得名)，并且还有额外的头部，比如：

```sql
nik=# select pg_column_size('ok'::text);
 pg_column_size
----------------
              6
(1 row)
```

👉 一个 4 字节的 `VARLENA` 头 (用于行内存储，参见 [struct varlena header](https://github.com/postgres/postgres/blob/c161ab74f76af8e0f3c6b349438525ad9575683b/src/include/c.h#L661-L681)) 和 2 字节的数据。

Boolean 示例：

```sql
nik=# select pg_column_size(true), pg_column_size(false);
 pg_column_size | pg_column_size
----------------+----------------
              1 |              1
(1 row)
```

回想一下上一篇指南，[列拼图](https://postgres-howto.cn/#/./docs/84)，我们可以得出，不仅需要 1 字节来存储一个比特，如果我们创建一个表 (`c1 boolean`，`c2 int8`)，由于对齐填充，它会变成 8 个字节，达到了 64 位！因此，如果以 `text` 存储 'true'，也不会浪费存储：

```sql
nik=# select pg_column_size('true'::text);
 pg_column_size
----------------
              8
(1 row)
```

👉 也是 8 字节 (4 字节 `VARLENA` 头，4 字节实际值)。而对于 `false`，由于使用对齐填充 (如果下一列是 8 或 16 字节)，情况很快就会变得更糟：

```sql
nik=# select pg_column_size('false'::text);
 pg_column_size
----------------
              9
(1 row)
```

👉 这些 9 字节可能因对齐填充被补齐为 16 字节，因此不要使用 `text` 来存储 `true/false`。对于多个布尔"标志"值的存储优化，考虑使用一个整数，将布尔值"打包"其中，并使用位操作符 `<<`、`~`、`|`、`&` 来编码和解码这些值 (参考 [Bit String Functions and Operators](https://postgresql.org/docs/current/functions-bitstring.html)) 。这样的话，便可以在一个 `int8` 中"打包" 64 个布尔值。

更多示例：

```sql
nik=# select pg_column_size(now());
pg_column_size
---------------
             8
(1 row)

nik=# select pg_column_size(interval '1s');
 pg_column_size
----------------
             16
(1 row)
```

## 如何检查行的存储大小

还有几个例子 — 如何检查行的大小：

```sql
nik=# select pg_column_size(row(true, now()));
 pg_column_size
----------------
             40
(1 row)
```

👉 一个 24 字节的元组头 (23 字节加上 1 个零值填充)，然后是 1 字节的布尔值 (填充 7 个零)，接着是 8 字节的 `timestamptz` 值。总计：24 + 1+7 + 8 = 40。

```sql
nik=# select pg_column_size(row(1, 2));
 pg_column_size
----------------
           32
(1 row)
```

👉 一个 24 字节的元组头，然后是两个 4 字节的整数，无需填充，总计 24 + 4 + 4 = 32。

## 无需记忆函数名称

在 psql 中，无需记忆函数名称 — 可以使用 `\df+` 来搜索函数名称：

```sql
nik=# \df *pg_*type*
                                           List of functions
   Schema   |                   Name                    | Result data type | Argument data types | Type
------------+-------------------------------------------+------------------+---------------------+------
 pg_catalog | binary_upgrade_set_next_array_pg_type_oid | void             | oid                 | func
 pg_catalog | binary_upgrade_set_next_pg_type_oid       | void             | oid                 | func
 pg_catalog | binary_upgrade_set_next_toast_pg_type_oid | void             | oid                 | func
 pg_catalog | pg_stat_get_backend_wait_event_type       | text             | integer             | func
 pg_catalog | pg_type_is_visible                        | boolean          | oid                 | func
 pg_catalog | pg_typeof                                 | regtype          | "any"               | func
(6 rows)
```

## 我见

作为最佳实践，不建议使用 `text` 类型存储布尔值。文中提到的将多个布尔值"打包"的操作，在追求极致优化的场景下可以尝试。