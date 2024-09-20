# Ad-hoc monitoring

// 我每天发布一篇新的 PostgreSQL "howto" 文章。和我一起踏上这段旅程 — 订阅、提供反馈、分享！

在某些情况下，我们需要观察 Postgres 或其运行环境中的一些值 (操作系统、文件系统等)，然而：

- 我们没有好的监控工具，或者
- 现有的监控缺少我们所需的某些功能，或者
- 我们不完全信任现有的监控 (例如，你是刚开始接手某个系统的工程师)。

能够根据特定的指标组织一个"临时"的观察是一项重要的技能。此处我们将描述一些技巧和方法，可能会对你有帮助。

在某些情况下，你可以快速安装类似 [Netdata](https://www.netdata.cloud/) 的工具 (一个非常强大的现代监控工具，能快速安装，并且有Postgres plugin)，或者使用一些临时的控制台工具，如 [pgCenter](https://github.com/lesovsky/pgcenter) 或 [pg_top](https://pg_top.gitlab.io/)。但你可能仍然希望手动监控某些特定的方面。

以下假设我们使用的是 Linux，但大多数方法也可以应用于 macOS 或 BSD 系统。

## 关键原则

- 观察不应依赖于你的互联网连接。
- 保存结果以便长期存储和后续分析。
- 记录时间戳。
- 防止程序因意外中断 (如按下Ctrl-C) 而终止。
- 在 Postgres 重启后仍能保持观察。
- 既要记录正常信息，也要记录错误信息。
- 优先以有助于编程处理的形式收集数据 (例如 CSV)。

## 示例

假设我们需要收集 `pg_stat_activity` (pgsa) 的样本，以研究运行超过 1 分钟的长事务。

以下是操作步骤 — 我们将逐步详细讨论。

1. 启动一个 `tmux` 会话。这是一个幂等的代码片段，可以用来连接到一个名为 "observe" 的现有会话，或者如果找不到该会话，则创建一个：

```bash
export TMUX_NAME=observe
tmux a -t $TMUX_NAME || tmux new -s $TMUX_NAME
```

2. 每秒收集一次 `pgsa` 样本，并无限次地记录日志 (直到手动中断)：

```bash
while sleep 1; do
    psql -XAtc "
        copy (
        with samples as (
            select
            clock_timestamp(),
            clock_timestamp() - xact_start as xact_duration,
            *
            from pg_stat_activity
        )
        select *
        from samples
        where xact_duration > interval '1 minute'
        order by xact_duration desc
        ) to stdout delimiter ',' csv
    " 2>&1 \
    | tee -a long_tx_$(date +%Y%m%d).log.csv
done
```

各种技巧和提示详解

## 使用 tmux (或 screen)

好处显而易见：如果你遇到网络连接问题而断开连接，只要 tmux 会话在服务器上运行，工作不会丢失。遵循预定义的会话命名约定在团队协作时也很有帮助。

## 如何使用循环/批处理

某些程序支持批量报告 (例如：`iostat -x 5`、`top -b -n 100 -d 5`)，而有些不支持。在后者的情况下，可以使用 `while` 循环。我倾向于使用无限循环，如 `while sleep 5; do ... ; done`。它有一个小缺点 — 开始时会先等待一会儿，然后再执行有用的工作 — 但它的好处是，大多数时候你可以使用 `Ctrl-C` 中断。

## 使用 psql 选项：-X, -A, -t

用于观测相关采样的 `psql` 最佳实践 (以及通常的工作自动化)：

- 始终使用 `-X` — 这在某一天会对你非常有用，尤其当你在一个服务器上工作时，意外的`~/.psqlrc`文件可能包含影响输出的不确定内容（例如，一个简单的 `\timing on`）。`-X` 选项告诉psql忽略 `~/.psqlrc `文件。
- 选项 `-A` 和 `-t` — 提供无对齐且无标题的输出。这有助于生成更容易解析和处理的大量结果。

## 使用 CSV 作为输出

有几种方式可以生成 CSV：

- 使用 psql 的命令 `\copy` — 这种情况下，结果将保存在客户端。
- `copy (...) to '/path/to/file'` — 这种方式会将结果保存在服务器上 (如果路径对于 Postgres 运行的操作系统用户不可写，可能会遇到权限问题)。
- `psql --csv -F,` — 生成 CSV (但可能存在字段分隔符与转义值冲突的问题)。
- `copy (...) to stdout` — 这种方式可能最适合采样和记录日志。

## 日志技巧：不要丢失 STDERR，记录时间戳，追加而非覆盖

为了后续分析 (谁知道我们几个小时后会检查什么？)，最好将所有内容保存到文件中。

但同样重要的是不要丢失错误信息 — 通常，它们被打印到 `STDERR`，因此我们需要将它们写入单独的文件。我们还希望不要丢失文件中的现有内容，所以我们不想简单地覆盖 (`>`)，而是要追加 (`>>`)：

```bash
command   2>>error.log >>messages.log
```

或者将所有内容都重定向到一个文件中：

```bash
command  &>>everything.log
```

如果你想同时查看和记录所有内容，使用 `tee` — 或者在追加模式下使用 `tee -a` (这里，`2>&1 `将 `STDERR` 重定向到 `STDOUT`，之后 `tee` 获取从 `STDOUT` 输出的所有内容)：

```
command 2>&1 | tee -a everything.log
```

如果输出缺少时间戳 (在我们上面的 psql 代码片段中则不需要)，可以使用 `ts` 为每一行前添加时间戳：

```bash
command 2>&1 \
  ts \
  tee -a everything.log
```

最后，通常明智的做法是使用当前日期 (甚至是日期+时间，取决于情况) 为结果文件命名：

```bash
command 2>&1 \
  | ts \
  | tee -a observing_our_command_$(date +%Y%m%d).log
```

使用 `tee` 的一个缺点是，有时你可能会意外中断它 (例如，按错了 `tmux` 窗口/窗格中的 `Ctrl-C`)。因此，有些人更喜欢使用 `nohup ... &` 将观测任务放在后台运行，并使用 `tail -f` 查看结果。

## 结果处理

如果结果是 CSV 格式，后续处理非常方便 — Postgres 对其处理非常好。我们只需要记住使用的列集，就可以创建一个表并加载数据：

```sql
nik=# create table log_loaded as select clock_timestamp(), clock_timestamp() - xact_start as xact_duration, * from pg_stat_activity limit 0;
SELECT 0
nik=# copy log_loaded from '/Users/nik/long_tx_20231006.log.csv' csv delimiter ',';
COPY 4
```

然后你可以用 SQL 进行分析。或者你也可以将 CSV 加载到某些电子表格工具中，然后分析数据并创建一些图表。

## 另一种采样方式——直接在 psql 中操作

在 psql 中，你可以使用 `\o | tee -a logfile` 和`\watch` 来同时查看数据和记录日志 — 不过注意，这不会捕捉到文件中的错误。示例：

```sql
nik=# \o | tee -a alternative.log

nik=# \t
Tuples only is on.
nik=# \a
Output format is unaligned.
nik=# \f ┃
Field separator is "┃".
nik=# select clock_timestamp(), 'test' as c1 \watch 3
2023-10-06 21:01:05.33987-07┃test
2023-10-06 21:01:08.342017-07┃test
2023-10-06 21:01:11.339183-07┃test
^C
nik=# \! cat  alternative.log
2023-10-06 21:01:05.33987-07┃test
2023-10-06 21:01:08.342017-07┃test
2023-10-06 21:01:11.339183-07┃test
nik=#
```