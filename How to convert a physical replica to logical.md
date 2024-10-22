# How to convert a physical replica to logical

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

在某些情况下，将现有的异步物理副本转换为逻辑副本可能是有利的，或者先创建一个新的物理副本，然后将其转换为逻辑副本。

这种方式：

- 一方面，消除了加载初始数据这一步骤的必要，这个步骤在处理大型、负载繁重的数据库时可能会很脆弱且压力很大。
- 另一方面，使用这种方式创建的逻辑副本拥有源 Postgres 实例中的所有数据。

因此，这种方法非常适用于需要将源数据库中的所有数据呈现在新创建的逻辑副本中的情况，特别是在处理非常大且负载繁重的集群时极为有用。

下面的步骤十分简单。我们使用一个物理副本，通过流复制 `primary_conninfo` 和复制槽 (比如通过 Patroni 进行控制) 立即从主库复制数据，不涉及级联复制 (尽管也有可能实现)。

## 步骤1：选择一个用于转换的物理副本

选择一个物理副本进行转换，或者通过 `pg_basebackup`、从备份恢复或从云快照创建新的物理副本。

确保在转换期间没有用户使用该副本。

## 步骤2：确保满足要求

首先，确保为逻辑复制配置了相关设置，如[逻辑复制配置中](https://postgresql.org/docs/current/logical-replication-config.html)所述。

主库设置：

- `wal_level = 'logical'`
- `max_replication_slots > 0`
- `max_wal_senders > max_replication_slots`

在我们将要转换的物理副本上：

- `max_replication_slots > 0`
- `max_logical_replication_workers > 0`
- `max_worker_processes >= max_logical_replication_workers + 1`

此外：

- 复制延迟较小；
- 每个表都有主键或设置了 [REPLICA IDENTITY FULL](https://postgresql.org/docs/current/sql-altertable.html#SQL-ALTERTABLE-REPLICA-IDENTITY);
- 用于转换的副本没有设置 `restore_command` (如果有，临时将其值设为空字符串)；
- 临时增加主库上的 `wal_keep_size`  (PG13+，在 PG12 或更早版本中为 `wal_keep_segments`)，增加到对应于几个小时 WAL 生成的量。

## 步骤3：停止物理副本

关闭物理副本，并在下一步执行期间保持其关闭。这确保其位置比我们将在主库上创建的逻辑槽的位置早。

## 步骤4：创建发布、逻辑槽并记录其LSN

在主库上：

1. 手动执行一次 `CHECKPOINT`；
2. 创建发布；
3. 创建逻辑复制槽并记录其 LSN 位置。

示例：

```sql
checkpoint;

create publication my_pub for all tables;

select lsn
from pg_create_logical_replication_slot(
  'my_slot',
  'pgoutput'
);
```

请务必记住最后一条命令返回的 LSN 值 — 稍后我们会使用到它。

## 步骤5：让物理副本追赶上主库

重新配置物理副本：

- `recovery_target_lsn` — 将其设置为上一步获得的 LSN 值
- `recovery_target_action = 'promote'`
- `restore_command`,`recovery_target_timeline`,`recovery_target_xid`,`recovery_target_time`,`recovery_target_name` 为空或未设置。

现在启动物理副本，监控其延迟情况，直到副本追赶上我们需要的 LSN 并自动提升。这可能需要一些时间。完成之后进行检查：

```sql
select pg_is_in_recovery();
```

- 必须返回 `f`，这意味着该节点现在本身就是一个主节点 (一个克隆节点)，其位置与源节点上的复制槽位置相对应。

## 步骤6：创建订阅并启动逻辑复制

在新创建的"克隆"节点上，创建逻辑订阅 `copy_data = false` 和 `create_slot = false`：

```sql
create subscription 'my_sub'
connection 'host=.. port=.. user=.. dbname=..'
publication my_pub
with (
  copy_data = false,
  create_slot=false,
  slot_name = 'my_slot'
);
```

确保复制处于运行中的状态 — 在源主库上检查：

```sql
select * from pg_replication_slots;
```

确保 `active` 字段为 `t`。

## 完成

- 等待逻辑复制的延迟完全赶上 (偶尔出现的峰值是正常的)。
- 将主库上的`wal_keep_size` (或 `wal_keep_segments`) 恢复到原始值。

## 额外说明

在这个方案中，我们使用了单个发布和逻辑复制槽。也可以使用多个槽，只需对过程进行一些调整。但是，如果选择这样做，请记住使用多个复制槽/发布的潜在复杂性，首先是这些：

- 无法保证逻辑副本的引用完整性 (偶尔会暂时性地违反 FK)，
- 多个发布的创建更为脆弱 (创建 `FOR ALL TABLES` 的发布不需要表级锁，但是当我们使用多个发布，并为某些表创建发布时，需要表级锁 — 然而，这只是 `ShareUpdateExclusiveLock`，参考 [this comment on PostgreSQL source code](https://github.com/postgres/postgres/blob/1b6da28e0668eb977dcab6987d192ddedf32b752/src/backend/commands/publicationcmds.c#L1550)).

以及无论如何：

- 确保已准备好处理对应版本的逻辑复制限制 (例如，[PG16](https://postgresql.org/docs/16/logical-replication-restrictions.html))；
- 如果考虑使用此方法执行大版本升级，请避免在已转换的节点上运行 `pg_upgrade` — 这可能不安全 (参照：[pg_upgrade and logical replication](https://postgresql.org/message-id/flat/20230217075433.u5mjly4d5cr4hcfe@jrouhaud))。

# 我见

看了一下操作流程，确实十分繁琐，17 版本中的 pg_createsubscriber 无疑是个巨大的福音，大大简化了将物理副本转化为逻辑副本的步骤。

pg_createsubscriber to create a logical replica from a physical standby server

- Speed up creation of logical subscriber
- It can be used for upgrades

另外原文中所说的最后一点，在 17 版本看似也已解决。具体可以参照上面的邮件

