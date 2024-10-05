# How to use pg_restore

今天，我们将分享一些使用 `pg_restore` 从转储文件中恢复数据库 (或仅恢复其中一部分) 的技巧。
文档地址：https://postgresql.org/docs/current/app-pgrestore.html

## 并行化与单表限制

当处理以"目录"格式 (`-Fd`) 创建的转储文件时，你可以使用 `-j` (`--jobs`) 选项，通过运行多个 `pg_restore` 工作进程来加速恢复过程。但是，如果有一个或几个大表，这种并行化并不会有帮助 — 单表的转储或恢复不支持并行化。在第 8 天的 [pg_dump](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0008_how_to_speed_up_pg_dump.md) 系列中，我们讨论过这个问题。

例如，如果你创建了一个标准的 `pgbench` 数据库 (如 `pgbench -i -s1000`)，你会发现并行化对转储和恢复都没有太大帮助，因为大部分数据都存储在单个表 `pgbench_accounts` 中。但是，如果你使用了分区 (PG13+ 支持，`pgbench -i -s1000 --partitions=16`)，你会发现并行化可以加速转储和恢复步骤。

## 原子恢复

默认情况下，`pg_restore` 不会在出现错误时停止。这可能会让人感到意外，因为我们已经习惯了 Postgres 中的更**严格**的行为。这也可能导致数据库只部分恢复，但这一点被忽视了。要切换到**严格**模式，请使用 `-e` (`--exit-on-error`)。此外，将恢复过程包裹在一个事务中可能也很有帮助，使用选项 `-1` (`--single-transaction`)。

## 详细信息

要查看恢复的进度和详细信息，请使用 `--verbose` 选项。

## 模式与数据分离

你可以从两个不同的角度来查看你的转储文件，两者都提供了高层次结构化转储的方法。

首先，你可以区分模式和数据 — 使用以下选项：

- `-s` (`--schema-only`) — 仅恢复模式。
- `-a` (`--data-only`) — 仅恢复数据 (当然，模式必须已经存在)。

有趣的是，对于转储来说，这种 "模式 + 数据" 的分离在恢复时间和结果质量方面并不是最有效的：索引是模式的一部分，但如果你先创建索引，然后再加载数据，加载过程会变得更慢，并且索引的质量也会变差。如果在加载数据之后构建索引，效果会更好。

因此，有第二种查看转储结构的方式，这种方式对应于完整恢复过程的常规顺序：

- "pre-data" — 模式定义 (不包括索引、约束、触发器和规则)。
- "data" — 表数据。
- "post-data" — 索引、约束、触发器和规则。

`--section` 选项允许你仅运行恢复过程中的其中一个步骤。这可能会在你想要在各步骤之间执行其他操作时很有帮助，或者你想为不同步骤使用不同的并行化级别 (`-j`)。

## 细粒度控制

还有一种方法可以实现更细粒度的控制。

对于以 "目录" 格式 (`-Fd`) 创建的转储，你可以控制恢复的内容。为此，有两个方便的选项：`-l` 列出内容，`-L` 过滤你需要的内容。例如，考虑以下例子：

```bash
pgbench -i -s100 test --partitions 16
pg_dump -f dump1 -j8 -Fd test
```

现在我们可以使用 `-l` (`--list`) 快速列出转储的内容：

```bash
❯ pg_restore -l dump1
;
; Archive created at 2023-10-15 22:00:43 PDT
; dbname: test
; TOC Entries: 94
; Compression: -1
; Dump Version: 1.14-0
; Format: DIRECTORY
; Integer: 4 bytes
; Offset: 8 bytes
; Dumped from database version: 15.4 (Homebrew)
; Dumped by pg_dump version: 15.4 (Homebrew)
;
...
```

如果我们想要恢复除索引之外的所有内容，我们首先需要准备一个包含转储中所有对象的"列表"文件，但不包括约束和索引：

```bash
❯ pg_restore -l dump1 \
| grep -v INDEX \
| grep -v CONSTRAINT \
> no_indexes.list
```

然后使用 `-L` (`--use-list`) 选项指定这个列表文件：

```bash
❯ psql -c 'create database restored'
CREATE DATABASE
❯ pg_restore -j8 -L no_indexes.list --dbname=restored dump1
```

我们可以看到，恢复后的表没有主键或索引：

```sql
❯ psql restored -c '\d pgbench_accounts_1'
           Table "public.pgbench_accounts_1"
Column  |     Type      | Collation | Nullable | Default
----------+---------------+-----------+----------+---------
aid      | integer       |           | not null |
bid      | integer       |           |          |
abalance | integer       |           |          |
filler   | character(84) |           |          |
Partition of: pgbench_accounts FOR VALUES FROM (MINVALUE) TO (625001)
```

## 权限、属主等

在某些情况下，当在不同环境或集群之间复制或移动架构和数据时，以下选项可以非常有帮助 (这些选项的名称即其含义)：

- `--no-owner`
- `--no-privileges`
- `--no-security-labels`
- `--no-publications`
- `--no-subscriptions`
- `--no-tablespaces`

然而，如果原始数据库中使用了RLS (行级安全)，没有一个选项可以跳过恢复转储中包含的 `CREATE POLICY` 查询。在这种情况下，我们需要使用前述的细粒度控制方法来移除 `CREATE POLICY` 命令。

因此，当在不同环境之间移动架构和数据时，从转储中恢复的可能步骤如下：

```bash
pg_restore --list /path/to/dump \
    | grep -v POLICY \
  > dump-no-policies.list

pg_restore \
  --jobs=8 \
  --dbname=${DBNAME} \
  --use-list=./dump-no-policies.list \
  --no-owner \
  --no-privileges \
  --no-security-labels \
  --no-publications \
  --no-subscriptions \
  --no-tablespaces \
  --verbose \
  --exit-on-error \
  /path/to/dump
```

## 附录

很多人 (包括我) 常常忘记这一步 — 实际上，它应该成为 `pg_restore` 的默认行为，但事实并非如此：

在运行 `pg_restore` 之后，不要忘记：

- 收集统计信息 (除非你想等待 `autovacuum` 来做这件事)  — `ANALYZE`。
- 构建可见性映射 — 执行 `VACUUM`。

因此，记得运行 `VACUUM ANALYZE`。也可以并行化：`vacuumdb --jobs=$N`。