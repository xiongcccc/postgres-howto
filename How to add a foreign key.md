# How to add a foreign key

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

添加外键 (FK) 很简单：

```sql
alter table messages
add constraint fk_messages_users
foreign key (user_id)
references users(id);
```

然而，此操作需要锁定涉及的两个表：

- 被引用表上的 `ShareRowExclusiveLock`，`RowShareLock` 和 `AccessShareLock` 锁，在本例中为 `users` (包括该表主键上的 `AccessShareLock`)。这会阻止对 `users` 表的任何数据修改 (`UPDATE`、`DELETE`、`INSERT`)，以及 DDL 操作。
- 引用表上的 `ShareRowExclusiveLock` 和 `AccessShareLock` ，在本例中为 `messages` (包括其主键的`AccessShareLock`)。同样，这也会阻止对该表的写入以及 DDL 操作。

为了确保现有数据不违反约束，需要对表进行全表扫描 — 因此，表中的数据越多，隐式扫描的时间越长。在此期间，锁会阻塞所有写入和 DDL 操作。

要避免停机，我们需要分三步创建 FK：

1. 快速定义带有 `NOT VALID` 标志的约束。
2. 对于现有数据，如果需要，修复会破坏 FK 的行。
3. 在单独的事务中，验证现有行是否满足约束。

## 第1步：使用NOT VALID添加FK

```sql
alter table messages
add constraint fk_messages_users
foreign key (user_id)
references users(id)
not valid;
```

此操作仅需短暂的 `ShareRowExclusiveLock` 和 `AccessShareLock`，所以在负载较大的系统上，建议设置较低的 `lock_timeout` 并进行重试 (参照 [Zero-downtime database schema migrations](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries))，以避免锁队列阻塞对表的写入。

🖋️ **重要**：一旦带有 `NOT VALID` 的约束生效，新的写入会立即进行约束检查 (而旧数据尚未验证，可能会违反约束)：

```sql
nik=# \d messages
              Table "public.messages"
 Column  |  Type  | Collation | Nullable | Default
---------+--------+-----------+----------+---------
 id      | bigint |           | not null |
 user_id | bigint |           |          |
Indexes:
    "messages_pkey" PRIMARY KEY, btree (id)
Foreign-key constraints:
    "fk_messages_users" FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID

nik=# insert into messages(id, user_id) select 1, -1;
ERROR:  insert or update on table `messages` violates foreign key constraint "fk_messages_users"
DETAIL:  Key (user_id)=(-1) is not present in table `users`.
```

## 第2步：如有需要，修复现有数据

添加了 `NOT VALID` 标志的 FK 后，Postgres 已经根据新约束检查了所有新数据，但旧数据可能仍有部分行违反该约束。在进行下一步操作之前，有必要确保没有违反新 FK 的旧行。可以使用以下查询来完成：

```sql
select id
from messages
where
  user_id not in (
    select id from users
  );
```

该查询会扫描整个 `messages` 表，因此可能需要较长时间。确保 `users` 通过主键访问以提高性能 (这取决于数据量和规划器设置)。

找到的行将阻止下一步操作，因此需要删除或进行更改，以避免 FK 冲突。

## 第3步：验证

完成后，需要在单独的事务中验证旧行：

```sql
alter table messages
validate constraint fk_messages_users;
```

如果表较大，此 `ALTER` 操作可能需要较长时间。然而，它仅需要获取引用表 (本例中为 `messages`) 上的 `ShareUpdateExclusiveLock` 和 `AccessShareLock`。

因此并不会阻塞 `UPDATE` / `DELETE` / `INSERT`，但会与 DDL 和 `VACUUM` 冲突。对于被引用的表 (本例中为 `users`)，需要获取 `AccessShareLock` 和 `RowShareLock`。

与往常一样，如果 `autovacuum` 在预防事务 ID 回卷的模式下处理该表，它将不会"妥协"— 因此在运行此操作之前，请确保没有 `autovacuum` 在该模式下运行，也没有 DDL 操作在进行。

## 我见

📒 TODO：包括前一篇文章，都提到了在 

>processes this table in the transaction ID wraparound prevention mode

即使不在冻结，也需要获取 `ShareUpdateExclusiveLock`，尚不明白作者为何需要特别提及？待验证。