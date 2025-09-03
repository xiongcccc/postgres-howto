# How to run ANALYZE (to collect statistics)

`ANALYZE` 命令会收集数据库的统计信息 ([文档](https://www.postgresql.org/docs/current/sql-analyze.html))。维护统计信息对于获得良好的数据库性能至关重要。

运行方式非常简单：

~~~sql
analyze;
~~~

但是，由于它是**单线程**的，可能会花费很长时间。

## 如何全速运行 ANALYZE

为了利用多核 CPU，可以使用客户端工具 `vacuumdb`，加上 `--analyze-only` 选项以及多个工作进程 ([文档](https://www.postgresql.org/docs/current/app-vacuumdb.html))。

下面的示例会在所有数据库上运行 ANALYZE (`--all`；注意某些托管版 Postgres，如 RDS，可能不支持)，使用的并发数与 vCPU 数相同，并将总时长限制为 2 小时 (此处未展示 `-h, -U` 等连接参数)：

~~~shell
{  
    while IFS= read -r line; do  
        echo "$(date '+%Y-%m-%d %H:%M:%S') $line"  
    done  
} < <(
  PGOPTIONS='-c statement_timeout=2h' \
    vacuumdb \
      --analyze-only \
      --all \
      --jobs $JOBS \
      --echo
) | tee -a analyze_all_$(date +%Y%m%d).log
~~~

这个脚本会把执行的命令带上时间戳打印并记录日志 (也可以使用 `moreutils` 提供的 `ts` 来替代 `while` 循环)。

`$JOBS` 的值应根据服务器的 vCPU 数量来选择。例如，如果想要全速运行，通常设置为和 vCPU 数量一致。客户端机器的性能只需要"够用"即可 (有必要检查客户端机器的 CPU 和磁盘 IO 饱和度，以确保它们不是瓶颈)。需要注意的是：如果存在未分区的大表，某个时间点可能只剩下少数 worker 在运行，导致效率不高。一个解决方案是分区：把大表拆分为多个较小的分区，可以保持所有工作进程都繁忙，从而在多核机器上显著加速整个过程。

强烈建议在靠近数据库服务器的可靠的客户端机器上运行，或者直接在服务器本机上运行，并放在 `tmux` 会话中，这样就不会受外部网络中断影响。

重要提醒：对于分区表，`vacuumdb --analyze-only` 不会更新分区表 (父表) 的统计信息，它只更新分区 (子表) ([讨论链接](https://www.postgresql.org/message-id/ZyQgY_ErJszSZTNq%40momjian.us))。好消息是，这个问题有望在 PostgreSQL 19 中得到修复 ([commitfest 条目](https://commitfest.postgresql.org/patch/5871/))。在此之前，如果要对分区表并行收集统计信息：可以使用单线程的 `ANALYZE` (它会逐个处理分区，并且也会更新父表)，或者需要采用其它并行化方案 (TODO)。

## 开销分析

在某些情况下，这个过程可能会非常耗时并占用大量资源 (CPU、磁盘)：

- 数据库规模很大；
- 表数量非常多；
- `default_statistics_target` 参数从默认的 100 提高到较高值(例如 1000)。

## 批量数据导入后的 ANALYZE

在表进行初始数据导入，或表中数据发生重大变化后，必须运行 `ANALYZE`。有两点需要额外注意：

1. 理论上这是 autovacuum 的工作，但根据配置和工作进程的数量，触发 autovacuum 可能需要很长时间。在此期间，数据库性能可能处于次优状态。
2. 在这种场景下，运行 `VACUUM` 也非常有必要 — 可以直接运行：`VACUUM VERBOSE ANALYZE <tablename>;` 或者使用 `vacuumdb` 并带上 `--analyze` (没有后缀 `-only`，就会同时执行 `VACUUM` 和 `ANALYZE`)。

## 大版本升级

在大版本升级之后，运行 `ANALYZE` 是一个必须的步骤，但目前并不会自动执行 (截至 2024 年)：

- `pg_upgrade` 不会自动运行；
- 主要的云厂商 (AWS RDS、GCP CloudSQL、Azure) 也不会自动运行。

因此，很多人在升级后忘记这一步，结果在升级后的第一个业务高峰日(通常是周一)，就遇到严重的数据库事故。

强烈建议将这一步纳入自动化流程。

## 关于 --analyze-in-stages

`--analyze-in-stages` 选项会分三阶段运行 ANALYZE：第一阶段使用最小的 `default_statistics_target`，尽快生成可用的统计信息；后续阶段逐步构建完整的统计信息 ([文档](https://www.postgresql.org/docs/current/app-vacuumdb.html))。

在 OLTP 场景 (如 Web、移动应用) 下，通常意义不大。考虑以下两种情况：

1. 就地升级。在这种情况下，通常会预留几分钟的停机窗口来运行 `pg_upgrade --link`。在大多数情况下，使用 `vacuumdb` 多线程执行完整的 `ANALYZE`，并不会让停机窗口显著延长。如果在统计信息尚未生成的情况下过早开放业务流量，往往会导致性能不佳，甚至出现意外宕机，因为缺乏合适的统计信息对性能影响极大。试图依靠"不完整"的统计信息就开始处理业务流量，容易引发事故。因此，推荐在维护窗口中，在运行完 `pg_upgrade` 之后，立即执行一次完整的 `ANALYZE`，并尽可能实现自动化。当然，了解耗时非常有帮助 — 为了预测整体操作时长，最好在最大规模的集群上做一次克隆测试，记录整体耗时。
2. 如果是依赖逻辑复制的零停机升级，则完全不需要 `--analyze-in-stages`。此时，只需在目标集群的主库上，在逻辑复制运行期间执行 `ANALYZE` 即可。