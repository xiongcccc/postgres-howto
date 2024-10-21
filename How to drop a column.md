# How to drop a column

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

删除一列很简单：

```sql
alter table t1 drop column c1;
```

然而，需要注意在某些情况下可能出现的一些复杂情况。

## 风险1：应用代码未准备好

应用代码需要停止使用该列。这意味着代码需要先部署。

## 风险2：部分停机

在高负载下，如果没有设置较低的 `lock_timeout` 和重试机制，执行此类 `ALTER` 操作是一个糟糕的主意，因为该语句需要获取表的 `AccessExclusiveLock` 。如果尝试获取锁的时间较长 (例如，由于现有事务持有此表上的任何一个锁 — 可能是读取表中一行数据的事务，或者是为了防止事务 ID 回卷而处理该表的`autovacuum`)，这将对当前所有查询造成阻塞，导致项目在高负载下出现部分停机。

解决方案：使用较低的 `lock_timeout` 和重试机制。以下是一个示例 (有关此问题的更多详细信息和更进阶的示例，请参照 [zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries))：

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
         execute 'alter table t1 drop column c1';
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

请注意，在此示例中，子事务是隐式使用的 (`BEGIN/EXCEPTION WHEN/END`块)。在高 `XID` 增长率的情况下 (例如，有大量写入事务) 和长时间运行的事务中，这可能成为一个问题 — 可能会在从库上触发`SubtransSLRU` 的争用 (参照：[PostgreSQL Subtransactions Considered Harmful](https://postgres.ai/blog/20210831-postgresql-subtransactions-considered-harmful))。在这种情况下，应在事务级别实现重试逻辑。

## 风险3：错误地预期数据会被删除

最后，在不同环境之间复制数据并删除敏感数据时，请记住 `ALTER TABLE ... DROP COLUMN ...` 并不安全，它不会真正删除数据。删除列 `c1` 后，元数据中仍然存在相关信息：

```sql
nik=# select attname from pg_attribute where attrelid = 't1'::regclass::oid order by attnum;
           attname
------------------------------
 tableoid
 cmax
 xmax
 cmin
 xmin
 ctid
 id
 ........pg.dropped.2........
(8 rows)
```

超级用户可以轻松恢复该列：

```sql
nik=# update pg_attribute
  set attname = 'c1', atttypid = 20, attisdropped = false
  where attname = '........pg.dropped.2........';
UPDATE 1
nik=# \d t1
                 Table "public.t1"
 Column |  Type   | Collation | Nullable | Default
--------+---------+-----------+----------+---------
 id     | bigint  |           |          |
 c1     | bigint  |           |          |
```

一些解决方法：

- 在删除列后使用 `VACUUM FULL` 重建表。在这种情况下，虽然尝试恢复会成功，但数据将不会存在。
- 考虑使用受限用户和列级别的权限，而不是删除列。列和数据仍然存在，但用户无法读取。当然，如果严格要求删除数据，此方法不适用。
- 在删除列之后转储/恢复。