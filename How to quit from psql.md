# How to quit from psql

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

只需输入 `quit` (或 `exit`)。就是这样。

除非你使用的是 Postgres 10 或更早的版本 — 在这种情况下，使用 `\q`。Postgres 11 将在几周后 ([最后一个小版本 11.22 计划于 11 月 9 日发布](https://www.postgresql.org/support/versioning/)) 正式退休，因此我们可以说对于所有受支持的版本，只需输入 `quit` 即可。

如果你需要在非交互模式下使用退出命令，那么也可以使用 `\q`：

```bash
psql -c '\q'  – 这个命令可用

psql -c 'quit'  – 这个命令不可用
```

另外，`Ctrl-D` 也可以使用。作为退出控制台的标准方式，它在各种 shell 中都适用，例如 bash, zsh, irb, python, node 等。

---

这个问题仍然是 [StackOverflow](https://stackoverflow.com/questions/9463318/how-to-exit-from-postgresql-command-line-utility-psql) 和其他平台上排名前五的热门问题之一。