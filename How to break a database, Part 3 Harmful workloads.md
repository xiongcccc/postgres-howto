# How to break a database, Part 3: Harmful workloads

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

参照

- [Part 1: How to Corrupt]().
- [Part 2: Simulate infamous transaction ID wraparound]().

## 过多的连接

下面是一个简单的代码片段，通过 `psql` 和命名管道 (即 FIFO，在 macOS 和 Linux 中都适用) 创建 100 个空闲连接：

```bash
mkfifo dummy

for i in $(seq 100); do
  psql -Xf dummy >/dev/null 2>&1 &
done

❯ psql -Xc 'select count(*) from pg_stat_activity'
  count
-------
  106
(1 row)
```

要关闭这些连接，我们可以打开一个写入文件描述符到 FIFO 然后关闭它而不写入任何数据：

```bash
exec 3>dummy && exec 3>&-
```

现在，100 个额外的连接消失了：

```bash
❯ psql -Xc 'select count(*) from pg_stat_activity'
  count
 -------
      6
 (1 row)
```

如果连接数达到了 `max_connections` 限制，当我们执行上述步骤时，尝试建立新连接时将看到以下错误：

```bash
❯ psql
psql: error: connection to server on socket "/tmp/.s.PGSQL.5432" failed: FATAL:  sorry, too many clients already
```

## 闲置事务会话

这个方法我们在模拟事务 ID 回卷实验中使用过：

```bash
mkfifo dummy

psql -Xc "
 set idle_in_transaction_session_timeout = 0;
 begin;
 select pg_current_xact_id()
 " \
-f dummy &
```

要释放会话：

```bash
exec 3>dummy && exec 3>&-
```

## 使用各种工具造成更多损害

这个工具可以帮助你模拟各种有害工作负载：[noisia – harmful workload generator for PostgreSQL](https://github.com/lesovsky/noisia)

截至 2023 年，它支持以下功能：

- 空闲事务 — 在频繁写的表上处于活动状态但不执行任何操作的事务。
- 回滚 — 伪造无效查询，生成错误并增加回滚计数。
- 等待事务 — 锁定频繁写的表然后闲置的事务，导致其他事务被阻塞。
- 死锁 — 同时进行的事务，每个事务持有其他事务所需的锁。
- 临时文件 — 由于 `work_mem` 不足，查询产生的临时文件。
- 终止后台进程 — 使用 `pg_terminate_backend()` 或 `pg_cancel_backend()` 终止随机的后台进程或查询。
- 连接失败 — 耗尽所有可用连接 (其他客户端无法连接到 Postgres)。
- fork 连接 — 在专用连接中执行单个、简短的查询 (导致 Postgres 后台进程的过度 fork)。

另一个工具 `pg_crash` 会定期崩溃你的数据库。

对于Aurora用户，还有一些有趣的函数：`aurora_inject_crash()`、`aurora_inject_replica_failure()`、`aurora_inject_disk_failure()`、`aurora_inject_disk_congestion()`：参见 [Testing Amazon Aurora PostgreSQL by using fault injection queries](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.Managing.FaultInjectionQueries.html)。

## 总结

混沌工程是一个非常有趣的话题，我认为它在数据库领域有着巨大的潜力 — 用于测试恢复、故障转移，并实践各种事故情况。一些资源 (不仅限于数据库)：

- Wikipedia: [Chaos engineering](https://en.wikipedia.org/wiki/Chaos_engineering)
- Netflix 的  [Chaos Monkey](https://github.com/Netflix/chaosmonkey) 是一款弹性工具，可帮助应用程序容忍随机实例故障

理想情况下，成熟的数据库管理流程，无论是在云端或本地，托管或自托管，都应该包括：

- 定期在非生产环境中模拟事故，以实践并改进事故缓解的手册。
- 定期在生产环境中发起事故，以查看自动化缓解措施的实际效果。例如：自动移除崩溃的副本、自动故障转移、针对长时间运行事务的告警以及团队的响应。