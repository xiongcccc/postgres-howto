# How to monitor transaction ID wraparound risks

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

事务 ID 和组事务 ID 的回卷问题是 Postgres 数据库中最严重的事件之一。著名的案例包括：

- [Sentry (2015)](https://blog.sentry.io/transaction-id-wraparound-in-postgres/)
- [Joyent (2015)](https://tritondatacenter.com/blog/manta-postmortem-7-27-2015)
- [Mailchimp (2019)](https://mailchimp.com/what-we-learned-from-the-recent-mandrill-outage/)

一些有价值的参考资源：

- [Postgres docs](https://postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND)
- Hironobu Suzuki 的书《The Internals of PostgreSQL》： [Vacuum Processing](https://interdb.jp/pg/pgsql06.html)
- Egor Rogov 的书《PostgreSQL 14 Internals》
- [PostgreSQL/Wraparound and Freeze](PostgreSQL/Wraparound and Freeze)

每个监控设置必须包含两类特定的指标 (以及适当的警报)：

1. 非冻结元组中使用的最老 XID 和 MultiXID (又称 MultiXact ID) 值
2. 监控 `xmin` 视界

在这里我们讨论第一类。

## 32位事务ID

XID 和 MultiXact ID 都是 32 位的，所以总的空间大小为 2^32 ≈ 42 亿。但由于使用了取模算术的原因，XID 有一个"未来"的概念 — 空间的一半被认为是过去的，另一半是未来的。因此，我们需要监控的容量是 2^31 ≈ 21 亿。

来自维基百科的一个例子： [Wikipedia "wraparound and freeze"](https://en.wikibooks.org/wiki/PostgreSQL/Wraparound_and_Freeze):

![Wraparound and freeze](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0044_wraparound_and_freeze.jpg)

为了防止 XID/MultiXID 回卷，`autovacuum` 进程除了其他任务之外，还会定期执行"冻结"操作，将老的元组标记为已冻结，表示这些元组是属于过去的。

因此，监控元组的 XID/MultiXID 年龄对于控制回卷风险至关重要。

## XID和MultiXID回卷风险监控

最常见的监控方法是：

- 检查 `pg_database.datfrozenxid`，以了解 `autovacuum` 是否正常执行"冻结"操作，替换元组中老的 XID。
- 要进一步深入，可以检查每个关系的 `pg_class.relfrozenxid`。
- XID 是数字，可以使用 `age(...)` 函数将这些值与当前 XID 进行比较。

此外，还应检查 [MultiXact ID 回卷](https://www.postgresql.org/docs/current/routine-vacuuming.html#VACUUM-FOR-MULTIXACT-WRAPAROUND)的风险 (在监控工具中这通常会缺失)：

- `pg_database` 中的 `datminmxid`
- `pg_class` 中的 `relminmxid`
- 需要使用 `mxid_age(...)`，而不是 `age(...)`。

集群级别的高层次监控查询 (数据库级别) 示例：

```sql
with data as (
  select
    oid,
    datname,
    age(datfrozenxid) as xid_age,
    mxid_age(datminmxid) as mxid_age,
    pg_database_size(datname) as db_size
  from pg_database
)
select
  *,
  pg_size_pretty(db_size) as db_size_hr
from data
order by greatest(xid_age, mxid_age) desc;
```

特定数据库中的表级监控查询，TOP 25：

```sql
with data as (
  select
    format(
      '%I.%I',
      nspname,
      c.relname
    ) as table_name,
    greatest(age(c.relfrozenxid), age(t.relfrozenxid)) as xid_age,
    greatest(mxid_age(c.relminmxid), mxid_age(t.relminmxid)) as mxid_age,
    pg_table_size(c.oid) as table_size,
    pg_table_size(t.oid) as toast_size
  from pg_class as c
  join pg_namespace pn on pn.oid = c.relnamespace
  left join pg_class as t on c.reltoastrelid = t.oid
  where c.relkind in ('r', 'm')
)
select *
from data
order by greatest(xid_age, mxid_age) desc
limit 25;
```

## 告警

如果 XID 或 MultiXID 的年龄增长超过一定的阈值 (通常为 2 亿，参考 [autovacuum_freeze_max_age](https://postgresqlco.nf/doc/en/param/autovacuum_freeze_max_age/))，这表明某些东西阻碍了正常的 autovacuum 工作。

因此，根据 autovacuum 的设置，监控系统应配置为在 XID 和 MultiXID 年龄超过预定义阈值 (通常在 3 亿到 10 亿范围内) 时发出警报。年龄超过 10 亿应被视为危险信号，要求紧急缓解措施。