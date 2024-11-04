# How to find redundant indexes

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

今天让我们讨论如何查找和清理冗余索引。

## 什么是冗余索引？

冗余索引是指一个表上存在多个索引，它们的作用相同，或者其中一个索引可以满足所有其他索引的查询需求。两个基本的例子：

1. **重复索引**：两个或多个具有完全相同的列且顺序相同的索引显然是冗余的。
2. **重叠索引**：当你有一个多列索引，例如在列 `(a, b)` 上创建了索引，然后又在 `(a)` 上创建了另一个索引。对于仅在列 `a` 上进行过滤的查询，`(a, b)` 上的索引就足够了，因此在 `(a)` 上的索引可能是不必要的。

注意，`(a)` 对于 `(a, b)` 是冗余的，但对于 `(b, a)` 则不是。同样，`(b)` 对于 `(a, b)` 也不是冗余的。

此外，我们还需要假设：

- 唯一索引在此类分析中不应被考虑，因为它们有特殊用途。
- 表达式索引遵循相同的规则 — 我们只需将每个表达式视为列，表达式的值应完全匹配。
- 在部分索引的情况下，条件应完全匹配 (此规则可以根据某些假设进行调整，但我们不会这样做)。
- 我们不考虑覆盖索引 (🎯 **TODO：**也将其包括在内)。

## 为什么要清理冗余索引

我们之前讨论[未使用索引的 6 个原因](https://postgres-howto.cn/#/./docs/75)同样适用于此处。

## 清理冗余索引的一般算法

1. 使用下面提供的查询，识别冗余索引集。只需要分析集群中的一个节点即可，例如主节点。因为此分析仅基于静态信息 (数据库结构)，所以统计信息是何时重置的并不重要。

2. 对于每个被认为是冗余的索引，进行手动分析以避免错误。如果不完全确定，请将该索引从考虑中移除。

   > 译者注：即，将该索引从考虑移除的列表中删除

3. 可选地，考虑在实际删除索引之前使用"软删除"技术，如 [Day 53: Index maintenance](https://postgres-howto.cn/#/./docs/53) 中所述。

4. 经过适当的分析和测试后，一次删除一个索引，同时密切关注数据库性能指标，确保没有预期之外的执行计划变化。要删除索引，请使用 `DROP INDEX CONCURRENTLY`。

## 查找冗余索引的查询

这是来自 [postgres-checkup](https://gitlab.com/postgres-ai/postgres-checkup/-/blob/a6c8b6ae0772d5f7afb0a7004a740f6643a31aa2/resources/checks/H004_redundant_indexes.sh) 的一个查询，它以 JSON 的形式生成结果，其中包括使用 `DROP INDEX CONCURRENTLY` 删除索引的命令。

```sql
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
    and not i.indisunique
    and conkey is not null
    and ci.relpages > (select min_relpages from const)
    and si.idx_scan < 10
), index_data as ( -- Redundant indexes
  select
    *,
    indkey::text as columns,
    array_to_string(indclass, ', ') as opclasses
  from pg_index i
  join pg_class ci on ci.oid = i.indexrelid and ci.relkind = 'i'
  where indisvalid = true and ci.relpages > (select min_relpages from const)
), redundant_indexes as (
  select
    i2.indexrelid as index_id,
    tnsp.nspname AS schema_name,
    trel.relname AS table_name,
    pg_relation_size(trel.oid) as table_size_bytes,
    irel.relname AS index_name,
    am1.amname as access_method,
    (i1.indexrelid::regclass)::text as reason,
    i1.indexrelid as reason_index_id,
    pg_get_indexdef(i1.indexrelid) main_index_def,
    pg_size_pretty(pg_relation_size(i1.indexrelid)) main_index_size,
    pg_get_indexdef(i2.indexrelid) index_def,
    pg_relation_size(i2.indexrelid) index_size_bytes,
    s.idx_scan as index_usage,
    quote_ident(tnsp.nspname) as formated_schema_name,
    coalesce(nullif(quote_ident(tnsp.nspname), 'public') || '.', '') || quote_ident(irel.relname) as formated_index_name,
    quote_ident(trel.relname) AS formated_table_name,
    coalesce(nullif(quote_ident(tnsp.nspname), 'public') || '.', '') || quote_ident(trel.relname) as formated_relation_name,
    i2.opclasses
  from index_data as i1
  join index_data as i2 on
    i1.indrelid = i2.indrelid -- the same table
    and i1.indexrelid <> i2.indexrelid -- NOT the same index
  join pg_opclass op1 on i1.indclass[0] = op1.oid
  join pg_opclass op2 on i2.indclass[0] = op2.oid
  join pg_am am1 on op1.opcmethod = am1.oid
  join pg_am am2 on op2.opcmethod = am2.oid
  join pg_stat_user_indexes as s on s.indexrelid = i2.indexrelid
  join pg_class as trel on trel.oid = i2.indrelid
  join pg_namespace as tnsp on trel.relnamespace = tnsp.oid
  join pg_class as irel on irel.oid = i2.indexrelid
  where
    not i2.indisprimary -- index 1 is not PK
    and not ( -- skip if index1 is (primary or uniq) and is NOT (primary and uniq)
      i2.indisunique and not i1.indisprimary
    )
    and am1.amname = am2.amname -- same access type
    and i1.columns like (i2.columns || '%') -- index 2 includes all columns from index 1
    and i1.opclasses like (i2.opclasses || '%')
    -- index expressions is same
    and pg_get_expr(i1.indexprs, i1.indrelid) is not distinct from pg_get_expr(i2.indexprs, i2.indrelid)
    -- index predicates is same
    and pg_get_expr(i1.indpred, i1.indrelid) is not distinct from pg_get_expr(i2.indpred, i2.indrelid)
), redundant_indexes_fk as (
  select
    ri.*,
    (
      select count(1)
      from fk_indexes fi
      where
        fi.fk_table_ref = ri.table_name
        and fi.opclasses like (ri.opclasses || '%')
     ) > 0 as supports_fk
  from redundant_indexes ri
), redundant_indexes_tmp_num as ( -- Cut recursive links
  select row_number() over () num, rig.*
  from redundant_indexes_fk rig
), redundant_indexes_tmp_links as (
  select
    ri1.*,
    ri2.num as r_num
  from redundant_indexes_tmp_num ri1
  left join redundant_indexes_tmp_num ri2 on
    ri2.reason_index_id = ri1.index_id
    and ri1.reason_index_id = ri2.index_id
), redundant_indexes_tmp_cut as (
  select *
  from redundant_indexes_tmp_links
  where num < r_num or r_num is null
), redundant_indexes_cut_grouped as (
  select
    distinct(num),
    *
  from redundant_indexes_tmp_cut
  order by index_size_bytes desc
), redundant_indexes_grouped as (
  select
    index_id,
    schema_name,
    table_name,
    table_size_bytes,
    index_name,
    access_method,
    string_agg(distinct reason, ', ') as reason,
    string_agg(main_index_def, ', ') as main_index_def,
    string_agg(main_index_size, ', ') as main_index_size,
    index_def,
    index_size_bytes,
    index_usage,
    formated_index_name,
    formated_schema_name,
    formated_table_name,
    formated_relation_name,
    supports_fk
  from redundant_indexes_cut_grouped
  group by
    index_id,
    table_size_bytes,
    schema_name,
    table_name,
    index_name,
    access_method,
    index_def,
    index_size_bytes,
    index_usage,
    formated_index_name,
    formated_schema_name,
    formated_table_name,
    formated_relation_name,
    supports_fk
  order by index_size_bytes desc
), redundant_indexes_num as (
  select row_number() over () num, rig.*
  from redundant_indexes_grouped rig
), redundant_indexes_json as (
  select
    json_object_agg(coalesce(rin.schema_name, 'public') || '.' || rin.index_name, rin) as json
  from redundant_indexes_num rin
), redundant_indexes_total as (
  select
    sum(index_size_bytes) as index_size_bytes_sum,
    sum(table_size_bytes) as table_size_bytes_sum
  from redundant_indexes_grouped
), do_lines as (
  select
    format(
      'DROP INDEX CONCURRENTLY %s; -- %s, %s, table %s',
      formated_index_name,
      pg_size_pretty(index_size_bytes)::text,
      reason,
      formated_relation_name
    ) as line
  from redundant_indexes_grouped
  order by table_name, index_name
), undo_lines as (
  select
    replace(
      format('%s; -- table %s', index_def, formated_relation_name),
      'CREATE INDEX',
      'CREATE INDEX CONCURRENTLY'
    ) as line
  from redundant_indexes_grouped
  order by table_name, index_name
), database_stat as (
  select row_to_json(dbstat)
  from (
    select
      sd.stats_reset::timestamptz(0),
      age(
        date_trunc('minute',now()),
        date_trunc('minute',sd.stats_reset)
      ) as stats_age,
      ((extract(epoch from now()) - extract(epoch from sd.stats_reset))/86400)::int as days,
      (select pg_database_size(current_database())) as database_size_bytes
    from pg_stat_database sd
    where datname = current_database()
  ) as dbstat
) -- final result
select
  jsonb_pretty(jsonb_build_object(
    'redundant_indexes',
    (select * from redundant_indexes_json),
    'redundant_indexes_total',
    (select row_to_json(rit) from redundant_indexes_total as rit),
    'do',
    (select json_agg(dl.line) from do_lines as dl),
    'undo',
    (select json_agg(ul.line) from undo_lines as ul),
    'database_stat',
    (select * from database_stat),
    'min_index_size_bytes',
    (select min_relpages * current_setting('block_size')::numeric from const)
  ));
```
