# How to check btree indexes for corruption

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

以下代码片段使用了 `$N` 个并行工作进程检查所有 Btree 索引 (一般推荐进行并行处理)：

```bash
pg_amcheck \
  --install-missing \
  --verbose \
  --progress \
  --jobs=$N \
  --heapallindexed \
  2>&1 \
  | ts \
  | tee -a amcheck_$(date "+%F-%H-%M").log
```

文档参考：

- [amcheck (extension)](https://postgresql.org/docs/current/amcheck.html)
- [pg_amcheck (CLI tool)](https://postgresql.org/docs/current/app-pgamcheck.html)

备注：

1. 此类检查只需获取索引的`AccessShareLock` — 因此 DML 操作不会被阻塞。不过，通常这种检查会产生较大的工作负载，因为它需要读取所有索引以及相应元组。
2. `--heapallindexed` 选项是可选的，但强烈推荐使用。如果不使用此选项，检查的时间会大幅缩短，但在这种情况下，仅进行"轻量级"索引检查 (不会涉及到表，不会跟随索引引用检查表中的元组)。
3. 这里还有一个很有用但没有使用的选项 `--parent-check`，它提供了更全面的索引结构检查 — 会检查索引中的父子关系。这对于索引的"深度"测试非常有帮助。不过，这个检查速度较慢，并且不幸的是，需要获取`ShareLock`锁。这意味着在这种模式下进行检查时，会阻塞修改操作(`INSERT`/`UPDATE`/`DELETE`)。因此，`--parent-check` 只能在不接收流量的克隆数据库或维护窗口期间使用。
4. 使用 `amcheck` 检查索引并不是启用数据校验和的替代方案。建议同时启用数据校验和并定期使用 `amcheck` 进行损坏检查 (例如，在创建副本之后、备份恢复验证时，或者任何时候怀疑有索引损坏)。
5. 如果发现错误，意味着有损坏 (除非 `amcheck` 有 bug)。如果没有发现错误，这并不意味着没有损坏 — 损坏问题的范围非常之广，没有任何单一工具可以检查所有可能的损坏类型。
6. 可能导致 Btree 索引损坏的情况：
   1. Postgres 的 bug (例如，14.0-14.3 版本中的 `REINDEX CONCURRENTLY` 存在 bug，可能会导致 Btree 索引损坏）。
   2. 将 `PGDATA` 从一个操作系统版本复制到另一个操作系统版本时，默默地切换到不同的 glibc 版本，导致排序规则发生变化。在这种情况下，PG15+ 会发出警告 (`WARNING: database XXX has a collation version mismatch`)，而旧版本则不会发出警告，因此静默损坏的风险很大。始终建议在升级时测试并使用 amcheck 验证索引。
   3. 执行重大升级时，使用`rsync --data-only`来升级副本，而未正确处理主服务器和副本停止的顺序。参考：[details](https://postgresql.org/message-id/flat/CAM527d8heqkjG5VrvjU3Xjsqxg41ufUyabD9QZccdAxnpbRH-Q@mail.gmail.com).

7. **GiST** 和 **GIN** 索引的损坏检测仍在开发中，虽然有一些[补丁](https://commitfest.postgresql.org/45/3733/)可以使用 (但需要注意，这些补丁尚未正式发布)。

检测唯一索引损坏的功能也在开发中。

# 我感

细节可以参考：https://www.postgresql.org/docs/current/amcheck.html

>`bt_index_check` does not verify invariants that span child/parent relationships, but will verify the presence of all heap tuples as index tuples within the index when *`heapallindexed`* is `true`.