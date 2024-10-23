# psql tuning

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

## .psqlrc

文件 `~/.psqlrc` 可以用于设置一些默认配置。例如：

```bash
echo '\timing on' >> ~/.psqlrc
```

现在，如果我们启动 `psql`：

```bash
❯ psql -U postgres
Timing is on.
psql (15.5 (Debian 15.5-1.pgdg120+1))
Type "help" for help.

nik=# select pg_sleep(.5);
 pg_sleep
----------

(1 row)

Time: 508.187 ms
```

❗**重要提示：**如果脚本涉及到 `psql`，建议使用 `-X` 选项来忽略 `~/.psqlrc` 中的设置，这样脚本逻辑 (例如，分析 `psql` 的输出) 不会依赖于 `~/.psqlrc` 中的任何配置。

## pspg

[pspg](https://github.com/okbob/pspg) 非常棒。如果可能的话，建议安装。

例如，对于查询 `select * from pg_class limit 3;`，安装前：

![psql ugly output](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0059_psql_ugly_output.png)

安装后：

![pspg improved output](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0059_pspg_improved_output.png)

![pspg menus](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0059_pspg_menus.jpg)

安装方法：

- macOS/Homebrew：`brew install pspg`
- Ubuntu/Debian：`sudo apt update && sudo apt install -y pspg`

然后：

```bash
echo "\setenv PAGER pspg
\pset border 2
\pset linestyle unicode
\set x '\setenv PAGER less'
\set xx '\setenv PAGER \'pspg -bX --no-mouse\''
" >> ~/.psqlrc
```

## NULLs

默认情况下，`psql` 输出中的 `NULL` 值是不可见的：

```sql
nik=# select null;
 ?column?
----------

(1 row)
```

使用如下方式进行修复 (将其放入 `~/.psqrc` 以确保持久性)

```bash
\pset null 'Ø'
```

现在 `NULL` 值将显示为：

```sql
nik=# select null;
 ?column?
----------
 Ø
(1 row)
```

## postgres_dba

[postgres_dba](https://github.com/NikolayS/postgres_dba) 是我个人收集的一些用于 `psql` 的脚本集合，带有菜单操作。

![postgres_dba menu support](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0059_postgres_dba.jpg)