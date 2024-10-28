# How to use UUID

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

截至目前 (PG16, 2023 年)，Postgres 根据 [RFC 4122](https://datatracker.ietf.org/doc/html/rfc4122) 实现了从 1 到 5 的 UUID 版本。

- 文档：[UUID Data Type](https://postgresql.org/docs/current/datatype-uuid.html)
- 附加模块 [uuid-ossp](https://postgresql.org/docs/current/uuid-ossp.html)

使用 `gen_random_uuid()` 会生成一个UUID值，它生成的是 4 版本的 UUID ([source code for PG16](https://github.com/postgres/postgres/blob/03749325d16c4215ecd6d6a6fe117d93931d84aa/src/backend/utils/adt/uuid.c#L405-L423))：

```sql
nik=# select gen_random_uuid();
           gen_random_uuid
--------------------------------------
 c027497b-c510-413b-9092-8e6c99cf9596
(1 row)

nik=# select gen_random_uuid();
           gen_random_uuid
--------------------------------------
 08e63fed-f883-45d8-9896-8f087074bff5
(1 row)
```

在标准 UUID 中，可以通过第二个连字符后的第一个字符来判断版本：

~~~bash
08e63fed-f883-4 ...  👈 this means v4
~~~

这些值以"伪随机"顺序出现。这对性能有一定的负面影响：在 B-tree 索引中，插入发生在不同位置，这通常会影响写入性能以及 Top-N 读取 (选择最新的 N 行) 的性能。

目前有一个建议，在 RFC 和 Postgres 中实现更新版本的 UUID — 7 版本中提供了一种基于时间的 UUID，其中包含毫秒级精度的时间戳，序列号，以及随机或固定位形式的额外熵。这种 UUID 不仅确保了全局唯一性，还保留了时间顺序，这对性能非常有益。

- [Commitfest: UUID v7](https://commitfest.postgresql.org/45/4388/)
- [rfc4122bis proposal](https://datatracker.ietf.org/doc/draft-ietf-uuidrev-rfc4122bis/)

UUID 值是 16 字节的 — 与 `timestamptz` 或 `timestamp` 值相同。

以下是一些解释性能方面的优秀资料：

- [The effect of Random UUID on database performance](https://twitter.com/hnasr/status/1695270411481796868) by [@hnasr](https://twitter.com/hnasr) (视频，大约 19 分钟)
- [Identity Crisis: Sequence v. UUID as Primary Key](https://brandur.org/nanoglyphs/026-ids#ulids) by [@brandur](https://twitter.com/brandur)

由于 Postgres 尚不原生支持UUID v7，目前有两个选择：

1. 在客户端生成 UUID。
2. 在 Postgres 中实现辅助函数。

对于第二种方法，这里提供了一个 [SQL function](https://gist.github.com/kjmph/5bd772b2c2df145aa645b837da7eca74) (感谢 [@DanielVerite](https://twitter.com/DanielVerite)):：

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

示例

```sql
nik=# select uuid_generate_v7();
           uuid_generate_v7
--------------------------------------
 018c1be3-e485-7252-b80f-76a71843466a
(1 row)

nik=# select uuid_generate_v7();
           uuid_generate_v7
--------------------------------------
 018c1be3-e767-76b9-93dc-23c0c48be6c7
(1 row)

nik=# select uuid_generate_v7();
           uuid_generate_v7
--------------------------------------
 018c1be3-e973-7704-82ad-5967b79cf5c4
(1 row)
```

几分钟之后：

```sql
nik=# select uuid_generate_v7();
           uuid_generate_v7
--------------------------------------
 018c1be8-5002-70ab-96c0-c96ad5afa151
(1 row)
```

一些注意事项：

1. 如果在 `ORDER BY` 子句中使用这些值，时间顺序会保持不变。

2. 在最初生成的三个值中 (几秒钟内生成)，存在一个共同前缀 `018c1be3-e`。在稍后生成的最后一个值中，有一个共同前缀 `018c1be`。

3. 注意所有值中第二个连字符后的 7：

   ~~~bash
   018c1be3-e973-7... 👈 this means v7
   ~~~

4. 该函数返回 `UUID` 类型的值，因此仍然是 16 字节 (而它的文本表示需要 36 个字符，包括连接符，这意味着加上 `VARLENA` header 的话，总共需要 40 个字节)：

```sql
nik=# select pg_column_size(gen_random_uuid());
 pg_column_size
----------------
             16
(1 row)

nik=# select pg_column_size(uuid_generate_v7());
 pg_column_size
----------------
             16
(1 row)
```

# 我见

关于 UUID 导致性能问题的案例，我也写过几篇 🔗 

- [从DBA的角度聊聊UUID的利与弊](https://mp.weixin.qq.com/s?__biz=MzUyOTAyMzMyNg==&mid=2247489659&idx=1&sn=f5cbf3851cd457f1c8093132e97b35e2&chksm=fa66304acd11b95c2a1aad0cde2048e867eaf977e86dcbdea06ab6ffb84c3d04f326bcee90c8&token=1789316483&lang=zh_CN#rd)
- [从一个案例聊聊FPI的危害](https://mp.weixin.qq.com/s?__biz=MzUyOTAyMzMyNg==&mid=2247488814&idx=1&sn=edd2be1b87259a0316f8d4ba233bc2a2&chksm=fa663d1fcd11b40992776be7f78052b8e77aa595d6773796d6e8e916859ecc521737c70c3383&token=1789316483&lang=zh_CN#rd)

为了规避前文所说的问题，你可以选择使用有序UUID：https://github.com/tvondra/sequential-uuids，以及UUID v7，v6 和 v7 ( https://github.com/fboulnois/pg_uuidv7 提供了v7的支持 ) 都有考虑可排序性，解决 UUID 应用时最常遇到的数据库性能问题。社区也在实现中。
