# How to format SQL (SQL style guide)

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

有多种格式化 SQL 的方法，其中主要两种是：

1. 传统方式：所有关键字大写，格式化以中央空白管道为基础，这种方式被 Joe Celko 的《SQL 编程风格》一书推荐。其中一种版本可以在 [SQL Style Guide](https://www.sqlstyle.guide/) 中找到。比如：

```sql
SELECT a.title, a.release_date, a.recording_date
  FROM albums AS a
 WHERE a.title = 'Charcoal Lane'
    OR a.title = 'The New Danger';
```

我个人认为这种风格显得过于老派、不便且繁琐。我更喜欢另一种方式：

2. 现代方式：关键字通常 (并非总是) 使用小写，并采用类似编程语言中常用的左对齐分层缩进风格。示例：

```sql
select
    a.title,
    a.release_date,
    a.recording_date
from albums as a
where
    a.title = 'Charcoal Lane'
    or a.title = 'The New Danger';
```

在这里，我们将讨论后一种"现代"方法的一个版本，来源于 Mozilla 的 [SQL 样式指南](https://docs.telemetry.mozilla.org/concepts/sql_style.html)。

与 Mozilla 指南的关键区别在于，这里我们使用小写 SQL。原因很简单：当 SQL 嵌入其他语言时，大写的 SQL 更有意义，有助于将其与周围的代码区分开来。

但如果你在各种 SQL 编辑器中单独处理大查询文本、执行优化任务等，使用大写关键字就不太方便了。小写关键字更易于输入。最后，不停地"对着数据库大喊大叫"并不是一个好主意。

同时，当在自然语言文本中使用 SQL 关键字时，使用大写仍然可能是合理的，例如：

>.. using multiple JOINs ..

## 1) 保留字

保留关键字始终使用小写，比如 `select`、`where` 或 `as`。

## 2) 变量名称

1. 使用一致且具描述性的标识符和名称。

2. 使用带下划线的小写名称，例如 `first_name`。**不要使用驼峰命名法。**
3. 函数如 `cardinality()`、`array_agg()` 和 `substr()` 是标识符，应该像变量名一样对待。
4. 名称必须以字母开头，不能以下划线结尾。
5. 变量名中仅使用字母、数字和下划线。

## 3) 显式表达

在显式或隐式语法之间进行选择时，优先使用显式语法。

### 3.1) 别名

在为变量或表名设置别名时，始终包括 `as` 关键字，显式表达更易于阅读。

✅ 推荐：

```sql
select date(t.created_at) as day
from telemetry as t
limit 10;
```

❌ 不推荐：

```sql
select date(t.created_at) day
from telemetry t
limit 10;
```

### 3.2) JOIN

始终包括连接类型，而不是依赖默认的连接方式。

✅ 推荐：

```sql
select
    submission_date,
    experiment.key as experiment_id,
    experiment.value as experiment_branch,
    count(*) as count
from
    telemetry.clients_daily
cross join
    unnest(experiments.key_value) as experiment
where
    submission_date > '2019-07-01'
    and sample_id = '10'
group by
    submission_date,
    experiment_id,
    experiment_branch;
```

❌ 不推荐：

```sql
select
    submission_date,
    experiment.key as experiment_id,
    experiment.value as experiment_branch,
    count(*) as count
from
    telemetry.clients_daily,
    unnest(key_value) as experiment -- 隐式JOIN
where
    submission_date > '2019-07-01'
    and sample_id = '10'
group by
    1, 2, 3; -- 隐式分组列名
```

### 3.3) 分组列

避免使用隐式分组列名。

✅ 推荐：

```sql
select state, backend_type, count(*)
from pg_stat_activity
group by state, backend_type
order by state, backend_type;
```

🆗 可接受：

```sql
select
    date_trunc('minute', xact_start) as xs_minute,
    count(*)
from pg_stat_activity
group by 1
order by 1;
```

❌ 不推荐：

```sql
select state, backend_type, count(*)
from pg_stat_activity
group by 1, 2
order by 1, 2;
```

## 4) 左对齐根关键字

所有根关键字应在同一字符边界上开始。

✅ 推荐：

```sql
select
    client_id,
    submission_date
from main_summary
where
    sample_id = '42'
    and submission_date > '20180101'
limit 10;
```

❌ 不推荐：

```sql
	select client_id,
         submission_date
    from main_summary
   where sample_id = '42'
     and submission_date > '20180101';
```

## 5) 代码块

根关键字应单独占一行，除非后面只跟随一个从属词。如果有多个从属词，它们应形成一个列，并且左对齐，缩进应在根关键字的左侧。

✅ 推荐：

```sql
select
    client_id,
    submission_date
from main_summary
where
    submission_date > '20180101'
    and sample_id = '42'
limit 10;
```

如果只有一个参数，则可以将参数包含在与根关键字的同一行上。

🆗 可接受：

```sql
select
    client_id,
    submission_date
from main_summary
where
    submission_date > '20180101'
    and sample_id = '42'
limit 10;
```

❌ 不推荐：

```sql
select client_id, submission_date
from main_summary
where submission_date > '20180101' and (sample_id = '42' or sample_id is null)
limit 10;
```

❌ 不推荐：

```sql
select
    client_id,
    submission_date
from main_summary
where submission_date > '20180101'
    and sample_id = '42'
limit 10;
```

## 6) 括号

如果括号跨多行：

1. 开始括号应在行尾。
2. 结束括号应对齐放置在开始构造行的第一个字符下。
3. 括号内的内容应缩进一级。

✅ 推荐：

```sql
with sample as (
    select
        client_id,
        submission_date
    from main_summary
    where
        sample_id = '42'
  )
  ...
```

❌ 不推荐 (结束括号与代码同一行)：

```sql
with sample as (
    select
        client_id,
        submission_date
    from main_summary
    where
        sample_id = '42')
  ...
```

❌ 不推荐（无缩进）：

```sql
with sample as (
select
    client_id,
    submission_date
from main_summary
where
    sample_id = '42'
)
  ...
```

## 7) 布尔运算符置于行首

"and" 和 "or" 应始终位于行首。

✅ 推荐：

```sql
...
where
    submission_date > '2018-01-01'
    and sample_id = '42'
```

❌ 不推荐：

```sql
...
where
    submission_date > '2018-01-01' and
    sample_id = '42'
```

------

## 8) 嵌套查询

不要使用嵌套查询，改用通用表表达式 (CTEs，关键字 `WITH`)，以提高可读性。

✅ 推荐：

```sql
with sample as (
    select
        client_id,
        submission_date
    from main_summary
    where
        sample_id = '42'
)
select *
from sample
limit 10;
```

❌ 不推荐：

```sql
select *
from (
    select
        client_id,
        submission_date
    from main_summary
    where
        sample_id = '42'
)
limit 10;
```

## 关于本文档

本文档受到 [SQL Style Guide](https://www.sqlstyle.guide/)https://www.sqlstyle.guide/ 和 [Mozilla SQL Style Guide](https://docs.telemetry.mozilla.org/concepts/sql_style.html) 的深刻影响。欢迎提出建议、扩展和修正。

