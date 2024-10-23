# How to use Docker to run Postgres

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

本指南适用于那些使用或需要使用 Postgres，但对 Docker 不熟悉的用户。

在容器中运行 Docker 进行开发和测试可以帮助你在多个环境之间对齐库、扩展和软件版本集。

## Docker安装 – macOS

使用 [Homebrew](https://brew.sh/) 安装：

```bash
brew install docker docker-compose
```

## Docker安装 – Ubuntu

```bash
sudo apt-get update
sudo apt-get install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg-agent \
  software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

sudo add-apt-repository -y \
 "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update && sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-compose-plugin
```

避免使用 `sudo` 运行 `docker` 命令：

```bash
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
```

## 在带有持久化PGDATA的容器中运行Postgres

假设我们希望数据目录 (`PGDATA`) 位于 `~/pgdata`，并将容器命名为 `pg16`：

```bash
sudo docker run \
  --detach \
  --name pg16 \
  -e POSTGRES_PASSWORD=secret \
  -v ~/pgdata:/var/lib/postgresql/data \
  --shm-size=128m \
  postgres:16
```

## 检查日志

查看过去 5 分钟的日志 (带有时间戳) 并观察新的日志条目：

```bash
docker logs --since 5m -tf pg16
```

## 使用psql连接

创建表：

```bash
❯ docker exec -it pg16 psql -U postgres -c 'create table t()'
CREATE TABLE

❯ docker exec -it pg16 psql -U postgres -c '\d t'
                Table "public.t"
 Column | Type | Collation | Nullable | Default
--------+------+-----------+----------+---------
```

交互式 psql 连接：

```bash
docker exec -it pg16 psql -U postgres
```

## 从外部应用程序连接

要从主机所在机器连接应用程序，我们需要映射端口。为此，我们将停止并删除现有容器，并创建一个新容器，并创建一个具有适当端口映射的新容器 — 注意 `PGDATA` 仍然存在 (我们创建的表就在那里)：

```bash
❯ docker stop pg16
pg16

❯ docker rm pg16
pg16

❯ docker run \
  --detach \
  --name pg16 \
  -e POSTGRES_PASSWORD=secret \
  -v ~/pgdata:/var/lib/postgresql/data \
  --shm-size=128m \
  -p 127.0.0.1:15432:5432 \
  postgres:16
8b5370107e1be7d3fd01a3180999a253c53610ca9ab764125b1512f65e83b927

❯ PGPASSWORD=secret psql -hlocalhost -p15432 -U postgres -c '\d t'
Timing is on.
                Table "public.t"
 Column | Type | Collation | Nullable | Default
--------+------+-----------+----------+---------
```

## 包含额外扩展的自定义镜像

例如，我们可以创建一个基于原始镜像的自定义镜像，包含 `plpython3u` 扩展 (继续使用相同的 `PGDATA`)：

```bash
docker stop pg16
docker rm pg16

echo "FROM postgres:16
RUN apt update
RUN apt install -y postgresql-plpython3-16" \
> postgres_plpython3u.Dockerfile

sudo docker build \
  -t postgres-plpython3u:16 \
  -f postgres_plpython3u.Dockerfile \
  .

docker run \
  --detach \
  --name pg16 \
  -e POSTGRES_PASSWORD=secret \
  -v ~/pgdata:/var/lib/postgresql/data \
  --shm-size=128m \
  postgres-plpython3u:16

docker exec -it pg16 \
  psql -U postgres -c 'create extension plpython3u'
```

## 共享内存

如果你看到如下错误：

```bash
FATAL:  could not resize shared memory segment "/PostgreSQL.12345" to 1048576 bytes: No space left on device1
```

在 `docker run` 命令中增加 `--shm-size` 值。

## 如何升级Postgres并保留数据

1. 原地升级：
   - 传统的 Postgres Docker 镜像仅包含一个主版本的二进制文件，因此无法执行 `pg_upgrade`，除非扩展这些镜像。
   - 另一种选择是使用包含多个二进制文件的镜像，例如 [Spilo by Zalando](https://github.com/zalando/spilo).

2. 简单的转储/恢复 (这里我展示了在没有不兼容的情况下进行降级；升级可以以相同的方式完成)：

```bash
docker exec -it pg16 pg_dumpall -U postgres | bzip2 > dumpall.bz2
docker rm -f pg16

rm -rf ~/pgdata
mkdir ~/pgdata

docker run \
  --detach \
  --name pg15 \
  -e POSTGRES_PASSWORD=secret \
  -v ~/pgdata:/var/lib/postgresql/data \
  --shm-size=128m \
  postgres:15

bzcat dumpall.bz2  \
  | docker exec -i pg15 psql -U postgres \
>>dump_load.log \
2> >(tee -a dump_load.err >&2)
```