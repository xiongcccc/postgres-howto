# How to enable data checksums without downtime

## 数据校验和，基础知识

PostgreSQL 提供了[启用数据校验和](https://www.postgresql.org/docs/current/checksums.html)的功能，这是防止某些类型的数据损坏的有效手段 (但并不能防止所有损坏)。

需要注意的是，WAL 也有其自己的校验和，并且[始终开启](https://gitlab.com/postgres/postgres/blob/40d5e5981cc0fa81710dc2399b063a522c36fd68/src/backend/access/transam/xloginsert.c#L896)以验证 WAL 数据的完整性；在本文中，我们讨论的是针对表和索引页面的数据校验和。

数据校验和在默认情况下是禁用的，可以在集群初始化时通过执行 `initdb` 命令并带上 `--data-checksums` 选项来启用。

## 是否应该启用数据校验和？

根据[官方文档](https://www.postgresql.org/docs/current/app-initdb.html#APP-INITDB-DATA-CHECKSUMS:)：

> 启用校验和可能会导致显著的性能损耗。

然而，我强烈建议为所有集群启用数据校验和。如果担心性能损耗，建议进行测试。例如，一个[合成基准测试](https://gitlab.com/postgres-ai/postgresql-consulting/tests-and-benchmarks/-/issues/44)显示，启用校验和只会使 CPU 负载增加约 2%。即使这个开销稍高一些，考虑到及时检测存储级别数据损坏的重要性，我认为仍然值得启用。

## 如何检查现有集群是否启用了数据校验和

有三种方法可以检查现有集群是否启用了数据校验和：

1. 使用 SQL (Postgres 需要在线)：

```sql
SHOW data_checksums;
 data_checksums
----------------
 off
```

2. 使用 `pg_controldata` (Postgres 是否在线无关紧要)：

```bash
pg_controldata -D /path/to/data_directory | grep checksum
Data page checksum version:           0
```

`0` 表示未开启数据校验和。

3. 使用 `pg_checksums` (自 PostgreSQL 12 起提供；Postgres 必须关闭；注意，如果校验和已启用，该工具会扫描文件并检查校验和，因此你可能需要使用选项 `--progress` 来运行以查看进度)：

```bash
❯ pg_checksums -D /opt/homebrew/var/postgresql@15 --check
pg_checksums: error: data checksums are not enabled in cluster
```

## 如何为现有集群启用数据校验和

不幸的是，无法为运行中的服务器启用数据校验和。启用数据校验和有两种通用方法：

1. 集群重新初始化 (转储/恢复，或逻辑复制)
1. `pg_checksums` (服务器必须关闭)

自 PostgreSQL 12 起，Postgres 自带了 [pg_checksums](https://postgresql.org/docs/current/app-pgchecksums.html) 工具。对于旧版本 (9.3-11)，可以从[此处](https://github.com/credativ/pg_checksums)获取。

在执行 `pg_checksums` 时，如前所述，Postgres 必须关闭：

```bash
❯   brew services stop postgresql@15
Stopping `postgresql@15`... (might take a while)
==> Successfully stopped `postgresql@15` (label: homebrew.mxcl.postgresql@15)

❯ time pg_checksums -D /opt/homebrew/var/postgresql@15 --enable --progress
31035/31035 MB (100%) computed
Checksum operation completed
Files scanned:   3060
Blocks scanned:  3972581
Files written:  1564
Blocks written: 3711369
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster
pg_checksums -D /opt/homebrew/var/postgresql@15 --enable --progress  5.19s user 14.23s system 56% cpu 34.293 total
```

在这台 MacBook 上，启用数据校验和的速度约为 1 GiB/秒。请注意，`pg_checksums` 应在拥有数据目录的同一操作系统用户下执行。

检查结果：

```bash
❯ pg_controldata -D /opt/homebrew/var/postgresql@15 | grep checksum
Data page checksum version:           1

❯ pg_checksums -D /opt/homebrew/var/postgresql@15 --check --progress
31035/31035 MB (100%) computed
Checksum operation completed
Files scanned:   3060
Blocks scanned:  3972581
Bad checksums:  0
Data checksum version: 1
```

一旦完成，我们便可以重新启动 PostgreSQL 并再次检查：

```bash
❯ brew services start postgresql@15
==> Successfully started `postgresql@15` (label: homebrew.mxcl.postgresql@15)

❯ psql -Xc 'show data_checksums'
 data_checksums
----------------
 on
(1 row)
```

重要的是，要确保在 `pg_checksums --enable` 运行时 Postgres 不会启动。不幸的是，`pg_checksums` 在运行时不会检查 (它只在一开始时检查)。有一个很好的技巧可以避免意外启动 ([来源](https://www.crunchydata.com/blog/fun-with-pg_checksums)) — 暂时移动 Postgres 工作所需的一些核心文件或目录：

~~~bash
mv $PGDATA/pg_twophase $PGDATA/pg_twophase.DO_NOT_START_THIS_DATABASE
~~~

... 一旦 `pg_checksums` 工作完成，恢复：

~~~bash
mv $PGDATA/pg_twophase.DO_NOT_START_THIS_DATABASE $PGDATA/pg_twophase
~~~

## 如何在多节点集群中无停机启用数据校验和

幸运的是，如果集群中有副本，我们可以通过 switchover 以在不中断服务的情况下启用数据校验和。步骤如下：

1. 停止一个备库 (确保它在第 3 步之前不会启动！)。
2. 在备库上运行 `pg_checksums --enable`。
3. 重新启动备库并让其完全追上主库。
4. 对所有其他备库重复步骤 1-3。
5. 进行切换 (为了最小化停机时间，建议在切换之前执行一个显式 `CHECKPOINT`；如果使用了`pgBouncer`，建议使用它的 `PAUSE/RESUME` 功能实现**零停机**)。
6. 对前主库 (现在是备库) 执行步骤 1-3。

通过这种方法，可以成功地转换非常大的集群。执行此操作前，建议采取以下措施：

- 先在克隆/低级别环境中测试 `pg_checksums --enable`，并预估生产环境的两个值：执行 `pg_checksums`的持续时间 (秒) 和期间的累积延迟情况 (字节)。
- 选择低活动时间执行 (例如，夜间或周末，视工作负载类型而定)，以使累积的延迟更小。

虽然截至 2023年 (PostgreSQL 16) 尚不支持并行处理，但未来版本很可能会实现此功能。

以 1GiB/秒的转换速度为例，对于 1TiB 的集群，需要约 17 分钟。在拥有更强磁盘性能的机器上，速度应更快。