## How to work with arrays, part 2

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

第一部分可以在[此处](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0028_how_to_work_with_arrays_part_1.md)找到。

## 如何搜索

数组搜索最有趣的操作符之一是 `<@`  — 表示"被包含"：

```sql
nik=# select array[2, 1] <@ array[1, 3, 2];
 ?column?
----------
 t
(1 row)
```

当你需要检查某个标量值是否存在于数组中时，只需创建一个单元素数组并将 `<@` 应用于两个数组。例如，检查 "2" 是否包含在要分析的数组中：

```sql
nik=# select array[2] <@ array[1, 3, 2];
 ?column?
----------
 t
(1 row)
```

你可以通过创建索引来加速使用 `<@` 的搜索操作。让我们创建一个 GIN 索引，并比较一个包含一百万行的表的查询计划：

```sql
nik=# create table t1 (val int8[]);
CREATE TABLE

nik=# insert into t1
select array(
  select round(random() * 1000 + i)
  from generate_series(1, 10)
  limit (1 + random() * 10)::int
)
from generate_series(1, 1000000) as i;
INSERT 0 1000000

nik=# select * from t1 limit 3;
          val
-----------------------
 {390,13,405,333,358,592,756,677}
 {463,677,585,191,425,143}
 {825,918,303,602}
(3 rows)

nik=# vacuum analyze t1;
VACUUM
```

我们创建了一个有 100 万行的单列表，每行包含具有各种数字的 `int8[]` 数组。

现在，搜索数组值中包含 "123" 的所有行：

```sql
nik=# explain (analyze, buffers) select * from t1 where array[123]::int8[] <@ val;
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
  Gather  (cost=1000.00..18950.33 rows=5000 width=68) (actual time=0.212..100.572 rows=2 loops=1)
    Workers Planned: 2
    Workers Launched: 2
    Buffers: shared hit=12554
    ->  Parallel Seq Scan on t1  (cost=0.00..17450.33 rows=2083 width=68) (actual time=61.293..94.212 rows=1 loops=3)
          Filter: ('{123}'::bigint[] <@ val)
          Rows Removed by Filter: 333333
          Buffers: shared hit=12554
  Planning:
    Buffers: shared hit=6
  Planning Time: 0.316 ms
  Execution Time: 100.586 ms
(12 rows)
```

共有 12554 次缓冲区命中，约为 12554 * 8 / 1024 ~= 98 MiB，只需找到包含 "123" 的两行数据 — 注意 "rows=2"。效率不高，因为这里是 Seq Scan。

现在，创建一个 GIN 索引：

```sql
nik=# create index on t1 using gin(val);
CREATE INDEX
```

然后再运行相同的查询：

```sql
nik=# explain (analyze, buffers) select * from t1 where array[123]::int8[] <@ val;
                                                      QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
  Bitmap Heap Scan on t1  (cost=44.75..4260.25 rows=5000 width=68) (actual time=0.021..0.022 rows=2 loops=1)
    Recheck Cond: ('{123}'::bigint[] <@ val)
    Heap Blocks: exact=1
    Buffers: shared hit=5
    ->  Bitmap Index Scan on t1_val_idx  (cost=0.00..43.50 rows=5000 width=0) (actual time=0.016..0.016 rows=2 loops=1)
          Index Cond: (val @> '{123}'::bigint[])
          Buffers: shared hit=4
  Planning:
    Buffers: shared hit=16
  Planning Time: 0.412 ms
  Execution Time: 0.068 ms
(11 rows)
```

没有顺序扫描，并且只有 5 次缓冲区命中，大约 40 KiB 的内存就能找到所需的 2 行数据。这解释了为什么执行时间从 ~100ms 缩短到 ~0.07ms，这快了 ~1400 倍。

更多操作符请参见[官方文档](https://www.postgresql.org/docs/current/functions-array.html#FUNCTIONS-ARRAY)。