# How to find the best order of columns to save on storage ("Column Tetris")

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

从存储角度来看，Postgres 中的列顺序重要吗？

答案是肯定的。我们来看一个例子 (注意：不建议使用 int4 作为主键，这里只是出于教学目的)。

```sql
create table t(
  id int4 primary key,
  created_at timestamptz,
  is_public boolean,
  modified_at timestamptz,
  verified boolean,
  published_at timestamptz,
  score int2
);

insert into t
select
  i,
  clock_timestamp(),
  true,
  clock_timestamp(),
  true,
  clock_timestamp(),
  0
from generate_series(1, 1000000) as i;

vacuum analyze t;
```

查看表大小：

```sql
nik=# \dt+ t
                   List of relations
 Schema | Name | Type  |  Owner   | Size  | Description
--------+------+-------+----------+-------+-------------
 public | t    | table | postgres | 81 MB |
(1 row)
```

现在，让我们使用来自 [postgres_dba](https://github.com/NikolayS/postgres_dba) 的报告 p1 (假设已安装)：

```sql
:dba

Type your choice and press <Enter>:
p1
 Table | Table Size | Comment |    Wasted *     |  Suggested Columns Reorder
-------+------------+---------+-----------------+-----------------------------
 t     | 81 MB      |         | ~23 MB (28.40%) | is_public, score, verified +
       |            |         |                 | id, created_at, modified_at+
       |            |         |                 | published_at
(1 row)
```

报告显示，仅通过更改列顺序，我们便可以节省大约 28% 的磁盘空间。注意这是一个估算值。

让我们检查一下优化后的顺序：

```sql
drop table t;

create table t(
  is_public boolean,
  verified boolean,
  score int2,
  id int4 primary key,
  created_at timestamptz,
  modified_at timestamptz,
  published_at timestamptz
);

insert into t
select
  true,
  true,
  0::int2,
  i::int4,
  clock_timestamp(),
  clock_timestamp(),
  clock_timestamp()
from generate_series(1, 1000000) as i;

vacuum analyze t;
```

再次查看表大小：

```sql
nik=# \dt+ t
                   List of relations
 Schema | Name | Type  |  Owner   | Size  | Description
--------+------+-------+----------+-------+-------------
 public | t    | table | postgres | 57 MB |
(1 row)
```

节省了约 `30%` 的磁盘空间，非常接近预期值 (57 / 81 ~= 0.7037)。

postgres_dba 的报告 p1 现在也不再显示潜在的存储节省：

```sql
Type your choice and press <Enter>:
p1
 Table | Table Size | Comment | Wasted * | Suggested Columns Reorder
-------+------------+---------+----------+---------------------------
 t     | 57 MB      |         |          |
(1 row)
```

为什么可以节省空间？

在 Postgres 中，存储系统可能会为列值添加对齐填充。若数据类型的自然对齐要求超过值的大小，Postgres 可能会在值后填充零以进行边界对齐。例如，当小于 8 字节的值后跟一个需要 8 字节对齐的值时，Postgres 会在第一个值后填充，以便在 8 字节边界对齐。对齐数据有助于确保内存中的值与特定硬件架构的 CPU 字边界对齐，从而提高性能。

例如，(int4, timestamptz) 的一行占用 16 字节：

- 4 字节用于 `int4`
- 4 字节的零值用于 8 字节对齐填充
- 8 字节用于 `timestamptz`

有些人喜欢先将 16 字节和 8 字节的列放在前面，然后依次放置更小的列。此外，建议将具有 `VARLENA` 类型 (`text`、`varchar`、`json`、`jsonb`、array) 的列放在最后。

关于此主题的相关文章：

- [StackOverflow answer by Erwin Brandstetter](https://stackoverflow.com/questions/2966524/calculating-and-saving-space-in-postgresql/7431468#7431468)
- [Ordering Table Columns in PostgreSQL (GitLab)](https://docs.gitlab.com/ee/development/database/ordering_table_columns.html)
- 官方文档：[Table Row Layout](https://postgresql.org/docs/current/storage-page-layout.html#STORAGE-TUPLE-LAYOUT)