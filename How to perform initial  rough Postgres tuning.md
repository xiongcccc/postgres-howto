# How to perform initial / rough Postgres tuning

现代 PostgreSQL 提供了超过 300 个设置 (即所谓的 GUC 变量，"grand unified configuration")。为特定环境、数据库和工作负载对 Postgres 进行精细调优是一项非常复杂的任务。

但在大多数情况下，帕累托法则 (即二八法则) 非常有效：你只需要投入有限的精力处理基本的调优领域，然后专注于查询性能。这个方法背后的逻辑很简单：你可能会花费大量时间为 `shared_buffers` 找到比传统的 25% 更合适的值 (很多人认为 25% 并不理想，比如  [Andres Freund's Tweet](https://twitter.com/andresfreundtec/status/1178765225895399424))，但你可能会发现某些查询由于性能不佳 (例如缺少适当的索引) 而破坏了所有调优的正面效果。

因此，我建议采用以下方法：

- 基本的"粗略"调优
- 日志相关设置
- 自动清理调优
- 检查点调优
- 然后专注于查询优化，无论是被动的还是主动的，且仅在有充分理由时对特定领域进行精细调优

## 基本的粗略调优

对于初步的粗略调优，遵循二八法则的经验工具在大多数情况下已经"足够好" (实际上，可能在这个情况下是 95/5 的法则)：

- [PGTune](https://pgtune.leopard.in.ua) ([source code](https://github.com/le0pard/pgtune))
- [PostgreSQL Configurator](https://pgconfigurator.cybertec.at)
- 对于 TimescaleDB 用户： [timescaledb-tune](https://github.com/timescale/timescaledb-tune)

此外，除了官方文档外，[这个资源](https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server)也是一个很好的参考 (它整合了来自多个来源的信息，不仅仅是官方文档)。例如，检查页面 [random_page_cost](https://postgresqlco.nf/doc/en/param/random_page_cost/)，这是一个经常被忽略的参数。

如果你使用的是托管 Postgres 服务 (如 RDS)，通常在你创建服务器时这个层面的调优已经完成了。但仍然值得再次检查，例如，有些服务提供商会使用 SSD 磁盘配置服务器，但仍然将 `random_page_cost` 保持为默认值 4，这个值适用于磁盘驱动器。如果你使用 SSD，将其设置为 1。

在进行查询优化之前，进行这个层面的调优非常重要，否则在调整基本配置后，可能需要重新进行查询优化。

## 日志相关设置

此处的通用规则是：日志记录越多越好。当然，前提是避免两种饱和情况：

- 磁盘空间 (日志撑爆磁盘)
- 磁盘 IO (日志写入次数过多导致的磁盘负载)

简而言之，我的建议如下 (这些内容值得另写一篇详细的文章)：

- 打开检查点日志：`log_checkpoints='on'` (幸运的是，在 PG15+ 中默认已经开启)
- 打开所有自动清理日志：`log_autovacuum_min_duration=0` (或一个非常小的值)
- 记录临时文件，除了非常小的文件 (例如，`log_temp_files = 100`)
- 记录所有 DDL 语句：`log_statement='ddl'`
- 调整 `log_line_prefix`
- 设置一个较低的 `log_min_duration_statement` (例如 500ms)，或使用 `auto_explain` 记录慢查询及其执行计划

## 自动清理调优

这是一个需要单独讨论的大话题。简而言之，默认设置不适用于任何现代 OLTP 场景 (例如 Web/移动应用)，因此自动清理功能必须进行调优。如果不调优，自动清理将把大量死元组"转换"为膨胀，最终会对性能产生负面影响。

需要解决两个调优方向：

1. 提高处理频率：通过降低 `*_scale_factor` 和 `*_threshold` 设置，让自动清理在累积较少死元组时就开始处理表。

2. 分配更多资源进行处理：更多的自动清理工作进程 (`autovacuum_workers`)、更多的内存 (`autovacuum_work_mem`) 以及更高的"配额" (通过 `*_cost_limit` 和 `*_cost_delay` 控制)。

## 检查点调优

同样，这个话题值得单独讨论。但简而言之，你需要考虑提高 `checkpoint_timeout`，最重要的是增加 `max_wal_size` (默认值对于现代机器和数据量来说非常小，仅为 1GB)，以减少检查点的频率，尤其是在大量写操作发生时。然而，朝这个方向改变设置意味着在发生崩溃或从备份中恢复时需要更长的恢复时间 — 这是一种需要针对特定情况进行分析的权衡。

就这样，PostgreSQL 配置的初步/粗略调优一般不会花费很长时间。对于某一类集群或特定集群来说，工程师大约只需要 1-2 天的工作。你并不需要 AI 来完成这项任务，经验工具工作得很好 — 除非你想要再提高5-10% (例如，如果你有成千上万的服务器)。