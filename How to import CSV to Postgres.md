# How to import CSV to Postgres

## 我们将使用的示例CSV

在下面的导入示例中，我们将使用以下代码片段，将 Postgres 中长时间运行的事务信息收集到 CSV 文件中：

```bash
header='header'

while sleep 1; do
  psql -XAtc "
      copy (
        with samples as (
          select
            clock_timestamp(),
            clock_timestamp() - xact_start as xact_duration,
            *
          from pg_stat_activity
        )
        select *
        from samples
        where xact_duration > interval '1 minute'
        order by xact_duration desc
      ) to stdout delimiter ',' csv ${header}
    " \
  2> >(ts | tee -a long_tx_$(date +%Y%m%d).err.log) \
  | tee -a long_tx_$(date +%Y%m%d).csv

  header=''
done
```

这是源自 [Ad-hoc monitoring how-to](https://twitter.com/samokhvalov/status/1710510207875559844) 中相关代码片段的修改版本，并进行了以下调整：

- 将错误写入一个单独的文件 (\*\*\*.err.log)，并在该文件的每一行前加上时间戳。对于常规记录(***.csv)，我们不希望时间戳前缀破坏 CSV 格式 — 因为这些记录的时间戳已经由 Postgres 通过 `clock_timestamp()` 生成。
- 在 CSV 文件中，我们需要一个 header，但不希望重复。

------

## 方法1：将CSV快照导入一张普通表

要将 CSV 文件的快照导入 Postgres，我们需要创建一个表。有两种选择：

**选项 1：**确切的表定义。如果我们确切知道表的结构 (在本例中我们确实知道)，我们便可以直接利用这一点 — 我通常使用原始查询，并加上 `LIMIT 0` (可选地省略 `WHERE` 和 `ORDER BY` 子句)：

```sql
create table slow_tx_from_csv_1 as
with samples as (
  select
    clock_timestamp(),
    clock_timestamp() - xact_start as xact_duration,
    *
  from pg_stat_activity
)
select *
from samples
limit 0;
```

(你也可以使用 `WITH NO DATA` 来替代 `LIMIT 0`，但我个人经常忘记这种语法。)

这种方法提供了明确的表结构 (列类型与 CSV 中保存的原始结构匹配)，但在某些情况下可能不太适用 (例如，当我们不知道 CSV 的确切创建方式时)。

**选项 2：**带有文本列的任意结构。这种方法非常灵活，适用于我们不知道结构或者不想花时间去弄清楚的情况。在这种情况下，我们利用 CSV 文件的第一行是 header 的事实，创建一个由 "text" 列组成的表：

```bash
psql -c "create table slow_tx_from_csv_2($(
  head -n 1 long_tx_$(date +%Y%m%d).log.csv \
  | tr '[:upper:]' '[:lower:]' \
  | sed -e 's/[^,a-zA-Z0-9\r]/_/g' \
  | sed -e 's/,/ text, /g' \
) text)"
```

现在，我们可以将当前 CSV 内容的快照导入到两个文件中 (以下是导入 `slow_tx_from_csv_2` 的示例)：

```bash
psql -c "copy slow_tx_from_csv_2 from '$(pwd)/long_tx_$(date +%Y%m%d).csv' delimiter ',' csv header"
```

请注意，这里我们使用在服务器端运行的 SQL 命令 COPY，并且需要位于运行 Postgres 的服务器上的 CSV 文件的完整路径。还有 psql 命令 `\copy`，允许导入位于客户端的 CSV 文件 (如果客户端和服务器是不同的，在本例中是相同的，所以我们可以使用任一命令)。文档参考：

- [server-side COPY](https://postgresql.org/docs/current/sql-copy.html)
- [psql's \copy](https://postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMANDS-COPY)

需要注意的是，当使用"灵活"表格式 (仅 text 列) 时 — 此处是 `slow_tx_from_csv_2`，我们需要记得进行类型转换。例如：

```sql
nik=# select clock_timestamp, pid, query from slow_tx_from_csv_2 order by clock_timestamp::timestamptz desc limit 2;
       clock_timestamp        |  pid  | query
-------------------------------+-------+--------
2023-10-14 19:15:27.984335-07 | 22957 | begin;
2023-10-14 19:15:26.933159-07 | 22957 | begin;
(2 rows)
```

对于定义明确的表结构 `slow_tx_from_csv_1` 则不需要进行类型转换：

```sql
nik=# select clock_timestamp, pid, query from slow_tx_from_csv_1 order by clock_timestamp desc limit 2;
       clock_timestamp        |  pid  | query
-----------------------------+-------+--------
2023-10-14 19:15:19.567062-07 | 22957 | begin;
2023-10-14 19:15:18.515422-07 | 22957 | begin;
(2 rows)
```

------

## 方法2：通过file_fdw实时查询CSV数据

当我们需要通过 SQL 实时查询 CSV 文件数据而无需加载快照时，可以使用 [file_fdw](https://postgresql.org/docs/current/file-fdw.html)。

这种方法有一个明显的优势 (实时数据！)，但也有一些显而易见的缺点：

- 它是只读的 (虽然你可以很容易地使用 `create table as select from ...` 创建一个快照)。
- 数据不受你可能拥有的备份系统所保护，存储也不可靠 (文件)。
- 性能限制：你不能创建索引来加速查询，所以对于大量数据它的表现并不好 (不过，你可以创建快照或物化视图，并在其上创建索引)。
- `file_fdw` 不支持某些 `COPY` 选项，例如 `FORCE_QUOTE` 选项。
- 在某些托管的 Postgres 服务 (如 RDS) 上无法使用 `file_fdw`。
- 要使用 `file_fdw`，需要超级用户权限或 `pg_read_server_files` 角色的权限。

首先，准备 `file_fdw`：

```sql
create extension file_fdw;
create server "import" foreign data wrapper file_fdw;
```

接下来，我们准备表 (使用与上面相同的 "flexible but text-only columns" 的方法)。但这次它不是常规的表，而是提供只读 SQL 接口的"外部表"，用于访问 CSV 文件：

```bash
psql -c "create foreign table slow_tx_csv_live($(
  head -n 1 long_tx_$(date +%Y%m%d).csv \
  | tr '[:upper:]' '[:lower:]' \
  | sed -e 's/[^,a-zA-Z0-9\r]/_/g' \
  | sed -e 's/,/ text, /g'
) text) server import options (
  filename '$(pwd)/long_tx_$(date +%Y%m%d).csv',
  format 'csv',
  header 'on'
)"
```

就是这样，我们不需要加载任何东西，数据已经"存在"了 — 并且我们可以实时观察它的变化 (不要忘记类型转换)：

```sql
nik=# select clock_timestamp, pid, query
from slow_tx_csv_live
order by clock_timestamp::timestamptz desc
limit 2 \watch 1
     Sat Oct 14 19:48:20 2023 (every 1s)

       clock_timestamp        |  pid  | query
-------------------------------+-------+-------
2023-10-14 19:48:19.598753-07 | 22957 | begin;
2023-10-14 19:48:18.554661-07 | 22957 | begin;
(2 rows)

     Sat Oct 14 19:48:21 2023 (every 1s)

       clock_timestamp        |  pid  | query
-------------------------------+-------+--------
2023-10-14 19:48:20.65133-07  | 22957 | begin;
2023-10-14 19:48:19.598753-07 | 22957 | begin;
(2 rows)

     Sat Oct 14 19:48:22 2023 (every 1s)

       clock_timestamp        |  pid  | query
-------------------------------+-------+--------
2023-10-14 19:48:21.706331-07 | 22957 | begin;
2023-10-14 19:48:20.65133-07  | 22957 | begin;
(2 rows)
```

------

## 额外示例 (给感到无聊的人)

如果你对仅有 Postgres 的示例感到无聊，这里有我之前使用的 Pokemon 数据示例，这些数据是从一个[公开的 Google 电子表格](https://gist.github.com/NikolayS/a819f139c37e0d54ad4a4ca70764f225)中获取的：

1. 在数据库中安装 `file_fdw`：

```sql
create extension file_fdw;
create server import foreign data wrapper file_fdw;
```

2. 下载 CSV 文件：

```bash
wget -O pokemons.csv \
"https://docs.google.com/spreadsheets/d/14mIpk_ceBWVnjc1cPAD7AWeXkE8-t729pcLqwcX_Iik/export?format=csv"
```

3. 创建一个外部表，将其结构定义为基于 CSV 文件第一行 (header) 的一组纯文本列，并将其连接到 CSV 文件：

```bash
psql -c "create foreign table pokemon_imported($(
  head -n 1 pokemons.csv \
  | tr '[:upper:]' '[:lower:]' \
  | sed -e 's/[^,a-zA-Z0-9\r]/_/g' \
  | sed -e 's/,/ text, /g'
) text) server import options (
  filename '$(pwd)/pokemons.csv',
  format 'csv',
  header 'on'
)"
```

4. 现在它应该可以工作了；进行测试 (在 psql 中)：

```sql
nik=# select generation, pokemon, type_i, max_hp
from pokemon_imported
order by max_hp::int desc
limit 5;
generation |  pokemon  | type_i  | max_hp
------------+-----------+---------+--------
2          | Blissey   | Normal  | 415
1          | Chansey   | Normal  | 407
2          | Wobbuffet | Psychic | 312
3          | Wailord   | Water   | 281
5          | Alomomola | Water   | 273
(5 rows)
```