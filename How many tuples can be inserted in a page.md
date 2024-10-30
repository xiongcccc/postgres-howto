# How many tuples can be inserted in a page

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

在 Postgres 中，所有表都有隐藏的系统列，`ctid` 便是其中之一。通过读取 `ctid`，我们可以看到元组 (元组 = 行的物理版本) 的物理位置，包括页号以及页内偏移量：

```sql
nik=# create table t0 as select 1 as id;
SELECT 1

nik=# select ctid, id from t0;
 ctid  | id
-------+----
 (0,1) |  1
(1 row)
```

👉 第 0 页，1 位置处。

单个 PostgreSQL 页面默认为 8 KiB，可以通过查看 `block_size` 来确认：

```sql
nik=# show block_size;
 block_size
------------
 8192
(1 row)
```

一个页面中可以容纳多少个元组？来看一下：

```sql
nik=# create table t0 as select i
from generate_series(1, 1000) as i;
SELECT 1000

nik=# select count(*)
from t0
where (ctid::text::point)[0] = 0;
 count
-------
   226
(1 row)

nik=# select pg_column_size(i) from t0 limit 1;
 pg_column_size
----------------
              4
(1 row)
```

👉 如果我们使用 4 字节的数字，那么可以容纳 226 条元组。此处我使用 `(ctid::text::point)[0]` 将 `ctid` 的值转换为"point"来获取第一个组成部分，即页号。

即使使用 2 字节的数字或 1 字节的布尔值 (注意，布尔值需要 1 字节，而不是 1 比特)，这个数量也是相同的：

```sql
nik=# drop table t0;
DROP TABLE

nik=# create table t0 as select true
from generate_series(1, 1000) as i;
SELECT 1000

nik=# select count(*)
  from t0
  where (ctid::text::point)[0] = 0;
 count
-------
   226
(1 row)
```

为什么还是 226？事实上，值的大小在这里无关紧要，只要小于或等于 8 字节即可。对于每一行，对齐填充都会添加"零"，因此每行始终有 8 个字节：
$$
\frac{8192 - 24}{4 + 24 + 8} = 226
$$
👉 这里我们统计了以下内容：

- 24 字节的页头 (`PageHeaderData`)。
- 每个元组指针 — 每个 4 字节 (`ItemIdData`)。
- 每个元组头 — 每个 23 字节，填充到 24 字节 (`HeapTupleHeaderData`)。
- 每个元组值 — 如果 ≤ 8字节，则填充到 8 字节。

源码定义了这些结构 (for [PG16](https://github.com/postgres/postgres/blob/REL_16_STABLE/src/include/storage/bufpage.h))。

**我们能容纳更多元组吗？**

答案是可以的。Postgres 允许创建没有列的表！在这种情况下，计算如下：
$$
\frac{8192 - 24}{4 + 24} = 291
$$
让我们观察一下 (注意 `SELECT` 子句中的空列)：

```sql
nik=# create table t0 as select
from generate_series(1, 1000) as i;
SELECT 1000

nik=# select count(*)
from t0
where (ctid::text::point)[0] = 0;
 count
-------
   291
(1 row)
```

## 我见

以 0️⃣ 填充：

![image-20241030122538478](https://gitee.com/xiongcccc/internals_image/raw/master/docsfiy/image-20241030122538478.png)

>Here we only need 40 bytes per row excluding the variable sized data and 24-byte tuple header. 8 bytes being saved may not sound like much, but for tables as large as the events table it does begin to matter. For example, when storing 80 000 000 rows this translates to a space saving of at least 610 MB, all by just changing the order of a few columns.