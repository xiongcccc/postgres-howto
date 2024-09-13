# Understanding how sparsely tuples are stored in a table

今天我们来讨论元组以及元组在页面中的位置——这是相当基础的内容，但在许多情况下非常有用。

了解表中行的物理布局在某些情况下可能非常重要，尤其是在进行性能优化时。

## 一些术语

- Page / buffer / block：磁盘和 Postgres 缓冲池中的存储单元 (加载到 RAM 中不会发生变化)，通常大小为 8 KiB (可以通过 `show block_size;` 查看)，它包含表或索引的一部分。
- Tuple (元组)：表中一行的物理版本。
- Tuple header (元组头)：元组的元数据，包括事务 ID、可见性信息等。
- Transaction ID (事务 ID，也称为 XID、`tid`、`txid`) — Postgres 中事务的唯一标识符：
  - 事务 ID 是为修改事务分配的。只读事务拥有"虚拟事务 ID"，以避免浪费 XID，因为截至 Postgres 16，XID 仍然是 32 位的。目前正在[进行工作](https://commitfest.postgresql.org/43/3594/)以切换到 64 位。
  - 你可以通过调用 `pg_current_xact_id()` (或在 PG12 及更早版本中使用 `txid_current()`) 获取你的事务 XID。

元组头部中的有趣的"隐藏"或"系统"列 ([docs](https://postgresql.org/docs/current/ddl-system-columns.html)):

- `ctid`：表示元组在表中的物理位置的隐藏 (系统) 列，格式为两个整数 `(X, Y)`，其中：
  - `X` 是页面编号，从 0 开始
  - `Y` 是页面内元组的顺序编号，从 1 开始
- `xmin`, `xmax`：创建该行版本 (元组) 和删除它 (将其标记为"无效") 的事务 XID。

## ctid

如果想了解元组的存储稀疏程度，我们可以将 `ctid` 包含在查询的 `SELECT` 子句中。例如，我们有以下表和一个简单查询：

```sql
nik=# \d t1
                      Table "public.t1"
 Column  |       Type       | Collation | Nullable | Default
---------+------------------+-----------+----------+---------
 id      | bigint           |           | not null |
 user_id | double precision |           |          |
Indexes:
    "t1_pkey" PRIMARY KEY, btree (id)
    "t1_user_id_idx" btree (user_id)

nik=# select * from t1 where user_id = 101469;
   id   | user_id
--------+---------
  28414 |  101469
 235702 |  101469
 478876 |  101469
 495042 |  101469
 555593 |  101469
 626491 |  101469
 635785 |  101469
 702725 |  101469
(8 rows)
```

为了了解这些行的物理位置，只需将 `ctid` 包含到相同查询的 `SELECT` 子句中：

```sql
nik=# select ctid, * from t1 where user_id = 101469;
    ctid    |   id   | user_id
------------+--------+---------
 (153,109)  |  28414 |  101469
 (1274,12)  | 235702 |  101469
 (2588,96)  | 478876 |  101469
 (2675,167) | 495042 |  101469
 (3003,38)  | 555593 |  101469
 (3386,81)  | 626491 |  101469
 (3436,125) | 635785 |  101469
 (3798,95)  | 702725 |  101469
(8 rows)
```

每行存储在不同的页面中 (153, 1274 等)。这种情况对查询性能并不是最优的 — 需要大量的 IO 操作。

如果我们想查看页面 1274 中有哪些元组，我们可以通过将 `ctid` 转换为 `point` 类型 (通过文本转换，因为无法直接转换)，并提取第一个数字 — 页面编号来实现：

```sql
nik=# select ctid, * from t1 where (ctid::text::point)[0] = 1274 order by ctid limit 15;
   ctid    |   id   | user_id
-----------+--------+---------
 (1274,1)  | 235691 |  225680
 (1274,2)  | 235692 |  617397
 (1274,3)  | 235693 |  233968
 (1274,4)  | 235694 |  714957
 (1274,5)  | 235695 |  742857
 (1274,6)  | 235696 |  837441
 (1274,7)  | 235697 |  745413
 (1274,8)  | 235698 |  752474
 (1274,9)  | 235699 |  102335
 (1274,10) | 235700 |  908715
 (1274,11) | 235701 |  909036
 (1274,12) | 235702 |  101469
 (1274,13) | 235703 |  599451
 (1274,14) | 235704 |  359470
 (1274,15) | 235705 |  196437
(15 rows)

nik=# select count(*) from t1 where (ctid::text::point)[0] = 1274;
 count
-------
   185
(1 row)
```

总体上有 185 行，`user_id=101469` 的行也在其中，位于第 12 个偏移量处。

(注意，然而，对于大表来说，这将是一个非常慢的查询，因为它需要进行顺序扫描 (Seq Scan)，并且我们不能在 `ctid` 或其他系统列上创建索引。但对于那些目标是查找特定 `ctid` 的查询 (例如 `...where ctid = '(123, 456)'`)，由于使用了 Tid Scan，性能会很好，参见 https://pgmustard.com/docs/explain/tid-scan）。

## ctid 与执行计划中的 BUFFERS 指标

确认原始查询确实涉及到了许多缓冲区操作 (见第一天我们讨论了 `BUFFERS` 的重要性)：

```sql
nik=# explain (analyze, buffers, costs off) select * from t1 where user_id = 101469;
                                                     QUERY PLAN
---------------------------------------------------------------------------------------------------------------------
 Index Scan using t1_user_id_idx on t1  (actual time=0.131..0.367 rows=8 loops=1)
   Index Cond: (user_id = '101469'::double precision)
   Buffers: shared hit=11
 Planning Time: 0.159 ms
 Execution Time: 0.383 ms
(5 rows)
```

11 次缓冲区命中，相当于 88 KiB。为了获取 8 行数据，Postgres 执行器必须处理 88 KiB 来返回 317 字节，这远非最佳情况。既然这里有 `Index Scan`，一些缓冲区命中是与索引相关的，一些则是从堆中获取数据。

## 如何改善？

**选项 0**：什么也不做，但了解正在发生什么。可能你不需要进行显著改进，因为以下讨论的任何选项都不完美。避免过度优化，但要了解元组分布的稀疏程度，并准备好重新检查它。在某些情况下，目标元组存储过于稀疏可能是导致查询性能下降的一个重要因素，甚至可能导致超时。在这种情况下，可以考虑以下策略。

**选项 1**：保持表和索引的良好状态：

- 表膨胀控制：通过精心调整的 autovacuum 进行定期分析、预防，并通过 `pg_repack` 进行定期清理。
- 索引维护：同样的膨胀控制 + 定期重建索引，因为即使 autovacuum 调整得很好，索引健康状况也会随着时间的推移下降。
- 分区：分区的好处之一是改善数据的局部性。

**选项 2**：使用索引仅扫描，而不是索引扫描。可以通过使用多列索引或覆盖索引来实现，以包含查询所需的所有列。对于我们的示例：

```sql
nik=# create index on t1(user_id) include (id);
CREATE INDEX

nik=# explain (analyze, buffers, costs off) select * from t1 where user_id = 101469;
                                       QUERY PLAN
-----------------------------------------------------------------------------------------
 Index Only Scan using t1_user_id_id_idx on t1 (actual time=0.040..0.044 rows=8 loops=1)
   Index Cond: (user_id = '101469'::double precision)
   Heap Fetches: 0
   Buffers: shared hit=4
 Planning Time: 0.179 ms
 Execution Time: 0.072 ms
(6 rows)
```

4 次缓冲区命中，而不是 11 次，效果更好。

**选项 3**：根据索引或列值对表进行物理重组：这是对表进行物理重组，但它有两个缺点：

- 你需要选择一个索引进行重组 — 只能选择一个索引。因此，它仅对工作负载中特定的查询子集有帮助，对其他查询可能没有任何作用。
- 行更新会移动元组，从而降低 `CLUSTER` 带来的好处，因此可能需要重复执行。

有两种方法可以重组表：

- SQL 命令 `CLUSTER` ([文档](https://www.postgresql.org/docs/current/sql-cluster.html)) — 这不是在线操作，不建议用于无法承受维护窗口的实时系统。
- pg_repack 提供了 `--order-by=<..>` 选项，它允许在不需要停机的情况下，以在线方式实现类似于 `CLUSTER` 的效果。

以我们的示例为例：

```sql
nik=# cluster t1 using t1_user_id_idx;
CLUSTER

nik=# explain (analyze, buffers, costs off) select * from t1 where user_id = 101469;
                                   QUERY PLAN
---------------------------------------------------------------------------------
 Index Scan using t1_user_id_idx on t1 (actual time=0.035..0.040 rows=8 loops=1)
   Index Cond: (user_id = '101469'::double precision)
   Buffers: shared hit=4
 Planning Time: 0.141 ms
 Execution Time: 0.070 ms
(5 rows)
```

同样是 4 次缓冲区命中，与覆盖索引和 Index-Only Scan 的效果相同。不过，此处我们有的是 `Index Scan`。

---

今天的讨论到此为止——我们讨论了 `ctid`。以后，我们会继续探讨 `xmin` 和 `xmax`，以及出于实际原因对表/索引页面的深入检查。

别忘了，欢迎订阅、分享、点赞！ 此外，我在 GitLab 上同步了这些技巧： https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/