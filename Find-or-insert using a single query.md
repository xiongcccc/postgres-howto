# "Find-or-insert" using a single query

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

考虑这样一个任务：你需要读取一行数据，如果该行不存在，则插入 (并读取刚插入的内容)。通过单个查询实现。

我们将使用以下表作为示例：

```sql
create table t1 (
  id bigserial primary key,
  ts timestamptz not null unique
);
```

## 方法1：INSERT ... CONFLICT

有人可能认为 `INSERT ... CONFLICT` 或 `MERGE` 很适合这种情况。其实不然。

让我们看一下：

```sql
insert into t1(ts)
select now()::timestamptz(0)
on conflict (ts) do nothing
returning *
```

该查询在发生冲突时并不会返回任何结果。根据[文档](https://www.postgresql.org/docs/current/sql-insert.html)：

> Only rows that were successfully inserted or updated will be returned.

为了修复这个问题，我们需要在查询中加入 `UPDATE`：

```sql
insert into t1(ts)
select now()::timestamptz(0)
on conflict (ts) do update set ts = excluded.ts
returning *
```

但这会导致在我们只需要 `SELECT` 的地方执行 `UPDATE`，这是一个巨大的开销，我们绝对应该避免。

另外，这两种查询都有另一个开销：每当 `ON CONFLICT` 发生冲突时，都会浪费一个序列增量值 (即使我们使用 `GENERATED ALWAYS AS IDENTITY`，因为底层也使用了序列)。

>TODO：关于本指南的后续编辑：考虑 `MERGE` 并解释为什么它也不好

## 方法 2：使用 UPDATE-or-SELECT 的简单 CTE

有人可能认为使用 CTE 会有所帮助：

```sql
with val(ts) as (
  values(now())
), inserted as (
  insert into t1(ts)
  select val.ts
  from val
  where not exists (
    select
    from t1
    where t1.ts = val.ts
  )
  returning *
)
select * from inserted
union all
select t1.* from t1 join val using (ts);
```

这确实有帮助，但只有当你单独使用数据库时才有用。在并发工作负载中，这将导致偶尔的回滚。为了观察这种情况，让我们使用 `now()::timestamptz(0)` 而不是 `now()`。这样，每秒只有一个可能的值需要查找或插入 (found-or-inserted)。我们用 8 个并发会话、TPS=100 来运行这个查询：

```bash
echo "with val(ts) as (
  values(now()::timestamptz(0))
), inserted as (
  insert into t1(ts)
  select val.ts
  from val
  where not exists (
    select
    from t1
    where t1.ts = val.ts
  )
  returning *
)
select * from inserted
union all
select t1.* from t1 join val using (ts)" > upsert.sql

❯ pgbench -c8 -j8 -R100 -P10 -T30 -nr -fupsert.sql
pgbench (15.4 (Homebrew))
pgbench: error: client 0 script 0 aborted in command 0 query 0: ERROR:  duplicate key value violates unique constraint "t1_ts_key"
DETAIL:  Key (ts)=(2023-11-01 01:00:28-07) already exists.
```

为什么会发生这些错误？因为在使用 sub-`SELECT` 检查行并尝试执行 `INSERT` 之间总有一个非常短的时间，此时另一个会话可能会执行 `INSERT`。因此，这种方法在一般情况下效果不佳。但我们可以改进它。

## 方法3：改进的CTE

以下是一个改进的版本，不会遇到上述问题：

```sql
with val(ts) as (
  values(now()::timestamptz(0))
), select_attempt as (
  select t1.*
  from t1
  join val using (ts)
), inserted as (
  insert into t1(ts)
  select val.ts
  from val
  where not exists (
    select from select_attempt
  )
  on conflict (ts) do nothing
  returning *
)
select * from select_attempt
union all -- "all" is for troubleshooting only
select * from inserted;
```

我们把它放到 `improved.sql` 文件中，并使用 pgbench 以相同方式进行测试：

```bash
❯ pgbench -c8 -j8 -R100 -P10 -T30 -nr -fimproved.sql

pgbench (15.4 (Homebrew))
progress: 10.0 s, 104.1 tps, lat 3.666 ms stddev 1.777, 0 failed, lag 2.849 ms
progress: 20.0 s, 99.3 tps, lat 3.801 ms stddev 3.299, 0 failed, lag 2.852 ms
transaction type: improved.sql
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 8
maximum number of tries: 1
duration: 30 s
number of transactions actually processed: 3046
number of failed transactions: 0 (0.000%)
latency average = 3.722 ms
latency stddev = 2.521 ms
rate limit schedule lag: avg 2.859 (max 27.065) ms
initial connection time = 22.667 ms
tps = 101.624575 (without initial connection time)
statement latencies in milliseconds and failures:
         0.847           0  with val(ts) as (
```

👉 没有失败，也没有浪费的序列值或过度的 `UPDATE` (在 `psql` 中很容易检查，使用 `now()::timestamptz(0)` 和 `\watch .2` 运行查询)。

## 📝 备注和补充说明

1. 偶尔出现空结果，导致高并发工作负载

 在上一个查询中，`select * from select_attempt` 有时可能会返回空结果 — 为了解决这个问题，我们需要从表中重新读取数据。这是个问题。用另一次的读取替代它是没有帮助的。

解决方案：使用 `ON CONFLICT DO UPDATE` 是一个可接受的解决方案，因为它是在 `SELECT` 尝试之后执行的，并且上面讨论的开销很少对我们造成影响 (与"简单 CTE" 情况下的查询失败频率相同)。

测试如下：

```bash
echo "with val(ts) as (
  values(now()::timestamptz(0))
), select_attempt as (
  select t1.*
  from t1
  join val using (ts)
), inserted as (
  insert into t1(ts)
  select ts from val
  except
  select ts from select_attempt
  on conflict (ts) do update set ts = excluded.ts
  returning *
), res as (
  select * from select_attempt
  union all -- "all" is for troubleshooting only
  select * from inserted
)
select 1/count(*) from res;
" > improved.sql

pgbench -c8 -j8 -R100 -P10 -T120 -nr -fimproved.sql
```

👉 没有 `division by zero` 的错误 (至少在我的测试中)

2. 使用 `EXCEPT` 而不是 `WHERE NOT EXISTS` 可能看起来更有吸引力 (没有子查询) — 这个想法来自 [@be_haki's Tweet](https://twitter.com/be_haki/status/1718993194938187935)，我喜欢！
3. 我希望这一切都能够变得更容易。

# 我见

行吧，看得有点懵，还需要细细品味。