# UUID v7 and partitioning (TimescaleDB)

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

你们要求的来了 — 这是一个使用 UUID v7 和分区的草稿配方 (我们将使用 [@TimescaleDB](https://twitter.com/TimescaleDB))。虽然这个方法可能不是最优雅的，也不是最好的，需要一些努力才能达到高效的执行计划 (涉及到分区修剪)。如果你有其他建议或改进送我想法，请联系我。

我们将基于 [@DanielVerite](https://twitter.com/DanielVerite) 的函数来生成 UUID v7：

```sql
create or replace function uuid_generate_v7() returns uuid
as $$
  -- use random v4 uuid as starting point (which has the same variant we need)
  -- then overlay timestamp
  -- then set version 7 by flipping the 2 and 1 bit in the version 4 string
select encode(
  set_bit(
    set_bit(
      overlay(
        uuid_send(gen_random_uuid())
        placing substring(int8send(floor(extract(epoch from clock_timestamp()) * 1000)::bigint) from 3)
        from 1 for 6
      ),
      52, 1
    ),
    53, 1
  ),
  'hex')::uuid;
$$ language SQL volatile;
```

## 辅助函数：UUID v7 <-> timestamptz

接下来，我们将创建两个函数：

- `ts_to_uuid_v7` — 基于任意 `timestamptz` 值生成 UUID v7。
- `uuid_v7_to_ts` — 从现有的 UUID v7 值中提取 `timestamptz`。

注意，这种方法并不是修订版 RFC 4122 (可能很快就会完成) 的作者所鼓励的；参见 [@x4mmmmmm](https://twitter.com/x4mmmmmm) 的[讨论和评论](https://postgresql.org/message-id/flat/C80B8FDB-8D9E-48A2-82A2-48863987A1B1%40yandex-team.ru#074a05d31c9ce38bee2f8c8097877485)：

> ...据我所知，RFC 不建议从 UUID 中提取时间戳。

无论如何，让我们继续：

```sql
create extension pgcrypto;

create or replace function ts_to_uuid_v7(timestamptz) returns uuid
as $$
  select encode(
    set_bit(
      set_bit(
        overlay(
          uuid_send(gen_random_uuid())
          placing substring(int8send(floor(extract(epoch from $1) * 1000)::bigint) from 3)
          from 1 for 6
        ),
        52, 1
      ),
      53, 1
    ),
    'hex')::uuid;
$$ language SQL volatile;

create or replace function uuid_v7_to_ts(uuid_v7 uuid) returns timestamptz
as $$
  select
    to_timestamp(
      (
        'x' || substring(
          encode(uuid_send(uuid_v7), 'hex')
          from 1 for 12
        )
      )::bit(48)::bigint / 1000.0
    )::timestamptz;
$$ language sql;
```

检查函数：

```sql
test=# select now(), ts_to_uuid_v7(now() - interval '1y');
              now              |            ts_to_uuid_v7
-------------------------------+--------------------------------------
 2023-11-30 05:36:32.205093+00 | 0184c709-63cd-7bd1-99c3-a4773ab1e697
(1 row)

test=# select uuid_v7_to_ts('0184c709-63cd-7bd1-99c3-a4773ab1e697');
       uuid_v7_to_ts
----------------------------
 2022-11-30 05:36:32.205+00
(1 row)
```

忽略微秒级的丢失，让我们继续。

> 🎯 **TODO**：
>
> - 在某些情况下，我们是否需要这种精度？
> - 时区

## 超表 (Hypertable)

创建一个表，我们将使用 UUID 作为 ID，但还包括一个 `timestamptz` 列 — 在转换为分区表 (TimescaleDB中的"hypertable") 时，此列将作为分区键：

```sql
create table my_table (
  id uuid not null
    default '00000000-0000-0000-0000-000000000000'::uuid,
  payload text,
  uuid_ts timestamptz not null default clock_timestamp() -- 或使用now()，视需求而定
);
```

ID 的默认值 `00000000-...00` 是"伪值" — 它将在触发器中根据时间戳进行替换：

```sql
create or replace function t_update_uuid() returns trigger
as $$
begin
  if new.id is null or new.id = '00000000-0000-0000-0000-000000000000'::uuid then
    new.id := ts_to_uuid_v7(new.uuid_ts);
  end if;

  return new;
end;
$$ language plpgsql;

create trigger t_update_uuid_before_insert_update
before insert or update on my_table
for each row execute function t_update_uuid();
```

现在，使用 TimescaleDB 的分区：

```sql
create extension timescaledb;

select create_hypertable(
  relation := 'my_table',
  time_column_name := 'uuid_ts',
  -- !! very small interval is just for testing
  chunk_time_interval := '1 minute'::interval
);
```

## 测试数据 – 填充分区

现在插入一些测试数据 — 包括"过去"的一些数据和"当前"的一些数据：

```sql
insert into my_table(payload, uuid_ts)
select random()::text, ts
from generate_series(
  timestamptz '2000-01-01 00:01:00',
  timestamptz '2000-01-01 00:05:00',
  interval '5 second'
) as ts;

insert into my_table(payload)
select random()::text
from generate_series(1, 10000);

vacuum analyze my_table;
```

在 psql 中使用 `\d+` 检查 `my_table `的表结构，现在我们可以看到 TimescaleDB 创建的多个分区("chunks")：

```sql
test=# \d+ my_table
...
Child tables: _timescaledb_internal._hyper_2_3_chunk,
              _timescaledb_internal._hyper_2_4_chunk,
              _timescaledb_internal._hyper_2_5_chunk,
              _timescaledb_internal._hyper_2_6_chunk,
              _timescaledb_internal._hyper_2_7_chunk,
              _timescaledb_internal._hyper_2_8_chunk,
              _timescaledb_internal._hyper_2_9_chunk
```

## 测试查询 – 分区裁剪

现在需要记住的是，查询时应始终使用 `uuid_ts`，以便规划器尽可能处理较少的分区 — 但知道 `ID` 的值后，我们可以使用 `uuid_v7_to_ts()` 重构 `uuid_ts` 的值。注意，我首先禁用了顺序扫描，因为 `my_table` 中的行数太少，否则 PostgreSQL 可能会更倾向于顺序扫描而不是索引扫描：

```sql
test# set enable_seqscan = off;
SET

test=# explain select * from my_table where uuid_ts = uuid_v7_to_ts('00dc6ad0-9660-7b92-a95e-1d7afdaae659');
                                                        QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------
 Append  (cost=0.14..8.16 rows=1 width=41)
   ->  Index Scan using _hyper_5_11_chunk_my_table_uuid_ts_idx on _hyper_5_11_chunk  (cost=0.14..8.15 rows=1 width=41)
         Index Cond: (uuid_ts = '2000-01-01 00:01:00+00'::timestamp with time zone)
(3 rows)

test=# explain select * from my_table
  where uuid_ts >= uuid_v7_to_ts('018c1ecb-d3b7-75b1-add9-62878b5152c7')
  order by uuid_ts desc limit 10;
                                                            QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------
 Limit  (cost=0.29..1.17 rows=10 width=41)
   ->  Custom Scan (ChunkAppend) on my_table  (cost=0.29..11.49 rows=126 width=41)
         Order: my_table.uuid_ts DESC
         ->  Index Scan using _hyper_5_16_chunk_my_table_uuid_ts_idx on _hyper_5_16_chunk  (cost=0.29..11.49 rows=126 width=41)
               Index Cond: (uuid_ts >= '2023-11-30 05:55:23.703+00'::timestamp with time zone)
(5 rows)
```

分区裁剪已生效，尽管在各种查询中使用可能需要一定的努力 (调整)，但确实有效。

## 附录

另请阅读 [@jamessewell](https://twitter.com/jamessewell) 的以下评论 (原评论位于[这里](https://x.com/jamessewell/status/1730125437903450129))：

> 如果使用以下命令更新 create_hypertable 调用：
>
> ```sql
> time_column_name => 'id'
> time_partitioning_func => 'uuid_v7_to_ts'
> ```
> 
> 那么你可以删除 `uuid_ts` 列和触发器！
> 
> ```sql
>SELECT * FROM my_table WHERE id = '018c1ecb-d3b7-75b1-add9-62878b5152c7';
> ```
>
> 这就会生效 🪄