# How to add a column

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

添加列很简单：

```sql
alter table t1 add column c1 int8;

comment on column t1.c1 is 'My column';
```

然而，可能会有一些潜在的复杂情况。

## 锁定问题

添加列需要获取表级的 `AccessExclusiveLock`，这会阻塞对该表的所有查询，包括 `SELECT`。这会产生两个后果：

1. 我们不希望操作持续太久 (例如，扫描整个表，或者更糟的是重写整个表)。
2. 锁的获取应当尽量优雅地进行。

对于第二点，在 [Zero-downtime Postgres schema migrations need this: lock_timeout and retries ](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries)中进行了详细分析。

以下是一个优雅获取锁的示例，使用较低的 `lock_timeout` 并进行重试：

```sql
do $do$
declare
   lock_timeout constant text := '50ms';
   max_attempts constant int := 1000;
   ddl_completed boolean := false;
begin

   perform set_config('lock_timeout', lock_timeout, false);

   for i in 1..max_attempts loop
      begin
         execute 'alter table t1 add column c1 int8';
         ddl_completed := true;
         exit;
      exception
         when lock_not_available then
           null;
      end;
   end loop;

   if ddl_completed then
      raise info 'DDL successfully executed';
   else
      raise exception 'DDL execution failed';
   end if;
end $do$;
```

请注意，此示例中隐式使用了子事务 (`BEGIN/EXCEPTION WHEN/END `块)。在 XID 增长率较高 (例如有许多写事务) 和长事务的情况下，在备库上这可能会触发 `SubtransSLRU` 的争用问题，详情参见：[PostgreSQL Subtransactions Considered Harmful](https://postgres.ai/blog/20210831-postgresql-subtransactions-considered-harmful)。在这种情况下，应在事务级别实现重试逻辑。

## 默认值

从 Postgres 11 开始 (因此，在所有当前支持的大版本中，截至 2023 年 11 月)，添加带有默认值的列不会导致整个表重写，因此我们通常不需要担心它。

在创建列时定义的默认值不会物理写入所有现有的行，相反，它是"虚拟的" — 存储在特殊系统目录  [pg_attrdef](https://postgresql.org/docs/current/catalog-pg-attrdef.html) 中。示例：

```sql
nik=# alter table t1 add column c2 int8 default -10;
ALTER TABLE

nik=# select pg_get_expr(adbin, 't1'::regclass::oid) from pg_attrdef;
  pg_get_expr
----------------
 '-10'::integer
(1 row)
```

而对于已存在的列，所定义的默认值存储在 `pg_attribute` 中：

```sql
nik=# alter table t1 alter column c2 set default -20;
ALTER TABLE

nik=# select attmissingval from pg_attribute where attrelid = 't1'::regclass::oid and attname = 'c2';
 attmissingval
---------------
 {-10}
(1 row)
```

这也意味着，在添加新列时，如果希望"虚拟"地为所有现有行回填值 A，但对未来的所有行使用值 B，可以：

1. 在列创建时使用一个默认值。
2. 创建后立即更改默认值。

如果使用的是较早的 Postgres 版本 (11 之前)，建议使用回填 (backfill) 操作以避免长时间锁定。

## NOT NULL

添加 `NOT NULL` 约束 (这是 [重新] 定义主键所必需的)，通常需要全表扫描，不支持两步添加法以避免长时间的锁定。

> 译者注：此处原文是 there is no support of two-step addition to avoid long-lasting locking，即不支持像定义主键那样，通过唯一索引 + 检查约束 (not null) 以降低锁的粒度。

但是，当新列需要此约束时，我们可以使用此技巧

1. 在创建列时使用一些临时的默认值与 NOT NULL 相结合：

```sql
alter table t1
add column id_new int8 not null default -1;

comment on column "t1"."id_new" is 'my future PK';
```

2. 注意此列中现有的值，以及新出现的值：

- 添加触发器为新行自动填充值。
- 批量回填现有行中的值。

3. 完成后，删除临时的默认值：

```sql
alter table t1 alter column id_new drop default;
```

4. 切换到该新列的常规用途，然后如果需要，进行清理 (删除触发器等)

## 回填

在某些情况下，单值 `DEFAULT` 不足以为所有现有的行定义新列中的值，我们仍然需要进行回填。为避免长时间锁定，建议按批次进行更新：

1. 对于 OLTP (如 Web 和移动应用)，建议控制批量更新的时间在 1-2 秒内。
2. 为了能够高效地找到下一个批处理的范围，我们可以在新列和现有 PK 上创建一个索引 (此索引可能是临时的，以支持高效批处理)，然后删除。此索引可以是部分索引。例如，如果我们的新列名为 `id_new`，并且在列创建时使用的默认值为 `-1`：

- 创建支持索引

```sql
create index concurrently i_t1_id_new on t1(id) where "id_new" = -1;
```

- 对于批处理，定义 UPDATE 的范围：


```sql
update t1 set id_new = <value> where id_new = -1 order by id limit <batch_size>;
```

- 注意监控死元组数量和 `autovacuum` 的行为，避免死元组数量过高 (导致膨胀)。必要时限制更新频率或者/并且时不时手动执行一下 `VACUUM`。


- 如果支持索引不再需要，可以删除：


```sql
drop index concurrently i_t1_id_new;
```

## 关于默认值存储的内部机制的修正

- `pg_attrdef` 存储了所有当前的默认值。当我们更改已存在列的默认值时，该系统目录会更新以存储新值：

```sql
nik=# alter table t1 alter column c2 set default -30;
ALTER TABLE

nik=# select pg_get_expr(adbin, 't1'::regclass::oid) from pg_attrdef;
  pg_get_expr
----------------
 '-30'::integer
(1 row)
```

而存储在 `pg_attribute` 中的 `attmissingval` 值是在列创建之前，存在的行所使用的默认值：

```sql
nik=# select attmissingval from pg_attribute where attrelid = 't1'::regclass::oid and attname = 'c2';
 attmissingval
---------------
 {-10}
(1 row)
```
