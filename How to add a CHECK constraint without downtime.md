# How to add a CHECK constraint without downtime

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

添加 `CHECK` 约束有益于：

- 提升数据质量
- 在 PG12+ 中，定义 `NOT NULL` 约束不需要停机 (更多信息参考[此处](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0060_how_to_add_a_column.md?ref_type=heads#not-null))

要在不中断服务的情况下添加 `CHECK` 约束，我们需要：

1. 快速定义带有 `NOT VALID` 标志的约束
2. 在一个单独的事务中，"验证"现有行是否满足该约束

## 使用NOT VALID添加CHECK约束

示例：

```sql
alter table t
add constraint c_id_is_positive
check (id > 0) not valid;
```

这仅需要一个非常短暂的 `AccessExclusiveLock`，因此在负载较大的系统上，应设置较低的 `lock_timeout` 并进行重试 (参考：[Zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries))。

🖋️ **重要**：当 `NOT VALID `约束添加之后，新的数据写入会立即进行检查 (而旧数据尚未验证，可能会违反约束)：

```sql
nik=# insert into t select -1;
ERROR:  new row for relation "t" violates check constraint "c_id_is_positive"
DETAIL:  Failing row contains (-1).
```

## 验证

完成添加约束的过程后，我们需要验证旧数据：

```sql
alter table t
validate constraint c_id_is_positive;
```

该操作会扫描整个表，因此对于大表可能需要较长时间 — 但此查询仅获取表上的 `ShareUpdateExclusiveLock`，不会阻塞运行 DML 查询的会话。但是，如果有 `autovacuum` 在预防事务 ID 回卷的模式下运行并处理该表，或有其他会话在该表上创建索引或执行其他 `ALTER` 操作，则锁请求会被阻塞。因此，在运行 `ALTER` 之前，应确保没有这些繁重的操作在执行，以避免过长的等待时间。

## 我感

这个技巧很不错，在创建分区表的时候也可以使用这个技巧，直接 alter table xxx add constraint 会全程 8 级锁，可以先 not valid，再慢慢校验，最后 attach。