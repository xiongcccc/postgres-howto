# Rough configuration tuning (80/20 rule; OLTP)

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

二八法则 (也称为[帕累托法则](https://en.wikipedia.org/wiki/Pareto_principle)) 通常足以为 OLTP 负载提供良好的性能。在大多数情况下，建议采用这种方法并专注于查询调优。尤其是在系统快速变化时，单纯依赖配置调优来争取"最后的 20%"，往往不如通过忽视的模式级优化 (例如缺失索引) 来提升性能有意义。然而，如果工作负载和数据库没有太快变化，或者有很多 (比如几千个) Postgres 节点时，争取这"20%”是很有意义的 (例如，预算方面)。

这就是为什么像 [PGTune](https://pgtune.leopard.in.ua) 这样的简易经验性调优工具通常已经足够的原因。此处让我们考虑一个例子：一台中等规模的服务器 (64 vCPU，512 GiB RAM)，处理中等规模的 OLTP (比如 Web/移动应用)工作负载。

以下设置应作为起点，数值仅为粗略的参考 — 请根据具体情况进行审查，通过非生产环境中的实验验证，并密切监控所有变化。

推荐资源：

- [PGTune](https://pgtune.leopard.in.ua)
- [postgresql.conf configurations](https://postgresqlco.nf)
- [postgresql_cluster's defaults](https://github.com/vitabaks/postgresql_cluster/blob/master/vars/main.yml)

1. `max_connections = 200`

   根据预期的并发连接数来设置。此处假设我们使用了连接池，减少维持大量空闲连接的需求。如果使用PG15+，可以设置更高的值 (参照：[Improving Postgres Connection Scalability: Snapshots](https://techcommunity.microsoft.com/t5/azure-database-for-postgresql/improving-postgres-connection-scalability-snapshots/ba-p/1806462#conclusion-one-bottleneck-down-in-pg-14-others-in-sight))。

2. `shared_buffers = 128GB`

   通常情况下，建议将其设置为总内存的 25%。这是 Postgres 缓存表和索引数据的地方。

   25% 是最常见的设置，尽管有时有人批评它不是最优的，但在大多数情况下是"足够好"且非常安全的。

3. `effective_cache_size = 384GB`

   提供给查询规划器的内存建议，表示可用于缓存数据的内存量，包括操作系统缓存。

4. `maintenance_work_mem = 2GB`

   提高维护性操作 (比如 `VACUUM`、`CREATE INDEX` 等) 的性能。

5. `checkpoint_completion_target = 0.9`

   控制 checkpoint 的完成目标，通过分散写入活动来减少 IO 峰值。

6. `random_page_cost = 1.1`

   微调此项以反映随机 IO 的实际成本。默认值是 4，`seq_page_cost` 是 1 — 对于旋转硬盘来说是可以接受的。对于 SSD 来说，使用相等或接近的值是有意义的 (Crunchy Data 最近的[基准测试](https://docs.crunchybridge.com/changelog#postgres_random_page_cost_1_1)表明，1.1 比 1 稍微好一些)。

7. `effective_io_concurrency = 200`

   对于 SSD 来说，可以设置比 HDD 更高的值，反映其处理更多 IO 操作的能力。

8. `work_mem = 100MB`

   每个查询的排序和连接操作所用的内存。设置时要小心，因为如果同时运行太多查询，过高的值可能会导致 OOM 的问题。

9. `huge_pages = try`

   使用大页内存可以通过减少页管理开销来提高性能。

10. `max_wal_size = 10GB`

    这是 checkpoint 调优的一部分。10GB 是一个相对较大的值，尽管有些人可能更倾向于使用更大的值，但这也带来了一个权衡：

    - 更大的值有助于更好地处理大量写入 (IO 压力更低)

    - 但同时也会导致崩溃后恢复时间更长。

      > 🎯 TODO：关于 checkpoint 调优的独立指南。

11. `max_worker_processes = 64`

    数据库集群可以使用的最大进程数。对应 CPU 核心数。

12. `max_parallel_workers_per_gather = 4`

    每个 Gather 或 Gather Merge 节点最多可以启动的工作进程数。

13. `max_parallel_workers = 64`

    可用于并行操作的工作进程总数。

14. `max_parallel_maintenance_workers = 4`

    控制并行维护任务 (比如创建索引) 的工作进程数。

15. `jit = off`

    对于 OLTP 工作负载，建议关闭 JIT 编译。

16. 超时设置

> 🎯 **TODO:** 一篇独立指南

~~~sql
statement_timeout = 30s
idle_in_transaction_session_timeout = 30s
~~~

17. Autovacuum调优

> 🎯 **TODO:** 一篇独立指南

~~~bash
autovacuum_max_workers = 16
autovacuum_vacuum_scale_factor = 0.01
autovacuum_analyze_scale_factor = 0.01
autovacuum_vacuum_insert_scale_factor = 0.02
autovacuum_naptime = 1s
# autovacuum_vacuum_cost_limit – increase if disks are powerful
autovacuum_vacuum_cost_delay = 2
~~~

18. 可观测性与日志

> 🎯 **TODO:** 一篇独立指南

~~~bash
logging_collector = on
log_checkpoints = on
log_min_duration_statement = 500ms # review
log_statement = ddl
log_autovacuum_min_duration = 0 # review
log_temp_files = 0 # review
log_lock_waits = on
log_line_prefix = %m [%p, %x]: [%l-1] user=%u,db=%d,app=%a,client=%h
log_recovery_conflict_waits = on 
track_io_timing = on # review
track_functions = all
track_activity_query_size = 8192
~~~