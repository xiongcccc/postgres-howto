# How to create an index, part 1

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

索引创建很简单：

```sql
create index concurrently on t1(c1);
```

一个更长的包含显式命名的格式：

```sql
create index concurrently i_1 on t1(c1);
```

甚至更长，显式包括索引名称和类型：

```sql
create index concurrently i_1
on t1
using btree(c1);
```

下面是一些"最佳实践"的考虑。

## 索引命名

创建索引时：

- 使用显式命名利于更好的控制。
- 建立并遵循某种命名规则。例如，可以在索引名称中包含表和列的名称：`i_table1_col1_col2`。其他可考虑包括的属性：
  - 是常规索引还是唯一索引？
  - 索引类型
  - 是否为部分索引？
  - 排序、使用的表达式
  - 使用的操作符类

## 设置

1. `statement_timeout`：如果全局设置了 `statement_timeout`，在创建索引的会话中将其取消：

~~~sql
set statement_timeout to 0;
~~~

或者，可以为创建索引创建一个专用的数据库用户，并调整其 `statement_timeout`：

~~~sql
alter user index_creator set statement_timeout to 0;
~~~

2. `maintenance_work_mem`：将此参数 ([docs](https://postgresqlco.nf/doc/en/param/maintenance_work_mem/)) 提高到几百 MiB 或几 GiB (在较大的系统上) 以更快地创建索引。

## CONCURRENTLY

除非以下情况，建议始终使用 `CONCURRENTLY` 选项：

- 在一个尚未使用的表上创建索引 — 例如，一个刚创建的表 (此时可以避免使用 `CONCURRENTLY`，以便能够在事务块中一起创建索引)。
- 在独自操作数据库。

👉 这是其在 16 版本中实现的[原理解读](https://github.com/postgres/postgres/blob/c136eb02981566d56e950f12ab7ee4a6ea51d698/src/backend/catalog/index.c#L1443-L1511)。

使用 `CONCURRENTLY` 会增加索引的创建时间，但会优雅地持有锁，不会长时间阻塞其他会话。使用此方法，可以在创建新索引时允许持续访问表、维护数据完整性和一致性，以及最大限度地减少对正常数据库操作的干扰之间取得平衡。

当使用此选项时，创建创建可能会因各种原因失败 — 例如，如果尝试同时创建两个索引，可能会看到类似如下的错误：

```sql
nik=# create index concurrently i_3 on t1 using btree(c1);
ERROR:  deadlock detected
DETAIL:  Process 518 waits for ShareLock on virtual transaction 3/459; blocked by process 553.
Process 553 waits for ShareUpdateExclusiveLock on relation 16402 of database 16401; blocked by process 518.
HINT:  See server log for query details.
```

一般来说，Postgres 支持 DDL 事务，但对于 `CREATE INDEX CONCURRENTLY` 不适用：

- 不能在事务块中包含 `CREATE INDEX CONCURRENTLY`。
- 如果操作失败，会留下一个无效的索引，因此需要进行清理。

## 清理和重试

由于 `CREATE INDEX CONCURRENTLY `可能会失败，因此应准备手动或自动重试。在重试之前，需要清理无效的索引。此时，使用显式命名和某种命名约定的好处就体现出来了。清理时，同样使用 `CONCURRENTLY`：

```sql
drop index concurrently i_1;
```

## 进度监控

如何监控索引创建进度参照 [Day 15: How to monitor CREATE INDEX / REINDEX progress in Postgres 12+]().

## ANALYZE

通常，Postgres `autovacuum` 会维护最新的每列统计信息，并对内容发生足够变化的每个表运行 `ANALYZE`。

在新索引创建后，如果只是对列进行索引，通常无需重新构建统计信息。

但是，如果是对表达式创建索引，例如：

```sql
create index i_t1_lower_email on t1 (lower(email));
```

那么你应该在表上运行 `ANALYZE`，以便 Postgres 收集表达式的统计信息：

```sql
analyze verbose t1;
```