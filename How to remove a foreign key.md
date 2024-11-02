# How to remove a foreign key

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

删除一个外键 (FK)  非常简单：

```sql
alter table messages
drop constraint fk_messages_users;
```

然而，在高负载下，这并不是一个安全的操作，原因参照 [zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries) 文中所述：

- 此操作在两个表上均需要获取短暂的 `AccessExclusiveLock`。
- 如果至少有一个锁无法快速获取，此操作便需要等待，可能会阻塞其他会话 (包括 `SELECT`) 对这两个表的访问。

为了解决这个问题，我们需要使用较低的 `lock_timeout` 并进行重试：

```sql
set lock_timeout to '100ms';
alter ... -- be ready to fail and try again.
```

同样的技巧也需要用于创建一个新外键操作的第一步，当我们使用 `NOT VALID` 选项定义一个新外键时，正如我们在第 70 天的 [How to add a foreign key](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0070_how_to_add_a_foreign_key.md) 中讨论的那样。

另外请参考：[day 71, How to understand what's blocking DDL](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0071_how_to_understand_what_is_blocking_ddl.md)