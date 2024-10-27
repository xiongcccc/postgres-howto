# How to help others

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

帮助他人 (无论是对团队成员的内部帮助还是外部咨询) 有两个关键原则：

1. 依赖可靠的资源。
2. 测试一切。

## 原则1：可靠的资源

依赖你所信任的高质量资源。例如，我的观点是，根据质量/可信度等级来排名，大致如下：

- PostgreSQL文档 – 9/10
- PostgreSQL源码 – 10/10 (真理的来源！)
- StackOverflow上的随机答案 – 5/10或更低（但不包括 [Erwin Brandstetter](https://stackoverflow.com/users/939860/erwin-brandstetter), [Laurenz Albe](https://stackoverflow.com/users/6464308/laurenz-albe), [Peter Eisentraut](https://stackoverflow.com/users/98530/peter-eisentraut) 的回答 — 他们回答得很棒，8/10 或更高)。
- 随机博客文章，同样 5/10 或更低。
- [Suzuki](https://www.interdb.jp/pg/) 和 [Rogov](https://postgrespro.com/blog/pgsql/5969637) 的 Internals 两本书 – 8/10 或更高

另外 (非常重要！)，始终提供你参考资源的链接；这样做有两个好处：

- 推广好的资源，回馈它们；
- 在某种程度上分担责任 (如果你经验不足，这非常有帮助；每个人都有可能犯错)。

## 原则2：验证 — 数据库实验

始终对所有事物保持怀疑，不要轻信语言模型，无论它们的性质如何。

如果有人 (包括你或我) 在没有通过实验 (测试) 验证的情况下说了某件事，就需要通过实验进行验证。

所有的决策都应该基于数据，而可靠的数据是通过测试收集的。

大多数与数据库相关的想法都需要通过数据库实验来验证。

两种类型的数据库实验：

1. 多会话实验，成熟的基准测试，例如使用 `pgbench`、`JMeter`、`sysbench`、`pgreplay-go` 等工具进行的基准测试。

   它们的目的是研究 Postgres 的整体行为及其所有组件，必须在专用资源上进行 (在这里，其他人不应有任何工作)。环境应与生产环境相匹配 (虚拟机和磁盘类型、PostgreSQL 的版本、相关设置等)。

   此类实验包括负载测试、压力测试、性能回归测试等。主要工具是用于研究宏观级别查询分析的工具：`pg_stat_statements`、等待事件分析 (即历史活跃会话或性能/查询洞察)、`auto_explain`、`pgBadger` 等等。

   关于此类实验的更多信息：详见 [Day 13: How to benchmark](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0013_how_to_benchmark.md).

2. 单会话实验，使用单个会话 (有时两个) 测试一个或一系列 SQL 查询，以检查查询语法、研究单个查询行为、优化特定查询等。

   这些实验可以在共享环境中进行，使用较弱的机器也可以。但是，要研究查询性能，你需要使用相同的 PostgreSQL 版本、相同或类似的数据库，以及匹配的规划器设置 (如何做到这一点：参照[Day 56: How to make the non-production Postgres planner behave like in production](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0056_how_to_imitate_production_planner.md))。需要注意的是，计时指标可能与生产环境相比显著偏差，在查询优化过程中主要应关注执行计划和数据量 (`EXPLAIN` 中的实际行数、`BUFFERS` 选项提供的缓冲区操作计数)。

   此类实验的示例：检查 SQL 查询序列的语法和逻辑行为，查询性能分析 `EXPLAIN (ANALYZE, BUFFERS)`，测试优化想法，架构更改测试等。快速克隆大型数据库非常有用，而且不需要为存储付额外的钱 (例如 Neon、Amazon Aurora)。如果存储和计算都不需要额外费用，那么测试活动，包括 CI/CD 中的自动化测试，就真正得到了释放 (DBLab Engine [@Database_Lab](https://twitter.com/Database_Lab))。

## 总结

如你所见，原则非常简单：

1. 阅读优质的文章；
2. 不要盲目相信 — 测试一切。