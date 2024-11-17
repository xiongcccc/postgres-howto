# How to make "\e" work in psql on a new machine ("editor/nano/vi not found")

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

有时在 psql 中使用 `\e` 命令编辑查询时会遇到以下错误：

```sql
nik=# \e
/usr/bin/sensible-editor: 20: editor: not found
/usr/bin/sensible-editor: 31: nano: not found
/usr/bin/sensible-editor: 20: nano-tiny: not found
/usr/bin/sensible-editor: 20: vi: not found
Couldn't find an editor!
Set the $EDITOR environment variable to your desired editor.
```

设置编辑器非常简单 (使用 `nano` 或喜欢的其他编辑器)：

```sql
\setenv PSQL_EDITOR vim
```

但是，如果你在容器内或一台新机器上工作，可能还未安装所需的编辑器。假设有权限运行安装命令，可以直接在 `psql` 中安装编辑器。例如，在基于 Debian 的标准 Postgres 环境中 (不需要 `sudo`)：

```sql
nik=# \! apt update && apt install -y vim
```

👉 `\e` 命令就能正常工作了！

要持久化该设置，可以将其添加到 `~/.bash_profile` (或 `~/.zprofile`) 中：

```bash
echo "export PSQL_EDITOR=vim" >> ~/.bash_profile
source ~/.bash_profile
```

对于 Window，请参见 [@PavloGolub](https://twitter.com/PavloGolub) 的[博客文章](https://cybertec-postgresql.com/en/psql_editor-fighting-with-sublime-text-under-windows/)。

文档： https://postgresql.org/docs/current/app-psql.html