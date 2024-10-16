# How to install Postgres 16 with plpython3u: Recipes for macOS, Ubuntu, Debian, CentOS, Docker

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

PL/Python 是 PostgreSQL 的一种过程语言扩展，允许你使用 Python (一种广泛使用、高级且功能多样的编程语言) 编写存储过程和触发器。

`plpython3u` 是 PL/Python 的"不受信任"版本。这种变体允许 Python 函数执行诸如文件 I/O、网络通信等操作，可能会影响服务器的行为或安全。

我们在 [Day 23: How to use OpenAI APIs right from Postgres to implement semantic search and GPT chat](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/blob/main/0023_how_to_use_openai_apis_in_postgres.md) 使用了 `plpython3u`  ，接下来我们将讨论如何安装它。

我有预感，未来我们会在各种任务中使用它。

本文适用于自行管理的 Postgres 环境。

## macOS (Homebrew)

```shell
brew tap petere/postgresql
brew install petere/postgresql/postgresql@16

psql postgres \
  -c 'create extension plpython3u'
```

## Ubuntu 22.04 LTS or Debian 12

~~~bash
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" \
  > /etc/apt/sources.list.d/pgdg.list'

curl -fsSL https://postgresql.org/media/keys/ACCC4CF8.asc \
  | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/postgresql.gpg

sudo apt update
sudo apt install -y \
  postgresql-16 \
  postgresql-contrib-16 \
  postgresql-plpython3-16

sudo -u postgres psql \
  -c 'create extension plpython3u'
~~~

## CentOS Stream 9

~~~bash
dnf module reset -y postgresql
dnf module enable -y postgresql:16

dnf install -y \
  postgresql-server \
  postgresql \
  postgresql-contrib \
  postgresql-plpython3

postgresql-setup --initdb

systemctl enable --now postgresql

sudo -u postgres psql \
  -c 'create extension plpython3u'
~~~

## Docker

~~~bash
echo "FROM postgres:16
RUN apt update
RUN apt install -y postgresql-plpython3-16" \
> postgres_plpython3u.Dockerfile

sudo docker build \
  -t postgres-plpython3u:16 \
  -f postgres_plpython3u.Dockerfile \
  .

sudo docker run \
  --detach \
  --name pg16 \
  -e POSTGRES_PASSWORD=secret \
  -v $(echo ~)/pgdata:/var/lib/postgresql/data \
  postgres-plpython3u:16

sudo docker exec -it pg16 \
  psql -U postgres -c 'create extension plpython3u'
~~~

