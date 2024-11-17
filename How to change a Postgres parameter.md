# How to change a Postgres parameter

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

如果你需要更改 Postgres 参数 (又称之为 GUC，Grand Unified Configuration) 以实现永久效果，请按照以下步骤操作。

参考文档：[设置参数](https://postgresql.org/docs/current/config-setting.html.)

## 1) 确认是否需要重启

快速检查是否需要重启的两种方法：

- 使用 [postgresqlco.nf](https://postgresqlco.nf/) 并查看 `Restart: xxx` 字段。例如，对于 [max_wal_size](https://postgresqlco.nf/doc/en/param/max_wal_size/)，`Restart: false`，而对于 [shared_buffers](https://postgresqlco.nf/doc/en/param/shared_buffers) 则为 `true`。
- 检查 `pg_settings` 中的 `context` 字段 — 如果是 `postmaster`，则需要重启，否则不需要 (例外情况：`internal` — 这类参数无法更改)。例如：

```sql
nik=# select name, context
from pg_settings
where name in ('max_wal_size', 'shared_buffers');
      name      |  context
----------------+------------
 max_wal_size   | sighup
 shared_buffers | postmaster
(2 rows)
```

## 2) 进行更改

在 Postgres 配置文件中应用更改 (`postgresql.conf` 或其依赖项，如果使用了 `include` 命令的话)。除非必要，建议避免使用 `ALTER SYSTEM`，以免将来产生混淆 (它会写入 `postgresql.auto.conf`，且可能很容易被忽视；参见[相关讨论](https://postgresql.org/message-id/flat/CA%2BVUV5rEKt2%2BCdC_KUaPoihMu%2Bi5ChT4WVNTr4CD5-xXZUfuQw%40mail.gmail.com))。

## 3) 应用更改

如果需要重启，则重启 Postgres。如果忘记重启，稍后可以在 `pg_settings` 中查看 `pending_restart` 来检测未应用的更改。

如果不需要重启，在超级用户下执行：

```sql
select pg_reload_conf();
```

或者使用以下方法之一：

- `pg_ctl reload $PGDATA`

- 发送 `SIGHUP` 信号给 postmaster 进程，例如：

  ~~~bash
  kill -HUP $(cat "${PGDATA}/postmaster.pid")
  ~~~

当应用了无需重启的更改时，Postgres 日志中会显示类似以下内容：

```bash
LOG: received SIGHUP, reloading configuration files
LOG: parameter "max_wal_size" changed to "2GB"
```

## 4) 验证更改

使用 `SHOW` 或 `current_setting(...)` 确认更改已生效，例如：

```sql
nik=# show max_wal_size;
 max_wal_size
--------------
 2GB
(1 row)
```

或者

```sql
nik=# select current_setting('max_wal_size');
 current_setting
-----------------
 2GB
(1 row)
```

## 额外信息：数据库级、用户级和表级设置

如果 `pg_settings` 中的 `context` 为 `user` 或 `superuser`，那么可以在数据库或用户级别调整设置，例如：

```sql
alter database verbosedb set log_statement = 'all';
alter user hero set statement_timeout = 0;
```

这些结果可以在 `pg_db_role_setting` 中查看：

```sql
nik=# select * from pg_db_role_setting;
 setdatabase | setrole |       setconfig
-------------+---------+-----------------------
       24580 |       0 | {log_statement=all}
           0 |   24581 | {statement_timeout=0}
(2 rows)
```

当使用 `CREATE TABLE `或 `ALTER TABLE` 时，某些设置还可以在表级别进行调整，参照 [CREATE TABLE / Storage Parameters](https://postgresql.org/docs/current/sql-createtable.html#SQL-CREATETABLE-STORAGE-PARAMETERS)。请注意命名差异：`autovacuum_enabled` 用于启用或禁用特定表的 autovacuum 守护进程，而全局设置名称为 `autovacuum`。

## 我见

在 15 中，还支持了 `\dconfig` 命令，也很方便。

>The \dconfig command can display the parameter settings of the instance. If no option is specified, a list of parameters that have changed from the default values is displayed. If a parameter name is specified, the specific parameter setting value can be checked. Wildcards can be used for parameter names.

~~~sql
postgres=# \dconfig *mem*
     List of configuration parameters
            Parameter             | Value 
----------------------------------+-------
 autovacuum_work_mem              | -1
 dynamic_shared_memory_type       | posix
 enable_memoize                   | on
 hash_mem_multiplier              | 2
 logical_decoding_work_mem        | 64MB
 maintenance_work_mem             | 64MB
 min_dynamic_shared_memory        | 0
 multixact_member_buffers         | 256kB
 shared_memory_size               | 145MB
 shared_memory_size_in_huge_pages | 73
 shared_memory_type               | mmap
 work_mem                         | 4MB
(12 rows)
~~~