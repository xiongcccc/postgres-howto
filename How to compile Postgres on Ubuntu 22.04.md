# How to compile Postgres on Ubuntu 22.04

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

本文介绍了如何快速在 Ubuntu 22.04 上编译 Postgres。

[官方文档](https://postgresql.org/docs/current/installation.html)提供了非常详细的信息，作为参考非常有用，但你不会找到针对特定操作系统版本的具体步骤，比如 Ubuntu。

本文提供的指南足够让你从 `master` 分支构建 Postgres 并开始使用。如果需要，可以根据实际需求扩展这些基本步骤。

几点说明：

- 当前 (截至 2023 年，PG16) 文档中提到了一种新的 Postgres 构建方法 — [Meson](https://mesonbuild.com/) — 但它仍被认为是实验性的，因此本文不会使用。
- 此处使用的二进制文件和 `PGDATA` 路径仅作为示例 (适用于"快速而粗糙"的设置来测试，例如，来自 `pgsql-hackers` 邮件列表的新补丁)。

1. 安装必需的软件
2. 获取源代码
3. 配置
4. 编译和安装
5. 创建集群并开始使用

### 1) 安装必需的软件

```bash
sudo apt update

sudo apt install -y \
  build-essential libreadline-dev \
  zlib1g-dev flex bison libxml2-dev \
  libxslt-dev libssl-dev libxml2-utils \
  xsltproc ccache
```

### 2) 获取源代码

```bash
git clone https://gitlab.com/postgres/postgres.git
cd ./postgres
```

### 3) 配置

```bash
mkdir ~/pg16 # consider changing it!

./configure \
  --prefix=$(echo ~/pg16) \
  --with-ssl=openssl \
  --with-python \
  --enable-depend \
  --enable-cassert
```

(查看可选项列表)

### 4) 编译和安装

```bash
make -j$(grep -c processor /proc/cpuinfo)
make install
```

### 5) 创建集群并开始使用

```bash
~/pg16/bin/initdb \
  -D ~/pgdata \
  --data-checksums \
  --locale-provider=icu \
  --icu-locale=en

~/pg16/bin/pg_ctl \
  -D ~/pgdata \
  -l pg.log start
```

现在，进行检查：

```bash
$ ~/pg16/bin/psql postgres -c 'show server_version'
 server_version
----------------
 17devel
(1 row)
```