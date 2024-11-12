# Postgres major upgrade without any downtime for a very large cluster running under heavy load

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

目前，在 HN 首页上有一篇由 knock.app 撰写的关于[零停机升级 Postgres](https://news.ycombinator.com/item?id=38616181) 的文章。

本周晚些时候，GitLab 的 [Alexander Sosna](https://twitter.com/xxorde) 将会做一个[演讲](https://www.postgresql.eu/events/pgconfeu2023/schedule/session/4791-how-we-execute-postgresql-major-upgrades-at-gitlab-with-zero-downtime/)，解释 GitLab 的大型集群是如何在高负载下进行零停机升级的 — 我强烈推荐大家了解这项工作。

实现真正的零停机升级背后有许多细节和挑战需要解决，这里我只提供一个高层次的规划。该方法适用于非常大的 (数十 TB) 集群，具有多个副本，并在高 TPS (每秒数万次事务) 环境下运行。这里描述的过程涉及逻辑复制 (使用了"物理到逻辑"的技巧) 以及 PgBouncer 的 PAUSE/RESUME (假设使用了 PgBouncer)。

详细的材料需要编写几篇单独的使用指南。

该过程包括两个步骤：

1. **升级**：创建新的集群，运行在新的 Postgres 主版本上。
2. **切换**：逐步切换流量。

## 第一步：升级

一种"标准"方法是从头创建一个新集群 (`initdb`)，然后基于它创建一个全新的逻辑副本 (在创建逻辑订阅时使用 `copy_data = false`)。然而，这种逻辑副本配置方式需要大量时间，在高负载下执行可能会出现很大问题。

替代方法之一是获取一个物理副本并将其转换为逻辑副本。只要知道逻辑复制槽的 LSN 位置并使用 `recovery_target_lsn`，这便相对容易做到。这个方法被称为"physical2logical"转换，它允许基于物理副本 (例如，基于几分钟内的云快照创建) 快速可靠地创建逻辑副本。

然而，将 physical2logical 转换与 `pg_upgrade` 结合使用需要慎之又慎。详细信息可以在[这里](https://postgresql.org/message-id/flat/20230217075433.u5mjly4d5cr4hcfe%40jrouhaud)找到。

步骤：

1. 创建一个新集群，它将是使用级联物理复制的辅助集群 (Patroni 支持)。
2. 确保新集群的主节点没有显著滞后于旧集群的主节点。
3. 以适当的顺序停止新集群的节点 (确保停止的副本与其主节点完全同步)。
4. 在旧集群的主节点上，创建一个逻辑复制槽并为所有表创建发布 (这不需要表级锁，因此我们不需要设置较低的 `lock_timeout` 并进行重试)，并记下复制槽的位置。
5. 重新配置新集群的主节点，设置 `recovery_target_lsn` 为所记录的复制槽的 LSN，并禁用 `restore_command`。
6. 启动新集群的主节点和副本，让它们达到所需的 LSN。然后，便可以提升主节点了。
7. 再次以适当的顺序停止新集群的节点。
8. 在新集群的主节点上运行 `pg_upgrade --link`。
9. 对新集群的副本使用 `rsync --hard-links --size-only` — 这是一个有争议的步骤 (详见[此处](https://www.postgresql.org/message-id/flat/CAM527d8heqkjG5VrvjU3Xjsqxg41ufUyabD9QZccdAxnpbRH-Q%40mail.gmail.com))，但这是大多数人在使用 `pg_upgrade --link` 进行就地 (无需逻辑复制) 升级时采用的方法，目前还没有其他快速的替代方案。
10. 配置新集群的主节点 (现在已经是主节点) 以使用逻辑复制 — 创建订阅，设置 `copy_data = false`，让它追赶上正在运行的旧集群。

在所有这些步骤中，旧集群一直在运行，而新集群对用户是不可见的。这使得你能够在生产环境中测试整个过程 (在较低环境中进行适当测试后)，这可以带来巨大的好处。

## 第二步：切换

首先，切换只读 (RO) 流量是有意义的。如果应用程序允许，首先将部分只读流量重定向到新的副本是有意义的。这需要在负载均衡代码中实现高级的复制滞后检测（参照：[Day 17: How to determine the replication lag - Hybrid case: logical & physical](https://postgres-howto.cn/#/./docs/17))。

当需要重定向读写 (RW) 流量时，为了实现零停机，可以使用 PgBouncer 的 PAUSE/RESUME 功能。如果有多个 PgBouncer 节点 (运行在不同的主机/端口上，或涉及 `SO_REUSEPORT`)，实现一个优雅的 PAUSE 获取方法非常重要。一旦所有的 PAUSE 都被获取，就可以重定向流量，然后发出 RESUME。在此之前，重要的是确保所有写操作都已完全传播到新的主节点。

重要的是要制定措施来防止来自应用程序或用户的写操作影响旧集群 — 例如，调整 `pg_hba.conf` 或关闭集群。然而，为了实现高级的回滚功能，有必要以与"正向"相同的方式实现"反向"逻辑复制，并在切换期间设置。在这种情况下，对旧集群的写操作是允许的 — 但只能来自逻辑复制。反向复制允许即使在整个过程完成后的一段时间内进行回滚，这使整个过程在任何时候都可以完全逆转。

如前所述，还有许多方面需要解决，这只是一个高级别的计划。如果有机会参加这个演讲，请在[这里](https://www.postgresql.eu/events/pgconfeu2023/schedule/session/4791-how-we-execute-postgresql-major-upgrades-at-gitlab-with-zero-downtime/)参加。

## 我见

参照之前的文章 [How to convert a physical replica to logical](https://postgres-howto.cn/#/./docs/57?id=how-to-convert-a-physical-replica-to-logical)，另外推荐去阅读一下这篇 PDF：https://www.postgresql.eu/events/pgconfeu2023/schedule/session/4791-how-we-execute-postgresql-major-upgrades-at-gitlab-with-zero-downtime/，针对 logical replication + pg_upgrade 的升级方式进行了优化。

