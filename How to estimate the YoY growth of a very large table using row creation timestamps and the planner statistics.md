# How to estimate the YoY growth of a very large table using row creation timestamps and the planner statistics

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

假设我们有一个创建于多年前的 10 TiB 的大表，无论分区与否，结构如下：

```sql
create table t (
  id int8 primary key,   -- of course, not int4
  created_at timestamptz default now(),
  ...
);
```

我们需要快速了解这个表的年度增长情况，假设没有删除行 (或删除的数量可以忽略不计)。因此，我们只需统计每年的行数。

一个简便方法是：

```sql
select
  date_trunc('year', created_at) as year,
  count(*)
from t
group by year
order by year;
```

然而，对于 10 TiB 的表，这样的分析可能需要等待数小时甚至数天才能完成。

以下是一种快速但不精确的方法来获取每年的行数 (假设表具有最新的统计数据；如果不确定，请先在表上运行 `ANALYZE`)：

~~~sql
do $$
declare
  table_fqn text := 'public.t';
  year_start int := 2000;
  year_end int := extract(year from now())::int;
  year int;
  explain_json json;
begin
  for year in year_start..year_end loop
    execute format(
      $e$
        explain (format json) select *
        from %s
        where created_at
        between '%s-01-01' and '%s-12-31'
      $e$,
      table_fqn,
      year,
      year
    ) into explain_json;

    raise info 'Year: %, Estimated rows: %',
      year,
      explain_json->0->'Plan'->>'Plan Rows';
  end loop;
end $$;
~~~