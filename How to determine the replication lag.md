# How to determine the replication lag

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

## 在主节点/备节点领导节点上

当连接到主节点 (或级联复制情况下的备节点的领导节点) 时，你可以使用 `pg_stat_replication`：

```sql
nik=# \d pg_stat_replication
                    View "pg_catalog.pg_stat_replication"
      Column      |           Type           | Collation | Nullable | Default
------------------+--------------------------+-----------+----------+---------
 pid              | integer                  |           |          |
 usesysid         | oid                      |           |          |
 usename          | name                     |           |          |
 application_name | text                     |           |          |
 client_addr      | inet                     |           |          |
 client_hostname  | text                     |           |          |
 client_port      | integer                  |           |          |
 backend_start    | timestamp with time zone |           |          |
 backend_xmin     | xid                      |           |          |
 state            | text                     |           |          |
 sent_lsn         | pg_lsn                   |           |          |
 write_lsn        | pg_lsn                   |           |          |
 flush_lsn        | pg_lsn                   |           |          |
 replay_lsn       | pg_lsn                   |           |          |
 write_lag        | interval                 |           |          |
 flush_lag        | interval                 |           |          |
 replay_lag       | interval                 |           |          |
 sync_priority    | integer                  |           |          |
 sync_state       | text                     |           |          |
 reply_time       | timestamp with time zone |           |          |
```

此视图包含物理复制 (仅限于使用流复制的情况，非 WAL 传输方式) 和逻辑复制的信息。延迟情况在此视图中可以通过字节 (lsn 相关列) 或时间间隔 (lag 相关列) 进行衡量，并且可以为每个复制流观察到多个步骤。

要分析此视图中的 LSN 值，我们需要将它们与所连接服务器上的当前 `LSN` 值进行比较：

- 如果是主服务器 (`pg_is_in_recovery()` 返回 false )，则使用 `pg_current_wal_lsn()`
- 否则 (备节点领导节点)，使用 `pg_last_wal_replay_lsn()`

可以使用 `pg_wal_lsn_diff()` 函数或 `-` 操作符进行 LSN 的差值计算。 

文档：[Monitoring pg_stat_replication view](https://postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-REPLICATION-VIEW).

以下是一些查询延迟的示例：

- [Netdata](https://github.com/netdata/netdata/blob/9edcad92d83dba359f5eb6c06d0741b50030edcf/collectors/python.d.plugin/postgres/postgres.chart.py#L486)
- [Postgres exporter for Prometheus](https://github.com/prometheus-community/postgres_exporter/blob/2a5692c0283fddf96e776cc73c2fc0d5caed1af6/cmd/postgres_exporter/queries.go#L46)

## 在物理备节点上

当连接到物理备节点时，获取其延迟时间间隔：

~~~sql
select now() - pg_last_xact_replay_timestamp();
~~~

在某些情况下，`pg_last_xact_replay_timestamp()` 可能返回 `NULL`：

- 如果备节点刚刚启动并且尚未重放任何事务；
- 如果主节点上没有最近的事务。

`pg_last_xact_replay_timestamp()` 的这种行为可能会导致错误的结论，认为备节点存在延迟，且复制不健康 — 这在低负载的环境 (例如非生产环境) 中并不罕见。

文档：[pg_last_xact_replay_timestamp](https://postgresql.org/docs/current/functions-admin.html#id-1.5.8.33.6.3.2.2.4.1.1.1).

------

## 逻辑复制

在主节点 (在逻辑复制上下文中通常称为"发布端") 上可以观察到复制延迟。除了前面讨论的 `pg_stat_replication`，还可以使用 `pg_replication_slots`：

```sql
select
    slot_name,
    pg_current_wal_lsn() - confirmed_flush_lsn as lag_bytes
from pg_replication_slots;
```

------

## 混合情况：逻辑和物理复制

在某些情况下，你可能需要同时处理逻辑复制和物理复制。例如，考虑以下情况：

- 集群 A：一个由 3 个节点 (主节点 + 2 个物理备节点) 组成的常规集群。
- 集群 B：也是主节点 + 2 个物理备节点，且主节点通过逻辑复制连接到集群 A 的主节点。

这是一种典型的情况，通常在涉及逻辑复制的复杂更改 (例如大版本升级) 时会发生。在某些时候，你可能希望将部分只读流量从集群 A 的备节点重定向到集群 B 的备节点。

在这种情况下，如果我们需要了解每个节点的延迟情况，并且希望代码能很好地适用于所有节点，这可能会有些棘手。

以下是解决方法 (感谢来自 GitLab 的 [Dylan Griffith](https://gitlab.com/gitlab-org/gitlab/-/merge_requests/121621))，假设我们可以从我们正在分析的节点和集群 A 的主节点获取信息 (记住上面的所有评论)：

首先，获取主节点的 LSN：

```sql
select pg_current_wal_lsn() as primary_lsn;
```

然后获取观察节点的 LSN 位置，并使用它来计算以字节为单位的延迟值：

```sql
with current_node as (
  select case
    when exists (select from pg_replication_origin_status) then (
      select remote_lsn
      from pg_replication_origin_status
    )
    when pg_is_in_recovery() then pg_last_wal_replay_lsn()
    else pg_current_wal_lsn()
  end as lsn
)
select lsn – {{primary_lsn}} as lag_bytes
from current_node;
```