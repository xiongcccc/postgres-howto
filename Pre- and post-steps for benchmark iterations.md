# Pre- and post-steps for benchmark iterations

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

在进行 PostgreSQL 基准测试时 (参照 [Day 13: How to benchmark]())，我们通常需要在相同的设置上运行多次基准测试迭代。为保证每次迭代的统一性，通常会在每次迭代前后执行以下步骤：

- 前置：刷新缓存 (或者，相反地，预热缓存)。
- 前置：重置累积统计信息。
- 后置：保存统计信息和其他基准测试结果。

这里描述的方法允许以统一的方式进行基准测试。

## 前置步骤：刷新缓存

在准备工作时，通常有两种策略：

- 预热缓存 (可以使用 `pg_prewarm` 扩展，或者多次执行基准测试多次，只记录最后一次的结果。)此处就不讨论了。
- 刷新缓存 (清空操作系统页面缓存和 PostgreSQL 缓冲池)，使每次迭代从"冷启动"开始。

如果选择第二种策略，建议：

- 增加每次迭代的持续时间，以便缓存有足够时间热起来。
- 查看每次迭代内的时间序列指标 (例如，使用 `pgbench` 的 `-P 10` 选项以每隔 10 秒显示延迟/吞吐量)。

如何刷新缓存：

1. PostgreSQL 缓冲池 – 只需重启 Postgres 即可：

```bash
pg_ctl restart -D $PGDATA -m fast
```

2. 释放 `pagecache`、`dentry` 和 `inode`：

```bash
sync  # 将所有缓冲操作写入磁盘
echo 3 > /proc/sys/vm/drop_caches
```

## 前置步骤：重置统计信息

在运行基准测试之前，可能需要重置集群中的所有累积统计信息 (取决于当前的 PostgreSQL 版本)，包括标准扩展如 `pg_stat_statements` 和其他扩展如 `pg_wait_sampling`、`pg_stat_kcache`、`pg_qualstats`(如果安装了)。

以下是重置统计信息的脚本：

```sql
do $$
declare
  reset_cmd_main json := $json${
    "pg_stat_reset()":                             90000,
    "pg_stat_reset_shared('bgwriter')":            90000,
    "pg_stat_reset_shared('archiver')":            90000,
    "pg_stat_reset_shared('io')":                 160000,
    "pg_stat_reset_shared('wal')":                140000,
    "pg_stat_reset_shared('recovery_prefetch')":  140000,
    "pg_stat_reset_slru(null)":                   130000
  }$json$;

  reset_cmd_et json := $json${
    "pg_stat_statements_reset()":         "pg_stat_statements",
    "pg_stat_kcache_reset()":             "pg_stat_kcache",
    "pg_wait_sampling_reset_profile()":   "pg_wait_sampling",
    "pg_qualstats_reset()":               "pg_qualstats"
  }$json$;

  cmd record;
  cur_ver int;
begin
  cur_ver := current_setting('server_version_num')::int;
  raise info 'Current PG version (num): %', cur_ver;

  -- Main reset commands
  for cmd in select * from json_each_text(reset_cmd_main) loop
    if cur_ver >= (cmd.value)::int then
      raise info 'Execute SQL: select %', cmd.key;
      execute format ('select %s', cmd.key);
    end if;
  end loop;

  -- Extension reset commands
  for cmd in select * from json_each_text(reset_cmd_et) loop
    if '' <> (
      select installed_version
      from pg_available_extensions
      where name = cmd.value
    ) then
      raise info 'Execute SQL: select %', cmd.key;
      execute format ('select %s', cmd.key);
    end if;
  end loop;
end
$$;
```

在基准测试之前运行该步骤，确保统计信息是干净的，并且不会受到准备操作的影响。

## 后置步骤：收集基准测试结果

建议执行以下步骤：

1. 导出所有 `pg_stat_*` 视图的内容：

```bash
for viewname in $(psql -tAXc "
    select relname
    from pg_catalog.pg_class
    where relkind = 'view' and relname like 'pg_stat%'" \
); do
  psql -Xc "copy (select * from ${viewname})
      to stdout with csv header delimiter ','" \
    > "${destination}/${viewname}.csv"
done

psql -Xc "copy (select * from pg_stat_kcache())
      to stdout with csv header delimiter ','" \
    > "${destination}/pg_stat_kcache.csv"

psql -Xc "copy (
      select
        event_type as wait_type,
        event as wait_event,
        sum(count) as of_events
      from pg_wait_sampling_profile
      group by event_type, event
      order by of_events desc
    ) to stdout with csv header delimiter ','" \
  > "${destination}/pg_wait_sampling_profile.csv"
```

2. 收集日志文件：压缩并复制日志目录中的文件。

3. 导出当前配置 (`pg_settings`)：

```bash
psql -Xc "copy (
    select * from pg_settings order by name
  ) to stdout with csv header delimiter ','" \
  > "${destination}/pg_settings_all.csv"

psql -Xc "
    select name, setting as current_setting, unit, boot_val as default, context
    from pg_settings
    where source <> 'default'" \
  > "${destination}/pg_settings_non_default.txt"
```

4. 其他有用的基准测试数据：如果需要，收集系统日志、sar 数据等。

