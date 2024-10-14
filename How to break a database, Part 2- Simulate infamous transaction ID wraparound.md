# How to break a database, Part 2: Simulate infamous transaction ID wraparound

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

参见第一部分：[Part 1: How to Corrupt](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0039_how_to_break_a_database_part_1_how_to_corrupt.md)

## 直接模拟方法

这个方法需要一些时间，但效果很好。已在多个地方讨论过：

-  [A question from Telegram chat](https://twitter.com/samokhvalov/status/1415575072081809409)
- [How to simulate the deadly, infamous, misunderstood & complex ‘Transaction Wraparound Problem’ in PostgreSQL](https://fatdba.com/2021/07/20/how-to-simulate-the-deadly-transaction-wraparound-problem-in-postgresql/)

首先，开启一个"长时间运行的事务" (分配一个正常的 XID 给它，因为它将是一个写事务) — 并保持该会话处于开启状态 (我们将使用一个命名管道，也称为 FIFO，让 `psql` 会话夯住)：

```bash
mkfifo dummy

psql -Xc "
  set idle_in_transaction_session_timeout = 0;
  begin;
  select pg_current_xact_id()
" \
-f dummy &
```

确保会话已开启并处于事务中：

```bash
psql -Xc "select state
  from pg_stat_activity
  where
    pid <> pg_backend_pid()
    and query ~ 'idle_in_tran'
"
        state
---------------------
 idle in transaction
(1 row)
```

现在，长时间运行的事务已经分配了 XID，我们只需要高速推进事务 ID 即可 — 可以使用同一个函数，以及具有多个连接的 `pgbench`：

```bash
pgbench -c8 -j8 -P60 -T36000 -rn \
  -f - <<< 'select pg_current_xact_id()'
```

这应该会以非常高的速度 (100-200k TPS) 推进当前 XID。而且由于我们有一个处于开启状态的长事务，因此 `autovacuum` 无法运行。

在 `pgbench` 运行时，可以使用包含 XID/MultiXID 回卷检查的监控工具观察数据库状态 (每个工具都应该有，但并非所有工具)，或者使用一些代码片段 — 例如，[Managing Transaction ID Exhaustion (Wraparound) in PostgreSQL](https://crunchydata.com/blog/managing-transaction-id-wraparound-in-postgresql)

几小时后：

~~~bash
WARNING:  database "nik" must be vacuumed within 39960308 transactions
HINT:  To avoid a database shutdown, execute a database-wide VACUUM in that database.
You might also need to commit or roll back old prepared transactions, or drop stale replication slots.
~~~

如果继续：

```bash
nik=# create table t3();
ERROR:  database is not accepting commands to avoid wraparound data loss in database "nik"
HINT:  Stop the postmaster and vacuum that database in single-user mode.
You might also need to commit or roll back old prepared transactions, or drop stale replication slots.
```

太好了，损害已经完成！现在我们可以释放 FIFO (这将关闭长时间的"闲置中的事务"会话)：

```bash
exec 3>dummy && exec 3>&-
```

## 如何解决 — 基础

如何摆脱这种情况？我们可能会在另一个教程中详细讨论，但让我们先谈谈一些基础知识。

查看实际用例，从其他人的错误中吸取教训：

- [Sentry's case](https://blog.sentry.io/transaction-id-wraparound-in-postgres/),
- [Mailchimp's one](https://mailchimp.com/what-we-learned-from-the-recent-mandrill-outage/)

解决此问题通常需要很多时间。传统上，如上面的 `HINT` 所建议的那样，在单用户模式下完成。但我非常喜欢来自 [@PostSQL](https://twitter.com/PostSQL) 的想法 — 来自 GCP 团队 — 参照  [Do you vacuum every day?](https://youtube.com/watch?v=JcRi8Z7rkPg)  以及 `pgsql-hackers` 邮件列表 [We should stop telling users to "vacuum that database in single-user mode"](https://postgresql.org/message-id/flat/CAMT0RQTmRj_Egtmre6fbiMA9E2hM3BsLULiV8W00stwa3URvzA@mail.gmail.com).

## 另一种模拟回卷的方法

还有另一种模拟方法，它工作得更快 — 无需等待很多小时，可以使用 `pg_resetwal` 的 `-x` 选项来快速推进事务ID。

这个方法在 [Transaction ID wraparound: a walk on the wild side](https://cybertec-postgresql.com/en/transaction-id-wraparound-a-walk-on-the-wild-side/) 中有所描述。它非常有趣，看起来是这样的：

```bash
pg_ctl stop -D $PGDATA

pg_resetwal -x 2137483648 -D testdb ### 2^31 - 10000000

dd if=/dev/zero of=$PGDATA/pg_xact/07F6 bs=8192 count=15
```

## MultiXact ID 回卷

MultiXact ID 回卷也有可能发生： [Transaction ID wraparound: a walk on the wild side](https://cybertec-postgresql.com/en/transaction-id-wraparound-a-walk-on-the-wild-side/) 这种风险经常被忽略，但它确实存在，尤其是如果你使用外键和/或结合子事务使用 `SELECT ... FOR SHARE` (以及 `SELECT ... FOR UPDATE`)。

一些相关的文章可以帮助你了解：

- [A foreign key pathology to avoid](https://thebuild.com/blog/2023/01/18/a-foreign-key-pathology-to-avoid/)
- [Subtransactions considered harmful: Problem 3: unexpected use of Multixact IDs](https://postgres.ai/blog/20210831-postgresql-subtransactions-considered-harmful#problem-3-unexpected-use-of-multixact-ids)
- [Notes on some PostgreSQL implementation details](https://buttondown.email/nelhage/archive/notes-on-some-postgresql-implementation-details/)

模拟 MultiXact ID 回卷也是可能的 — 为此，我们需要创建多个重叠事务来锁定相同的行 (例如，使用显式的 `SELECT ... FOR SHARE`)。

>  🎯 **TODO:** working example – left for future edits.

每个人不仅应该监控 (并设置告警) 传统 XID 回卷的风险，还应监控 MultiXID 回卷风险。这应该包括在显示"容量" "capacity" 使用情况的代码片段中。

> 🎯 **待完成：** 展示 XID 和 MultiXID 使用情况的代码片段，包括数据库级别和表级别。显然，现实中遇到 MultiXID 回卷的情况较少 — 我几乎没见到有人使用 `datminmxid` 和 `relminmxid`。基本查询：

```sql
select
  datname,
  age(datfrozenxid),
  mxid_age(datminmxid)
from pg_database;
```