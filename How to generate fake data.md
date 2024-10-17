# How to generate fake data

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

## 简单数字

从 1 到 5 的连续数字：

```sql
nik=# select i from generate_series(1, 5) as i;
 i
---
 1
 2
 3
 4
 5
(5 rows)
```

从 0 到 100 的 5 个随机 `BIGINT` 数字：

```sql
nik=# select (random() * 100)::int8 from generate_series(1, 5);
 int8
------
   85
   61
   44
   70
   16
(5 rows)
```

注意，根据[文档](https://postgresql.org/docs/current/functions-math.html#FUNCTIONS-MATH-RANDOM-TABLE)，`random()` 

> 使用的是确定性的伪随机数生成器。它的速度很快，但不适合用于加密应用。
>
> uses a deterministic pseudo-random number generator. It is fast but not suitable for cryptographic applications...

因此，不应将其用于生成令牌或密码 (这种情况下应使用名为 `pgcrypto` 的库)。但它可以用于生成纯随机数据 (不用于混淆)。

从 Postgres 16 开始，还有 [random_normal(mean, stddev)](https://postgresql.org/docs/16/functions-math.html#FUNCTIONS-MATH-RANDOM-TABLE)：

>Returns a random value from the normal distribution with the given parameters; `mean` defaults to 0.0 and `stddev` defaults to 1.0
>
>返回具有给定参数的正态分布的随机值；`mean` 默认为 0.0，`stddev` 默认为 1.0

生成 100 万个数字并检查它们的分布：

```sql
nik=# with data (r) as (
  select random_normal()
  from generate_series(1, 1000000)
)
select
  width_bucket(r, -3, 3, 5) as bucket,
  count(*) as frequency
from data
group by bucket
order by bucket;

 bucket | frequency
--------+-----------
      0 |      1374
      1 |     34370
      2 |    238786
      3 |    450859
      4 |    238601
      5 |     34627
      6 |      1383
(7 rows)
```

再次提醒，`random()` 和 `random_normal()` 都不应用作加密性强的随机数生成器 — 为此应使用 `pgcrypto`。否则，知道一个值后，可以"猜测"下一个值：

```sql
nik=# set seed to 0.1234;
SET

nik=# select random_normal();
    random_normal
----------------------
 -0.32020779450641174
(1 row)

nik=# select random_normal();
   random_normal
--------------------
 0.8995226247227294
(1 row)

nik=# set seed to 0.1234; -- start again
SET

nik=# select random_normal();
    random_normal
----------------------
 -0.32020779450641174
(1 row)

nik=# select random_normal();
   random_normal
--------------------
 0.8995226247227294
(1 row)
```

## 时间戳、日期、间隔

2024 年 1 月的时间戳 (带时区)，从 2024-01-01 开始，每隔 7 天：

```sql
nik=# show timezone;
      TimeZone
---------------------
 America/Los_Angeles
(1 row)

nik=# select i from generate_series(timestamptz '2024-01-01', timestamptz '2024-01-31', interval '7 day') i;
           i
------------------------
 2024-01-01 00:00:00-08
 2024-01-08 00:00:00-08
 2024-01-15 00:00:00-08
 2024-01-22 00:00:00-08
 2024-01-29 00:00:00-08
(5 rows)
```

为上一周生成 3 个随机时间戳 (常用于填充如 `created_at` 等字段)：

```sql
nik=# select
  date_trunc('week', now())
  - interval '7 day'
  + format('%s day', (random() * 7))::interval
from generate_series(1, 3);

           ?column?
-------------------------------
 2023-10-31 00:50:59.503352-07
 2023-11-03 11:25:39.770384-07
 2023-11-03 13:43:27.087973-07
(3 rows)
```

生成年龄在 18-100 岁的人的随机出生日期：

```sql
nik=# select
  (
    now()
    - format('%s day', 365 * 18)::interval
    - format('%s day', 365 * random() * 82)::interval
  )::date;
    date
------------
 1954-01-17
(1 row)
```

## 伪单词

生成由 2 到 12 个小写拉丁字母组成的伪单词：

```sql
nik=# select string_agg(chr((random() * 25)::int + 97), '')
from generate_series(1, 2 + (10 * random())::int);
 string_agg
------------
 yegwrsl
(1 row)

nik=# select string_agg(chr((random() * 25)::int + 97), '')
from generate_series(1, 2 + (10 * random())::int);
 string_agg
------------
 wusapjx
(1 row)
```

生成包含 5 到 10 个伪单词的"句子"：

```sql
nik=# select string_agg(w, ' ')
from
  generate_series(1, 5) as i,
  lateral (
    select string_agg(chr((random() * 25)::int + 97), ''), i
    from generate_series(1, 2 + (10 * random())::int + i - i)
  ) as words(w);
             string_agg
-------------------------------------
 uvo bwp kcypvcnctui tn demkfnxruwxk
(1 row)
```

注意 `LATERAL` 和引用外部生成器的"空闲"方式 (`i` 和 `+i - i`)，以确保每次迭代都生成新随机值。

## 正常单词、名字、电子邮件、社会安全号码等 (Faker)

[Faker](https://faker.readthedocs.io/en/master/) 是一个 Python 库，用于生成虚假数据，如姓名、地址、电话号码等。其他语言的 Faker 版本有：

- [Faker for Ruby](https://github.com/faker-ruby/faker)
- [Faker for Go](https://github.com/go-faker/faker)
- [Faker for Java](https://github.com/DiUS/java-faker)
- [Faker for Rust](https://github.com/cksac/fake-rs)
- [Faker for JavaScript](https://github.com/faker-js/faker)

你可以通过以下几种方式使用 Python 的 Faker：

1. 使用 Python 程序和 Postgres 连接 (常规方法，但适用于任何 Postgres 实例，包括 RDS)。
2. [postgresql_faker](https://gitlab.com/dalibo/postgresql_faker/)
3. PL/Python 函数。

此处我们展示如何使用第三种方式，使用 `plpython3u` 的"不受信任"版本 ([Day 47: How to install Postgres 16 with plpython3u](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0047_how_to_install_postgres_16_with_plpython3u.md); 不适用于托管的 Postgres 服务，如 RDS；注意这种情况下"受信任"版本应该也适合)。

```sql
nik=# create or replace function generate_random_sentence(
    min_length int,
    max_length int
  ) returns text
  as $$
    from faker import Faker
    import random

    if min_length > max_length:
      raise ValueError('min_length > max_length')

    fake = Faker()

    sentence_length = random.randint(min_length, max_length)

    return ' '.join(fake.words(nb=sentence_length))
  $$ language plpython3u;
CREATE FUNCTION

nik=# select generate_random_sentence(7, 15);
                            generate_random_sentence
---------------------------------------------------------------------------------
 operation day down forward foreign left your anything clear age seven memory as
(1 row)
```

生成名字、电子邮件和社会安全号码的函数：

```sql
nik=# create or replace function generate_faker_data(
    data_type text,
    locale text default 'en_US'
)
returns text as $$
  from faker import Faker

  fake = Faker(locale)

  if data_type == 'email':
    result = fake.email()
  elif data_type == 'lastname':
    result = fake.last_name()
  elif data_type == 'firstname':
    result = fake.first_name()
  elif data_type == 'ssn':
    result = fake.ssn()
  else:
    raise Exception('Invalid type')

  return result
$$ language plpython3u;
CREATE FUNCTION

nik=# select
  generate_faker_data('firstname', locale) as firstname,
  generate_faker_data('lastname', locale) as lastname,
  generate_faker_data('ssn') as "SSN";
CREATE FUNCTION

nik=# select
  locale,
  generate_faker_data('firstname', locale) as firstname,
  generate_faker_data('lastname', locale) as lastname
from
  (values ('en_US'), ('uk_UA'), ('it_IT')) as _(locale);
 locale | firstname | lastname
--------+-----------+-----------
 en_US  | Ashley    | Rodgers
 uk_UA  | Анастасія | Матвієнко
 it_IT  | Carolina  | Donatoni
(3 rows)

nik=# select generate_faker_data('ssn');
 generate_faker_data
---------------------
 008-47-2950
(1 row)

nik=# select generate_faker_data('email', 'es') from generate_series(1, 5);
   generate_faker_data
-------------------------
 isidoro42@example.net
 anselma04@example.com
 torreatilio@example.com
 natanael39@example.org
 teodosio79@example.net
(5 rows)
```