# How to analyze heavyweight locks, part 1

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

重量级锁，包括关系级锁和行级锁，是由查询获取的，并且始终会保持到该查询所属事务结束为止。因此，重要原则是：一旦获取了锁，便不会释放，直到 `COMMIT` 或 `ROLLBACK`。

文档：[Explicit locking](https://postgresql.org/docs/current/explicit-locking.html)。关于该文档有几点需要注意：

- 标题 "Explicit Locking"  可能会让人误解 — 它实际上描述了可以通过任何语句隐式获取的锁级别，而不仅仅是通过 `LOCK` 显式获取的锁。
- 该页面还包含一张非常有用的表格"冲突锁模式"，帮助理解某些锁因冲突无法获取，并且需要等待持有此类锁的事务完成，才能释放这些"阻塞"锁。这篇文章有一张替代表格，也可能会有帮助：[PostgreSQL rocks, except when it blocks: Understanding locks](https://citusdata.com/blog/2018/02/15/when-postgresql-blocks/)
- 还可能在术语上有些混淆。当讨论"表级锁"时，我们实际上可能是指"关系级锁"。此处，术语"关系"有更广泛的含义：表、索引、视图、物化视图。

我们如何查看某个特定事务/会话中已经获取的锁 (granted)，或正在尝试但尚未获取的锁 (pending)？

为此，Postgres 提供了一个系统视图：[pg_locks](https://postgresql.org/docs/current/view-pg-locks.html)

重要的规则是：分析应在一个单独的会话中进行，以避免"观察者效应" (即由分析本身获取的锁)。

例如，考虑以下表：

```sql
nik=# \d t1
                Table "public.t1"
 Column |  Type  | Collation | Nullable | Default
--------+--------+-----------+----------+---------
 c1     | bigint |           |          |
Indexes:
    "t1_c1_idx" btree (c1)
    "t1_c1_idx1" btree (c1)
    "t1_c1_idx10" btree (c1)
    "t1_c1_idx11" btree (c1)
    "t1_c1_idx12" btree (c1)
    "t1_c1_idx13" btree (c1)
    "t1_c1_idx14" btree (c1)
    "t1_c1_idx15" btree (c1)
    "t1_c1_idx16" btree (c1)
    "t1_c1_idx17" btree (c1)
    "t1_c1_idx18" btree (c1)
    "t1_c1_idx19" btree (c1)
    "t1_c1_idx2" btree (c1)
    "t1_c1_idx20" btree (c1)
    "t1_c1_idx3" btree (c1)
    "t1_c1_idx4" btree (c1)
    "t1_c1_idx5" btree (c1)
    "t1_c1_idx6" btree (c1)
    "t1_c1_idx7" btree (c1)
    "t1_c1_idx8" btree (c1)
    "t1_c1_idx9" btree (c1)
```

在第一个 (主) 会话中：

```sql
nik=# begin;
BEGIN

nik=*# select from t1 limit 0;
--
(0 rows)
```

我们打开了一个事务，执行了 `SELECT from t1` — 请求 0 行和 0 列，但这已足够获取关系级锁。要查看这些锁，我们首先需要获取第一个会话的 `PID`，运行以下命令：

```sql
nik=*# select pg_backend_pid();
 pg_backend_pid
----------------
          73053
(1 row)
```

然后，在另一个会话中：

```sql
nik=# select relation, relation::regclass as relname, mode, granted, fastpath
from pg_locks
where pid = 73053 and locktype = 'relation'
order by relname;
 relation |   relname   |      mode       | granted | fastpath
----------+-------------+-----------------+---------+----------
    74298 | t1          | AccessShareLock | t       | t
    74301 | t1_c1_idx   | AccessShareLock | t       | t
    74318 | t1_c1_idx1  | AccessShareLock | t       | t
    74319 | t1_c1_idx2  | AccessShareLock | t       | t
    74320 | t1_c1_idx3  | AccessShareLock | t       | t
    74321 | t1_c1_idx4  | AccessShareLock | t       | t
    74322 | t1_c1_idx5  | AccessShareLock | t       | t
    74323 | t1_c1_idx6  | AccessShareLock | t       | t
    74324 | t1_c1_idx7  | AccessShareLock | t       | t
    74325 | t1_c1_idx8  | AccessShareLock | t       | t
    74326 | t1_c1_idx9  | AccessShareLock | t       | t
    74327 | t1_c1_idx10 | AccessShareLock | t       | t
    74328 | t1_c1_idx11 | AccessShareLock | t       | t
    74337 | t1_c1_idx12 | AccessShareLock | t       | t
    74338 | t1_c1_idx13 | AccessShareLock | t       | t
    74339 | t1_c1_idx14 | AccessShareLock | t       | t
    74345 | t1_c1_idx20 | AccessShareLock | t       | f
    74346 | t1_c1_idx15 | AccessShareLock | t       | f
    74347 | t1_c1_idx16 | AccessShareLock | t       | f
    74348 | t1_c1_idx17 | AccessShareLock | t       | f
    74349 | t1_c1_idx18 | AccessShareLock | t       | f
    74350 | t1_c1_idx19 | AccessShareLock | t       | f
(22 rows)
```

**注意事项：**

- 为简洁起见，我们仅请求 `locktype` 为 `relation` 的锁。在一般情况下，我们可能对其他类型的锁也感兴趣。
- 为了查看关系名，我们将 `oid` 值转换为 `regclass`，这是获取表/索引名称的最简方式 (例如，`select 74298::oid::regclass` 返回 `t1`)。
- 锁模式 `AccessShareLock` 是最"弱"的锁，它会阻止诸如 `DROP TABLE`、`REINDEX`、某些类型的 `ALTER TABLE/INDEX` 操作。通过锁定表及其所有索引，Postgres 可确保它们在我们的事务期间始终存在。
- 再次强调：所有索引都使用 `AccessShareLock` 锁定。
- 在这种情况下，所有锁都已被授予。有人可能认为这对于 `AccessShareLock` 来说总是如此，但并非如此 — 如果有已获取 (granted) 或挂起 (pending) 的 `AccessExclusiveLock` (最"强"的锁)，那么我们尝试获取的 `AccessShareLock` 将处于挂起状态。什么时候可能出现挂起的 `AccessExclusiveLock`？例如，当尝试获取 `AccessExclusiveLock` (如 `ALTER TABLE`) 时，但有一些长时间的 `AccessShareLock`— 形成了"夹心"情况。这种场景可能导致停机时间，例如在尝试执行一个非常简单的 `ALTER TABLE .. ADD COLUMN` 操作时，如果没有采取适当的预防措施 (如设置较低的 `lock_timeout` 并进行重试)，它可能会被一个长时间运行的 `SELECT` 阻塞，然后又会阻塞后续的 `SELECT` (以及其他DML操作）。更多内容请参考：[Zero-downtime Postgres schema migrations need this: lock_timeout and retries](https://postgres.ai/blog/20210923-zero-downtime-postgres-schema-migrations-lock-timeout-and-retries).
- 仅有 16 个锁的 `fastpath` 为 `true`。当 `fastpath=false` 时，Postgres 锁管理器使用一种较慢但更全面的方法来获取锁。这个问题在第 18 天的文章中有讨论：[Over-indexing](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0018_over_indexing.md)。

# 我感

文章中提到的锁冲突表格如下，也挺清晰：

![image-20241005180426121](/Users/xiongcancan/Library/Application Support/typora-user-images/image-20241005180426121.png)

另外，重锁会形成一个公平的等待队列 (这一点不用于行级锁)。如果进程试图获取与当前锁或与队列中其他进程已请求的锁不兼容的锁，那么这个进程便会加入队列。
