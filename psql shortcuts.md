# psql shortcuts

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

在我的工具 [postgres_dba](https://github.com/NikolayS/postgres_dba/) 中，安装命令包含：

```bash
printf "%s %s %s %s\n" \\set dba \'\\\\i $(pwd)/start.psql\' >> ~/.psqlrc
```

这种方式可以让我们在 `psql` 中只需输入 `:dba` 即可调用 `start.psql`，而且该命令在 `bash` 和 `zsh` 中均有效 (macOS 几年前将默认 shell 更改为 `zsh`，而 `bash` 通常是 Linux 发行版的默认 shell)。

我们可以很容易地在 `.psqlrc` 中添加自己的指令，以定义各种便捷的快捷键。比如：

```bash
\set pid 'select pg_backend_pid() as pid;'
\set prim 'select not pg_is_in_recovery() as is_primary;'
\set a 'select state, count(*) from pg_stat_activity where pid <> pg_backend_pid() group by 1 order by 1;'
```

👉 这添加了简便的快捷方式：

1. 当前 PID：

```sql
nik=# :pid
  pid
--------
 513553
(1 row)
```

2. 判断是否为主节点？

```sql
nik=# :prim
 is_primary
------------
 t
(1 row)
```

3. 简单活动摘要

```sql
nik=# :a
        state        | count
---------------------+-------
 active              |    19
 idle                |   193
 idle in transaction |     2
                     |     7
(4 rows)
```

在 SQL 上下文中，可以使用下面这种有趣的语法，将当前设置的 psql 变量值作为字符串进行传递：

```sql
nik=# select :'pid';
            ?column?
---------------------------------
 select pg_backend_pid() as pid;
(1 row)
```

在编写脚本时，不要忘了使用 `-X` (`--no-psqlrc`)选项，以确保 `.psqlrc` 中的内容不会影响脚本的逻辑。

