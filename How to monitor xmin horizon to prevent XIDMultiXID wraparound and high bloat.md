# How to monitor xmin horizon to prevent XID/MultiXID wraparound and high bloat

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

在之前的文章中，我们讨论了[如何监控 XID 和 MultiXID 回卷的风险](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0044_how_to_monitor_transaction_id_wraparound_risks.md)。这种检查至关重要，并且是任何监控系统中不可或缺的部分。

然而，虽然它有助于了解风险级别，但并不能揭示问题的根本原因 — 当你在进行 XID 回卷的事后分析时，你肯定需要找到问题的根源，比如使用"五个为什么"方法 (开玩笑的，我们要改进监控并控制 `autovacuum` 的行为，所以我们永远不会在生产环境中遇到 XID 回卷问题)。

这个问题可以通过 `xmin` 视界监控来解决。这种检查对于理解膨胀率增长的原因也很有帮助。

## XID/MultiXID回卷风险增加的四个原因

当 `autovacuum` 未能冻结元组时，会发生 XID/MultiXID 回卷问题。即使 `autovacuum` 已开启，仍可能存在以下四个原因导致冻结失败：

1. 主节点上有长时间运行的事务。
2. 废弃的复制槽。
3. 备节点上有长时间运行的事务且 `hot_standby_feedback = on`  (包括级联复制的情况 — 反馈可以传播)。
4. 废弃的预备事务。

这四个原因都需要检查，因为它们可能包含需要保留数据的事务 ID。

相关的好文章：[VACUUM won't remove dead rows: 4 reasons why](https://cybertec-postgresql.com/en/reasons-why-vacuum-wont-remove-dead-rows/)

澄清 (基于监控系统中常见的错误分析)：

- 仅监控长时间运行的事务是不够的 — 这四个原因都必须覆盖。实际上，监控长时间运行的事务更多是为了解决另一类问题：锁风险。
- 仅监控复制槽是不够的 — 某些备节点可能不使用复制槽。此外，仅监控 `pg_stat_replication` 也不够，因为它不会涵盖废弃的复制槽。需要同时检查 `pg_stat_replication` 和 `pg_replication_slots`。

## xmin视界

"`xmin` horizon" 该术语在 Postgres 文档中有使用 (例如在描述 `pg_stat_activity.backend_xmin` 时)，虽然未明确定义。它也在源码中使用，函数 `ComputeXidHorizons()` 的注释中有一个[很好的解释](https://github.com/postgres/postgres/blob/6bf2efb38285626a9de3004dd1c23d9a85453372/src/backend/storage/ipc/procarray.c#L1662)。

一行的 `xmin` 表示插入该行的事务 ID — 每个表都有一个隐藏的 (系统) 列 `xmin` (可以尝试：`select xmin, * from your_table;`)。

"`xmin` horizon" 表示必须保留的最老数据快照的事务 ID。

## 关于膨胀

如果 `xmin` 视界在短时间内没有推进，阻止了 `autovacuum`，这通常不是问题 — 这类情况经常发生。

然而，如果这种情况持续很长时间，并且 `xmin` 视界停留在过去，那么会导致两个大问题：

1. 如前文所述，XID/MultiXID 回卷风险增加。
2. 膨胀率增长：无法及时删除死元组，导致当 `xmin` 视界移动时需要进行大量删除操作 — 这使得 `autovacuum` 成为"死元组到膨胀的转换器"。

因此，重要的是监控 `xmin` 视界，并在其滞后过多时 (即 `xmin` 视界年龄过高) 采取行动。

## 如何监控

有两种监控 `xmin` 视界的方法：

1. 观察 `autovacuum` 日志。
2. 查询四个系统视图，以覆盖上述的四个原因。

## 基于日志的监控

基于日志的方法无法帮助理解 `xmin` 视界不推进的背后原因，但它可以展示 `autovacuum` 的实际行为。

以下是一个日志示例及其解读：

```bash
2023-11-10 01:04:03.828 PST [56538] LOG:  automatic vacuum of table "nik.public.t": index scans: 0
  pages: 0 removed, 4480 remain, 4480 scanned (100.00% of total)
  tuples: 0 removed, 1000000 remain, 666667 are dead but not yet removable
  removable cutoff: 784, which was 112449 XIDs old when operation ended
  frozen: 0 pages from table (0.00% of total) had 0 tuples frozen
  index scan not needed: 0 pages from table (0.00% of total) had 0 dead item identifiers removed
  avg read rate: 9.685 MB/s, avg write rate: 0.000 MB/s
  buffer usage: 7281 hits, 1698 misses, 0 dirtied
  WAL usage: 0 records, 0 full page images, 0 bytes
  system usage: CPU: user: 0.11 s, system: 0.02 s, elapsed: 1.36 s
```

其中，问题指标有：

- `666667 are dead but not yet removable` — 大量元组已死但 `autovacuum` 无法删除它们，因为这些元组比当前 xmin 视界"更年轻" (与 `xmin` 视界相比，它们的 `xmin` 值处于未来)。
- `removable cutoff: 784, which was 112449 XIDs old when operation ended` — 这告诉我们 XID视界是 784，并且它的年龄是 112449 — 意味着当 `autovacuum` 完成此次处理时， `xmin` 视界 (仍被认为需要的数据版本) 比当前时间落后超过 112k 个事务。

这表明 `xmin` 视界远远落后于当前，某些东西使其停留在过去。要找出具体原因，我们需要检查几个系统视图。

## 使用系统视图进行监控

以下是一个查询示例：

```sql
with bits as (
  select
    (
      select backend_xmin
      from pg_stat_activity
      order by age(backend_xmin) desc nulls last
      limit 1
    ) as xmin_pg_stat_activity,
    (
      select xmin
      from pg_replication_slots
      order by age(xmin) desc nulls last
      limit 1
    ) as xmin_pg_replication_slots,
    (
      select backend_xmin
      from pg_stat_replication
      order by age(backend_xmin) desc nulls last
      limit 1
    ) as xmin_pg_stat_replication,
    (
      select transaction
      from pg_prepared_xacts
      order by age(transaction) desc nulls last
      limit 1
    ) as xmin_pg_prepared_xacts
)
select
  *,
  age(xmin_pg_stat_activity) as xmin_pgsa_age,
  age(xmin_pg_replication_slots) as xmin_pgrs_age,
  age(xmin_pg_stat_replication) as xmin_pgsr_age,
  age(xmin_pg_prepared_xacts) as xmin_pgpx_age,
  greatest(
    age(xmin_pg_stat_activity),
    age(xmin_pg_replication_slots),
    age(xmin_pg_stat_replication),
    age(xmin_pg_prepared_xacts)
  ) as xmin_horizon_age
from bits;
```

请注意，`min(...)` 函数无法直接应用于 XID 值，因为它们是 32 位且会循环 — 将 XID 转换为 `int` 是不可行的。但 `age(XID)` 函数在这里非常有用。因此，与考虑 `xmin_horizon` 值相比，我们需要处理 `xmin_horizon_age`。