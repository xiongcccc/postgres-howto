# How to check btree indexes for corruption (pg_amcheck)

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

🥳🤩🎂🎉 今天是我的生日，而且我们刚刚软启动了我们的 [postgres.ai bot](https://twitter.com/samokhvalov/status/1726177412755677283)  — 所以请原谅这次文章并不完整。不过，我仍然继续发布 😅

有多种类型的数据损坏。某些类型的数据损坏可以通过扩展 `amcheck` 来识别 (参见：[Using amcheck Effectively](https://postgresql.org/docs/current/amcheck.html#AMCHECK-USING-AMCHECK-EFFECTIVELY))，该扩展包含在标准的"contrib 模块"包中 — 只需创建即可：

```sql
create extension amcheck;
```

对于 Postgres 14+，建议使用 CLI 工具 [pg_amcheck](https://postgresql.org/docs/current/app-pgamcheck.html)，其优势之一是`-j `(`--jobs=NUM`) 选项 — 可以通过并行工作进程更快地检查多个索引。

以下是一个使用示例 (填入连接选项并调整并行工作进程数)：

```bash
pg_amcheck \
    {{ connection options }} \
    --install-missing \
    --jobs=8 \
    --verbose \
    --progress \
    --heapallindexed \
    --parent-check \
  2>&1 \
  | ts \
  | tee -a pg_amcheck.$(date "+%F-%H-%M").log
```

**重要提示**：选项 `--heapallindexed` 和 `--parent-check` 会触发一个耗时但更高级的检查。
`--parent-check` 选项会阻塞写操作 (如 `UPDATE` 等)，因此不要在接收用户流量的生产节点上使用它。
选项 `--heapallindexed` 会增加检查的负载和持续时间，但可以小心地在生产环境中使用。
在没有这两个选项的情况下，执行的检查会较轻，可能无法发现某些问题 (参见：[Using amcheck Effectively](https://postgresql.org/docs/current/amcheck.html#AMCHECK-USING-AMCHECK-EFFECTIVELY))。

一旦上述命令执行完成，检查生成的日志中是否包含错误：

```bash
egrep 'ERROR:|DETAIL:|LOCATION:' \
 pg_amcheck.$(date "+%F-%H-%M").log
```

如果检测到有损坏的索引，需要仔细分析，一旦问题得到确认，则需要重新创建索引 (使用`REINDEX CONCURRENTLY`)。