# Autovacuum "queue" and progress

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

我们知道，在某些情况下，`autovacuum` 的设置 (尤其是默认设置) 需要进程调整，以跟上更新频率。一个判断现有设置"不足"的方式是比较 [autovacuum_max_workers](https://postgresqlco.nf/doc/en/param/autovacuum_max_workers/) 和实际使用的工作者进程的数量：

```sql
show autovacuum_max_workers;

select
  state,
  count(*),
  array_agg(left(query, 25) order by xact_start)
from pg_stat_activity
where backend_type = 'autovacuum worker'
group by state;
```

👉 如果我们大部分时间看到正在运行的工作者进程数量达到了 `autovacuum_max_workers`，这是一个强烈信号，表明需要考虑增加工作者进程的数量 (需要重启)，并/或通过[调整配额](https://www.postgresql.org/docs/current/runtime-config-autovacuum.html) (`[auto]vacuum_vacuum_cost_limit`/`[auto]vacuum_vacuum_cost_delay`) 以使它们运行地更快。

然而，我们可能会疑惑：当前有多少个表在"队列"中等待 `autovacuum` 进行处理？对这个"队列"的分析可以了解工作者进程需要处理的工作量，以及当前设置是否"足够"。队列的大小与工作者进程数量的对比，类似于 CPU 负载中的 "load average" 指标。

以下是回答这个问题的报告 ([来源](https://gitlab.com/-/snippets/1889668))，包括：

- 当前全局 `autovacuum` 的设置
- 每个表针对 `vacuum` 的单独设置
- 每个表的死元组数量。

它将这些信息进行比较，并构建需要清理的表的列表。

此报告衍生自[此处](https://github.com/avito-tech/dba-utils/blob/master/munin/vacuum_queue)。

另外，它还检查 `pg_stat_progress_vacuum` 以分析当前正在处理的表。

后续该查询可进一步扩展，以分析"需要自动分析"的表。

```sql
with table_opts as (
  select
    pg_class.oid,
    relname,
    nspname,
    array_to_string(reloptions, '') as relopts
  from pg_class
  join pg_namespace ns on relnamespace = ns.oid
), vacuum_settings as (
  select
    oid,
    relname,
    nspname,
    case
      when relopts like '%autovacuum_vacuum_threshold%' then
        regexp_replace(relopts, '.*autovacuum_vacuum_threshold=([0-9.]+).*', e'\\1')::int8
      else current_setting('autovacuum_vacuum_threshold')::int8
    end as autovacuum_vacuum_threshold,
    case
      when relopts like '%autovacuum_vacuum_scale_factor%'
        then regexp_replace(relopts, '.*autovacuum_vacuum_scale_factor=([0-9.]+).*', e'\\1')::numeric
      else current_setting('autovacuum_vacuum_scale_factor')::numeric
    end as autovacuum_vacuum_scale_factor,
    case
      when relopts ~ 'autovacuum_enabled=(false|off)' then false
      else true
    end as autovacuum_enabled
  from table_opts
), p as (
  select *
  from pg_stat_progress_vacuum
)
select
  coalesce(
    coalesce(nullif(vacuum_settings.nspname, 'public') || '.', '') || vacuum_settings.relname, -- current DB
    format('[something in "%I"]', p.datname) -- another DB
  ) as relation,
  round((100 * psat.n_dead_tup::numeric / nullif(pg_class.reltuples, 0))::numeric, 2) as dead_tup_pct,
  pg_class.reltuples::numeric,
  psat.n_dead_tup,
  format (
    'vt: %s, vsf: %s, %s', -- 'vt' – vacuum_threshold, 'vsf' - vacuum_scale_factor
    vacuum_settings.autovacuum_vacuum_threshold,
    vacuum_settings.autovacuum_vacuum_scale_factor,
    (case when autovacuum_enabled then 'DISABLED' else 'enabled' end)
  ) as effective_settings,
  case
    when last_autovacuum > coalesce(last_vacuum, '0001-01-01') then left(last_autovacuum::text, 19) || ' (auto)'
    when last_vacuum is not null then left(last_vacuum::text, 19) || ' (manual)'
    else null
  end as last_vacuumed,
  coalesce(p.phase, '~~~ in queue ~~~') as status,
  p.pid as pid,
  case
    when a.query ~ '^autovacuum.*to prevent wraparound' then 'wraparound' 
    when a.query ~ '^vacuum' then 'user'
    when a.pid is null then null
    else 'regular'
  end as mode,
  case
    when a.pid is null then null
    else coalesce(wait_event_type || '.' || wait_event, 'f')
  end as waiting,
  round(100.0 * p.heap_blks_scanned / nullif(p.heap_blks_total, 0), 1) as scanned_pct,
  round(100.0 * p.heap_blks_vacuumed / nullif(p.heap_blks_total, 0), 1) as vacuumed_pct,
  p.index_vacuum_count,
  case 
    when psat.relid is not null and p.relid is not null then
      (select count(*) from pg_index where indrelid = psat.relid)
    else null
  end as index_count
from pg_stat_all_tables psat
join pg_class on psat.relid = pg_class.oid
left join vacuum_settings on pg_class.oid = vacuum_settings.oid
full outer join p on p.relid = psat.relid and p.datname = current_database()
left join pg_stat_activity a using (pid)
where
  psat.relid is null
  or p.phase is not null
  or (
    autovacuum_vacuum_threshold
      + (autovacuum_vacuum_scale_factor::numeric * pg_class.reltuples)
    < psat.n_dead_tup
  )
order by status, relation
```

输出示例 (在 psql 中运行并用 `\gx` 代替末尾的 `;`): 

![tables to be autovacuumed](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0067_tables_to_be_autovacuumed_2.png)