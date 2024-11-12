# How to find int4 PKs with out-of-range risks in a large database

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

如今，现代 ORM 比如 Rails 或 Django 使用 int8 (bigint) 类型的主键 (PK)。然而，在旧项目中，可能存在使用 int4 (integer, int, serial) 主键的老表，这些表可能已增长并存在 int4 溢出的风险 — int4 的最大值为 [2,147,483,647](https://postgresql.org/docs/current/datatype-numeric.html)，且不停机将主键从 int4 转换为 int8 并不是一项简单的任务 (TODO：在另一篇使用指南中介绍转换过程)。

以下是快速检查是否存在 int2 或 int4 PK 表的方法，并了解在每种情况下已使用了多少"容量" (来自postgres-checkup 的查询)：

```sql
do $$
declare
  min_relpages int8 := 0; -- in very large DBs, skip small tables by setting this to 100
  rec record;
  out text := '';
  out1 json;
  i numeric := 0;
  val int8;
  ratio numeric;
  sql text;
begin
  for rec in
    select
      c.oid,
      spcname as tblspace,
      nspname as schema_name,
      relname as table_name,
      t.typname,
      pg_get_serial_sequence(format('%I.%I', nspname, relname), attname) as seq,
      min(attname) as attname
    from pg_index i
    join pg_class c on c.oid = i.indrelid
    left join pg_tablespace tsp on tsp.oid = reltablespace
    left join pg_namespace n on n.oid = c.relnamespace
    join pg_attribute a on
      a.attrelid = i.indrelid
      and a.attnum = any(i.indkey)
    join pg_type t on t.oid = atttypid
    where
      i.indisprimary
      and (
        c.relpages >= min_relpages
        or pg_get_serial_sequence(format('%I.%I', nspname, relname), attname) is not null
      ) and t.typname in ('int2', 'int4')
      and nspname <> 'pg_toast'
      group by 1, 2, 3, 4, 5, 6
      having count(*) = 1 -- skip PKs with 2+ columns (TODO: analyze them too)
  loop
    raise debug 'table: %', rec.table_name;

    if rec.seq is null then
      sql := format('select max(%I) from %I.%I;', rec.attname, rec.schema_name, rec.table_name);
    else
      sql := format('select last_value from %s;', rec.seq);
    end if;

    raise debug 'sql: %', sql;
    execute sql into val;

    if rec.typname = 'int4' then
      ratio := (val::numeric / 2^31)::numeric;
    elsif rec.typname = 'int2' then
      ratio := (val::numeric / 2^15)::numeric;
    else
      assert false, 'unreachable point';
    end if;

    if ratio > 0.1 then -- report only if > 10% of capacity is reached
      i := i + 1;

      out1 := json_build_object(
        'table', (
          coalesce(nullif(quote_ident(rec.schema_name), 'public') || '.', '')
          || quote_ident(rec.table_name)
        ),
        'pk', rec.attname,
        'type', rec.typname,
        'current_max_value', val,
        'capacity_used_percent', round(100 * ratio, 2)
      );

      raise debug 'cur: %', out1;

      if out <> '' then
        out := out || e',\n';
      end if;

      out := out || format('  "%s": %s', rec.table_name, out1);
    end if;
  end loop;

  raise info e'{\n%\n}', out;
end;
$$ language plpgsql;
```

输出示例

```sql
INFO:  {
  "oldbig": {"table" : "oldbig", "pk" : "id", "type" : "int4", "current_max_value" : 2107480000, "capacity_used_percent" : 98.14},
  "oldbig2": {"table" : "oldbig2", "pk" : "id", "type" : "int4", "current_max_value" : 1107480000, "capacity_used_percent" : 51.57}
}
```