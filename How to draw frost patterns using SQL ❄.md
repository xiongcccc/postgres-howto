# How to draw frost patterns using SQL ❄

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

此创意和实现最初由Kirill Borovikov (kilor) 完成 — 我只是重新格式化并稍微调整了一下，扩展了字符集。

下面是查询代码：

~~~sql
with recursive t as (
  select
    0 as x,
    0 as y,
    '{"{0,0}"}'::text[] as c,
    0 as i

  union all

  (
    with z as (
      select
        dn.x,
        dn.y,
        t.c,
        t.i
      from t,
      lateral (
        select
          ((random() * 2 - 1) * 100)::integer as x,
          ((random() * 2 - 1) * 100)::integer as y
      ) as p,
      lateral (
        select *
        from (
          select
            (unnest::text[])[1]::integer as x,
            (unnest::text[])[2]::integer as y
          from unnest(t.c::text[])
        ) as t
        order by sqrt((x - p.x) ^ 2 + (y - p.y) ^ 2)
        limit 1
      ) as n,
      lateral (
        select
          n.x + dx as x,
          n.y + dy as y
        from
          generate_series(-1, 1) as dx,
          generate_series(-1, 1) as dy
        where (dx, dy) <> (0, 0)
        order by
          case
            when (p.x, p.y) = (n.x, n.y) then 0
            else abs(
              acos(
                ((p.x - n.x) * dx + (p.y - n.y) * dy)
                / sqrt((p.x - n.x) ^ 2 + (p.y - n.y) ^ 2)
                / sqrt(dx ^ 2 + dy ^ 2)
              )
            )
          end
        limit 1
      ) as dn
    )
    select
      z.x,
      z.y,
      z.c || array[z.x, z.y]::text,
      z.i + 1
    from z
    where z.i < (1 << 10)
  )
),
map as (
  select
    gx as x,
    gy as y,
    (
      select sqrt((gx - T.x) ^ 2 + (gy - T.y) ^ 2) v
      from t
      order by v
      limit 1
    ) as v
  from
    generate_series(-40, 40) as gx,
    generate_series(-30, 30) as gy
),
gr as (
  select regexp_split_to_array('@%#*+=-:. ', '') as s
)
select
  string_agg(
    coalesce(s[(v * (array_length(s, 1) - 1))::integer + 1], ' '),
    ' '
    order by x
  ) as frozen
from
  (
    select
      x,
      y,
      v::double precision / max(v) over() as v
    from map
  ) as t,
  gr
group by y
order by y;
~~~

每次它都会绘制一个新的霜花图案，此处是几个例子：

![frozen pattern 1](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0082_01.png)

![frozen pattern 2](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0082_02.png)

![frozen pattern 3](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0082_03.png)

节日快乐🎅