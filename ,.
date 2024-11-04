# 如何查找未使用的索引

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

## 为什么要清理未使用的索引

数据库中包含未使用的索引是有害的，因为：

1. 占用了额外的磁盘空间。
2. 会减慢许多写操作 (`INSERT` 和非 `HOT UPDATE`) 的速度。
3. 会"污染"缓存 (操作系统页缓存，Postgres 缓冲池)，因为在上述写操作期间，索引块会被加载到缓存中。
4. 出于同样的原因，它们会"污染" WAL (因此，复制和备份系统需要处理更多数据)。
5. 每个索引都会减慢查询的规划速度 (参照[基准测试](https://gitlab.com/postgres-ai/postgresql-consulting/tests-and-benchmarks/-/issues/41#note_1602598974))。
6. 最后，如果没有执行计划缓存 (没有使用预备语句)，每个额外的索引都会增加涉及查询处理的关系数量 (表和索引) 达到 `FP_LOCK_SLOTS_PER_BACKEND=16` 的机会，这在高 QPS 的情况下会导致发生 `LWLock:LockManager` 争用的概率增加 (参照[基准测试](https://gitlab.com/postgres-ai/postgresql-consulting/tests-and-benchmarks/-/issues/41))。

因此，强烈建议定期分析和清理未使用的索引。

## 清理未使用索引的一般算法

1. 使用下面提供的查询之一 (它们都涉及到 `pg_stat_user_indexes`，其中可以找到索引的使用统计信息)，以识别未使用的索引。
2. 通过检查 `pg_stat_database` 中数据库的 `stats_reset` 时间戳，确保统计数据足够久远。例如，它们代表至少 1 个月的使用数据 (取决于应用程序，此阈值可以更多，也可以更少)。如果不满足此条件，请推迟分析。
3. 如果备库接收只读流量，也要分析所有的备库，注意统计数据重置的时间戳。
4. 如果有多个数据库的生产实例 (例如，开发的系统安装在各个地方，每个都有自己的数据库)，请按照上述步骤 1-3 尽可能多地分析数据库实例。
5. 最终，建立一个可以可靠地被称为"未使用"索引的列表 — 我们知道在重要时间内，这些索引在主库和备库以及我们可以观察到的所有生产系统中都未被使用。
6. 对于列表中的每个索引，使用 `DROP INDEX CONCURRENTLY` 进行删除。

## 用于分析未使用索引的查询

基础查询很简单：

```sql
SELECT *
FROM pg_stat_user_indexes
WHERE
  idx_scan = 0
  AND NOT indisunique;
```

了解统计数据上次重置的时间：

```sql
SELECT
  stats_reset,
  AGE(NOW(), stats_reset)
FROM pg_stat_database
WHERE datname = current_database();
```

为了进行全面的分析，你可以使用来自 [postgres-checkup](https://gitlab.com/postgres-ai/postgres-checkup/-/blob/7f5c45b18b04c665d11f744486db047a1dbc680e/resources/checks/H002_unused_indexes.sh) 的此查询，它还包括对很少使用的索引的分析、用于清理索引的 DDL 命令，并以 JSON 格式呈现结果，以便于在可观测性和自动化系统中更容易进行集成：

~~~sql
with const as (
  select 0 as min_relpages -- on very large DBs, increase it to, say, 100
), fk_indexes as (
  select
    n.nspname as schema_name,
    ci.relname as index_name,
    cr.relname as table_name,
    (confrelid::regclass)::text as fk_table_ref,
    array_to_string(indclass, ', ') as opclasses
  from pg_index i
  join pg_class ci on ci.oid = i.indexrelid and ci.relkind = 'i'
  join pg_class cr on cr.oid = i.indrelid and cr.relkind = 'r'
  join pg_namespace n on n.oid = ci.relnamespace
  join pg_constraint cn on cn.conrelid = cr.oid
  left join pg_stat_user_indexes si on si.indexrelid = i.indexrelid
  where
     contype = 'f'
     and i.indisunique is false
     and conkey is not null
     and ci.relpages > (select min_relpages from const)
     and si.idx_scan < 10
), table_scans as (
  select
    relid,
    tables.idx_scan + tables.seq_scan as all_scans,
    (tables.n_tup_ins + tables.n_tup_upd + tables.n_tup_del) as writes,
    pg_relation_size(relid) as table_size
  from pg_stat_user_tables as tables
  join pg_class c on c.oid = relid
  where c.relpages > (select min_relpages from const)
), all_writes as (
  select sum(writes) as total_writes
  from table_scans
), indexes as (
  select
    i.indrelid,
    i.indexrelid,
    n.nspname as schema_name,
    cr.relname as table_name,
    ci.relname as index_name,
    quote_ident(n.nspname) as formated_schema_name,
    coalesce(nullif(quote_ident(n.nspname), 'public') || '.', '') || quote_ident(ci.relname) as formated_index_name,
    quote_ident(cr.relname) as formated_table_name,
    coalesce(nullif(quote_ident(n.nspname), 'public') || '.', '') || quote_ident(cr.relname) as formated_relation_name,
    si.idx_scan,
    pg_relation_size(i.indexrelid) as index_bytes,
    ci.relpages,
    (case when a.amname = 'btree' then true else false end) as idx_is_btree,
    pg_get_indexdef(i.indexrelid) as index_def,
    array_to_string(i.indclass, ', ') as opclasses
  from pg_index i
  join pg_class ci on ci.oid = i.indexrelid and ci.relkind = 'i'
  join pg_class cr on cr.oid = i.indrelid and cr.relkind = 'r'
  join pg_namespace n on n.oid = ci.relnamespace
  join pg_am a ON ci.relam = a.oid
  left join pg_stat_user_indexes si on si.indexrelid = i.indexrelid
  where
    i.indisunique = false
    and i.indisvalid = true
    and ci.relpages > (select min_relpages from const)
), index_ratios as (
  select
    i.indexrelid as index_id,
    i.schema_name,
    i.table_name,
    i.index_name,
    idx_scan,
    all_scans,
    round(
      case
        when all_scans = 0 then 0.0::numeric
        else idx_scan::numeric / all_scans * 100
      end,
      2
    ) as index_scan_pct,
    writes,
    round(
      case
        when writes = 0 then idx_scan::numeric
        else idx_scan::numeric / writes
      end,
      2
    ) as scans_per_write,
    index_bytes as index_size_bytes,
    table_size as table_size_bytes,
    i.relpages,
    idx_is_btree,
    index_def,
    formated_index_name,
    formated_schema_name,
    formated_table_name,
    formated_relation_name,
    i.opclasses,
    (
      select count(1)
      from fk_indexes fi
      where
        fi.fk_table_ref = i.table_name
        and fi.schema_name = i.schema_name
        and fi.opclasses like (i.opclasses || '%')
    ) > 0 as supports_fk
  from indexes i
  join table_scans ts on ts.relid = i.indrelid
), never_used_indexes as ( -- Never used indexes
  select
    'Never Used Indexes' as reason,
    ir.*
  from index_ratios ir
  where
    idx_scan = 0
    and idx_is_btree
  order by index_size_bytes desc
), never_used_indexes_num as (
  select
    row_number() over () num,
    nui.*
  from never_used_indexes nui
), never_used_indexes_total as (
  select
    sum(index_size_bytes) as index_size_bytes_sum,
    sum(table_size_bytes) as table_size_bytes_sum
  from never_used_indexes
), never_used_indexes_json as (
  select
    json_object_agg(coalesce(nuin.schema_name, 'public') || '.' || nuin.index_name, nuin) as json
  from never_used_indexes_num nuin
), rarely_used_indexes as ( -- Rarely used indexes
  select
    'Low Scans, High Writes' as reason,
    *,
    1 as grp
  from index_ratios
  where
    scans_per_write <= 1
    and index_scan_pct < 10
    and idx_scan > 0
    and writes > 100
    and idx_is_btree
  union all
  select
    'Seldom Used Large Indexes' as reason,
    *,
    2 as grp
  from index_ratios
  where
    index_scan_pct < 5
    and scans_per_write > 1
    and idx_scan > 0
    and idx_is_btree
    and index_size_bytes > 100000000
  union all
  select
    'High-Write Large Non-Btree' as reason,
    index_ratios.*,
    3 as grp
  from index_ratios, all_writes
  where
    (writes::numeric / ( total_writes + 1 )) > 0.02
    and not idx_is_btree
    and index_size_bytes > 100000000
  order by grp, index_size_bytes desc
), rarely_used_indexes_num as (
  select row_number() over () num, rui.*
  from rarely_used_indexes rui
), rarely_used_indexes_total as (
  select
    sum(index_size_bytes) as index_size_bytes_sum,
    sum(table_size_bytes) as table_size_bytes_sum
  from rarely_used_indexes
), rarely_used_indexes_json as (
  select
    json_object_agg(coalesce(ruin.schema_name, 'public') || '.' || ruin.index_name, ruin) as json
  from rarely_used_indexes_num ruin
), do_lines as (
  select
    format(
      'DROP INDEX CONCURRENTLY %s; -- %s, %s, table %s',
      formated_index_name,
      pg_size_pretty(index_size_bytes)::text,
      reason,
      formated_relation_name
    ) as line
  from never_used_indexes nui
  order by table_name, index_name
), undo_lines as (
  select
    replace(
      format('%s; -- table %s', index_def, formated_relation_name),
      'CREATE INDEX',
      'CREATE INDEX CONCURRENTLY'
    ) as line
  from never_used_indexes nui
  order by table_name, index_name
), database_stat as (
  select
    row_to_json(dbstat)
  from (
    select
      sd.stats_reset::timestamptz(0),
      age(
        date_trunc('minute', now()),
        date_trunc('minute', sd.stats_reset)
      ) as stats_age,
      ((extract(epoch from now()) - extract(epoch from sd.stats_reset)) / 86400)::int as days,
      (select pg_database_size(current_database())) as database_size_bytes
    from pg_stat_database sd
    where datname = current_database()
  ) dbstat
)
select
  jsonb_pretty(jsonb_build_object(
    'never_used_indexes',
    (select * from never_used_indexes_json),
    'never_used_indexes_total',
    (select row_to_json(nuit) from never_used_indexes_total as nuit),
    'rarely_used_indexes',
    (select * from rarely_used_indexes_json),
    'rarely_used_indexes_total',
    (select row_to_json(ruit) from rarely_used_indexes_total as ruit),
    'do',
    (select json_agg(dl.line) from do_lines as dl),
    'undo',
    (select json_agg(ul.line) from undo_lines as ul),
    'database_stat',
    (select * from database_stat),
    'min_index_size_bytes',
    (select min_relpages * current_setting('block_size')::numeric from const)
  ));
~~~

