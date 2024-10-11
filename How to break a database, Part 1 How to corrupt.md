# How to break a database, Part 1: How to corrupt

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

有时候，你可能想要破坏一个数据库 — 用于学习目的，模拟故障，学习如何处理这些故障，测试缓解措施等。

让我们讨论一些破坏的方法。

⚠️ 不要在生产环境中尝试，除非你是一名混沌工程师 ⚠️

## 数据损坏

有很多类型的损坏，并且有非常简单的方法可以使数据库损坏，例如：

👉 **直接修改系统目录：**

```sql
nik=# create table t1(id int8 primary key, val text);
CREATE TABLE

nik=# delete from pg_attribute where attrelid = 't1'::regclass and attname = 'val';
DELETE 1

nik=# table t1;
ERROR:  pg_attribute catalog is missing 1 attribute(s) for relation OID 107006
LINE 1: table t1;
          ^
```

更多方法可以在这篇文章中找到：[How to corrupt your PostgreSQL database](https://cybertec-postgresql.com/en/how-to-corrupt-your-postgresql-database/)，其中一些有趣的方法包括：

- 设置 `fsync=off` 然后对 Postgres 使用 `kill -9` (或者 `pg_ctl stop -m immediate`)。
- 使用 `kill -9` + `pg_resetwal -f`。

一个有用的方法是使用 `dd` 直接写入数据文件。此方法可以用于模拟通过校验和验证检测到的损坏 ([Day 37: How to enable data checksums without downtime]())。在这篇文章中也有展示：[pg_healer: repairing Postgres problems automatically](https://endpointdev.com/blog/2016/09/pghealer-repairing-postgres-problems/).

首先，创建一个表并查看数据文件所在位置：

```sql
nik=# show data_checksums;
 data_checksums
----------------
 on
(1 row)

nik=# create table t1 as select i from generate_series(1, 10000) i;
SELECT 10000

nik=# select count(*) from t1;
 count
-------
 10000
(1 row)

nik=# select format('%s/%s',
  current_setting('data_directory'),
  pg_relation_filepath('t1'));
                      format
---------------------------------------------------
 /opt/homebrew/var/postgresql@15/base/16384/123388
(1 row)
```

现在，使用 `dd` 直接写一些垃圾数据至文件中 (请注意，这里我们使用的是 macOS 版本，其中 `dd` 有选项 `oseek` - 在 Linux 上，它是 `seek_bytes`)，然后重启 Postgres 以确保表不再存在于缓冲池中：

```bash
❯ echo -n "BOOoo" \
  | dd conv=notrunc bs=1 \
    oseek=4000 count=1 \
    of=/opt/homebrew/var/postgresql@15/base/16384/123388
 1+0 records in
 1+0 records out
 1 bytes transferred in 0.000240 secs (4167 bytes/sec)

❯ brew services stop postgresql@15
 Stopping `postgresql@15`... (might take a while)
 ==> Successfully stopped `postgresql@15` (label: homebrew.mxcl.postgresql@15)

❯ brew services start postgresql@15
 ==> Successfully started `postgresql@15` (label: homebrew.mxcl.postgresql@15)
```

成功损坏 — 数据校验和机制对此发出了警告：

```sql
nik=# table t1;
WARNING:  page verification failed, calculated checksum 52379 but expected 35499
ERROR:  invalid page in block 0 of relation base/16384/123388
```

🔜 未完待续 ...