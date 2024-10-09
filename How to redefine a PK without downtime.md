# How to redefine a PK without downtime

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

重新定义主键并不是一件困难的事情，但需要执行几个较为复杂的步骤。当使用"新列"方法 (单独讨论) 时，该过程也是 `int4`->`int8` 主键转换的一部分。

当然，我们可以简单地删除主键并定义一个新主键，如下所示：

```sql
ALTER TABLE your_table
DROP CONSTRAINT your_table_pkey;

ALTER TABLE your_table
ADD PRIMARY KEY (new_column);
```

但这种直截了当的方式通常是个糟糕的主意，因为它会在表上获取 `AccessExclusiveLock`，并持有锁很长时间，因为：

- 需要建立唯一约束；
- 需要建立非空约束。

这是因为创建一个主键需要两个要素：唯一约束，以及参与主键定义的所有列上的非空约束。幸运的是，在现代 PostgreSQL (PG12+) 中，可以避免长时间的排它锁 — 也就是说，可以实现真正的"在线"或"零停机"操作。

以下是假设前提：

- 新的主键列已经存在并且已经预填充；
- 如果有更多的 `INSERT` 和 `UPDATE` 操作，数据会继续被填充 — 因此数据已经存在；
- 对应的列已经实现了唯一性；
- 对应的列没有任何空值。

请注意，最后一条条件非常重要 — 与唯一键不同，主键要求定义中的所有列都必须有非空约束。

## NOT NULL：好消息和坏消息 (最终，都是好消息)

让我们深入探讨细节 — NOT NULL 值得这么做。我们会有很多好消息和坏消息。我们将深入探讨与主键不一定相关但仍相关的细节。最终我们将回到重新定义主键的任务。请耐心听我说。

坏消息：不幸的是，向现有列添加 `NOT NULL` 约束意味着 PostgreSQL 需要执行长时间的全表扫描，而在此期间会获取 `ALTER TABLE` 持有的 `AccessExclusiveLock`，这显然不是我们想要的。

好消息：自 PostgreSQL 11 起，我们可以通过一些技巧来应对这一问题。如果我们需要添加一个带有 `NOT NULL` 的列，我们可以受益于 PG11 的新功能 — 新列的非阻塞 `DEFAULT`，并将其与 `NOT NULL` 结合起来，例如：

```sql
ALTER TABLE t1
ADD COLUMN new_id int8 NOT NULL DEFAULT -1;
```

这是非常快的，因为 PG11 对新列的默认值进行了优化 (它是"虚拟"的，不会重写整个表)。

>ability to avoid a table rewrite for `ALTER TABLE ... ADD COLUMN` with a non-null column default
>
>([PG11 release notes](https://postgresql.org/docs/release/11.0/))

而且由于所有行都是预先填充的 ("虚拟地"，但这并不重要)，我们可以立即获得 `NOT NULL`，避免长时间等待。

坏消息：这种方法仅适用于新列。如果我们要对现有列添加 `NOT NULL` 约束，这个方法行不通。

好消息：如果我们只是需要一个"not null"，而不考虑其具体定义，我们可以使用 `CHECK` 约束。`CHECK` 约束的好处在于，它的定义可以分为两个阶段：

1. 首先，我们可以定义 `CHECK (col1 IS NOT NULL)` 并使用 `NOT VALID` ，这样操作很快，不会阻塞其他会话，因为不会立即检查现有的数据行 (不过仍会阻塞 — 它仍然是一个 ALTER TABLE，但只持续很短的时间；当然，仍然需要重试机制和较低的 lock_timeout，参照 [Zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries))。
2. 然后，我们执行 `ALTER TABLE ... VALIDATE CONSTRAINT ...` 来进行校验，这一步虽然慢，但不会阻塞其他操作。

坏消息：由于我们的最终目标是重新定义主键，因此 `CHECK` 约束对我们无效，因为主键需要真正的 `NOT NULL` 约束。

好消息：在 PG12+ 中，有一项优化允许 `NOT NULL` 约束依赖于现有的 `CHECK (... IS NOT NULL)` 约束：

>Allow ALTER TABLE ... SET NOT NULL to avoid unnecessary table scans
>
>([PG12 release notes](https://postgresql.org/docs/release/12.0/))

这意味着我们只需要这样做：

1. 创建一个 `CHECK` 约束，确保列不为空，并使用 `NOT VALID`  (获取具有较低 `lock_timeout` 的简短排他锁，如果需要，多次重试)。
2. 在单独的事务中进行校验。
3. 然后，为列添加 `NOT NULL` 约束，这一步会非常快 (同样，较低的 lock_timeout 以及重试机制)。
4. 最后，删除 `CHECK` 约束 (同样，较低的 lock_timeout 以及重试机制)。

有趣的是，如果我们的最终目标是创建主键，那么可以跳过步骤 3 — 在创建主键期间，将隐式创建 `NOT NULL` 约束；而且由于已经存在 `CHECK (...NOT NULL)`，速度会很快。

## 唯一约束

创建新的主键时，另一个必要条件是唯一约束。幸运的是，它可以分两个阶段创建，从而避免长时间的独占锁：

1. 借助 `CONCURRENTLY` 选项，以"零停机"方式创建唯一索引 — 为该索引命名非常重要，因为我们稍后会使用这个名称：

```sql
create unique index concurrently new_unique_index
on your_table using btree(your_column);
```

2. 定义主键时使用此索引 (... `USING INDEX` ...)

## 完整的流程

现在，让我们完成拼图，拨云见雾。

以零停机方式创建主键包括以下五个步骤：

1. 使用 `NOT VALID` 选项创建 `CHECK（...IS NOT NULL）`约束✂️：

```sql
alter table your_table
add constraint your_table_your_column_check
  check (your_column is not null) not valid;
```

2. 验证约束 (可能需要较长时间)：

```sql
alter table your_table
validate constraint your_table_your_column_check;
```

3. 使用 `CONCURRENTLY` 创建唯一索引：

```sql
create unique index concurrently u_your_table_your_column
on your_table using btree(your_column);
```

4. 基于现有的唯一索引和 `CHECK` 约束定义主键 (隐式创建 `NOT NULL` 跳过全表扫描)✂️：：

```sql
alter table your_table
add constraint your_table_pkey primary key
using index u_your_table_your_column;
```

5. 删除 `CHECK` 约束以清理环境✂️:：

```sql
alter table your_table
drop constraint your_table_your_column_check;
```

(✂️ - 建议使用较低的 lock_timeout 和重试机制)