# How to understand LSN values and WAL filenames

## 如何读取 LSN 值

LSN — 日志序列号，是指向预写日志 (WAL) 中某个位置的指针。理解并学会如何使用它，对于处理流复制、逻辑复制、备份和恢复非常重要。Postgres 相关文档：

- [WAL Internals](https://postgresql.org/docs/current/wal-internals.html)
- [pg_lsn Type](https://postgresql.org/docs/current/datatype-pg-lsn.html)

LSN 是一个 8 字节 (32 位) 的值 ([source code](https://gitlab.com/postgres/postgres/blob/4f2994647ff1e1209829a0085ca0c8d237dbbbb4/src/include/access/xlogdefs.h#L17))。它的表现形式为 `A/B` (更具体地说，`A/BBbbbbbb`，如下所述)，其中 `A` 和 `B` 都是 4 字节值。例如：

```sql
nik=# select pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 5D/257E19B0
(1 row)
```

- `5D` 是 LSN 的高 4 字节 (32 位) 部分
- `257E19B0` 可以进一步分为两部分：
  - `25` — LSN 的低 4 字节部分 (更具体地说，仅是该 4 字节部分最高的 1 字节)。
  - `7E19B0` — WAL 的偏移量 (默认情况下为 `16 MiB`；在某些情况下会更改 — 例如 RDS 将其更改为 `64 MiB`)。

有趣的是，LSN 值可以进行比较，甚至可以进行减法运算 — 前提是我们使用 `pg_lsn` 数据类型。结果将以字节为单位表示：

```sql
nik=# select pg_lsn '5D/257D6780' - '5D/251A8118';
 ?column?
----------
  6481512
(1 row)

nik=# select pg_size_pretty(pg_lsn '5D/257D6780' - '5D/251A8118');
 pg_size_pretty
----------------
 6330 kB
(1 row)
```

这也意味着我们可以通过查看从"点 0" (值 `0/0`) 出发的距离来获得 LSN 的整数值：

```sql
nik=# select pg_lsn '5D/257D6780' - '0/0';
   ?column?
--------------
 400060934016
(1 row)
```

## 如何读取 WAL 文件名

现在，让我们看看 LSN 值如何与 WAL 文件名对应 (文件位于 `$PGDATA/pg_wal` 中)。我们可以使用 `pg_walfile_name()` 函数获取任何给定 LSN 相对应的 WAL 文件名：

```sql
nik=# select pg_current_wal_lsn(), pg_walfile_name(pg_current_wal_lsn());
 pg_current_wal_lsn |     pg_walfile_name
--------------------+--------------------------
 5D/257E19B0        | 000000010000005D00000025
(1 row)
```

这里的 `000000010000005D00000025` 是 WAL 文件名，它由三个 4 字节 (32 位) 组成：

1. `00000001` — 时间线 ID (TimeLineID)，这是一个从 1 开始的连续 "历史号"，当 Postgres 集群初始化时就会启动。它用于 "标识不同的数据库历史记录，以防止在恢复数据库的先前状态之后出现混乱" ([source code](https://gitlab.com/postgres/postgres/blob/4f2994647ff1e1209829a0085ca0c8d237dbbbb4/src/include/access/xlogdefs.h#L50))。
2. `0000005D` — LSN 序列号的高 4 字节部分。
3. `00000025` — 可以分为两部分：
   - `000000` — 6 个前导零
   - `25` — 序列号低位部分的最高字节

需要记住的重要一点是，WAL 文件名的第三个 4 字节内容里有 6 个前导零 — 通常，这会导致在比较两个 WAL 文件名以理解其中包含的 LSN 时产生混淆和错误。

以下示例中的说明很有用：

```
LSN:                5D/      25   7E19B0
WAL: 00000001 0000005D 00000025
```

这在需要处理 LSN 值或 WAL 文件名，或者同时处理它们时非常有帮助。快速导航或比较这些值以了解它们之间的"距离"可能很有用。例如，在以下情况下：

- 了解服务器每天生成多少字节的数据。
- 自复制槽创建以来经过了多长时间。
- 备份之间的"距离"是多少。
- 为了达到一致性位点，还需要重放多少 WAL 数据。

此外，除了官方文档外，还有一些优秀的博客文章值得阅读：

- [Postgres 9.4 feature highlight - LSN datatype](https://paquier.xyz/postgresql-2/postgres-9-4-feature-highlight-lsn-datatype/)
- [Postgres WAL Files and Sequence Numbers](https://crunchydata.com/blog/postgres-wal-files-and-sequuence-numbers)
- [WAL, LSN, and File Names](https://fluca1978.github.io/2020/05/28/PostgreSQLWalNames)

---

谢谢阅读！和往常一样，请与从事 #PostgreSQL 的同事和朋友分享这篇文章。

# 我见

![Postgres WAL Files and Sequence Numbers | Crunchy Data Blog](https://imagedelivery.net/lPM0ntuwQfh8VQgJRu0mFg/c94fcd44-0fb3-4f28-db7a-ff8076c82a00/public)

对于给定一个 LSN，我们可以迅速算出对应的 WAL 文件名，假设 pg_current_wal_insert_lsn() 返回 `0/9A80D10` ，如果当前时间线为 1，那么 logid 就是 `0/9A80D10` 中的第一个数字 — 0，logseg 就是 `0/9A80D10` 中的第二个数字除以 16MB 的大小，9A80D10 左移 6 位，也就是 09。那么根据 WAL 文件的格式timelineID + logid + logseg，则相当于："00000001"+"00000000"+"00000009"，即为："000000010000000000000009" ，而写位置的偏移量则是第二个数字 "9A80D10" 后六位 "A80D10"。