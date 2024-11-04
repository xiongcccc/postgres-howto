# How to flush caches (OS page cache and Postgres buffer pool)

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

在实验时，考虑当前缓存的状态非常重要 — 包括 Postgres 缓冲池 (其大小由 `shared_buffers` 控制) 和操作系统的页缓存。如果决定每次实验都从冷状态的缓存开始，我们需要刷新它们。

## 刷新Postgres缓冲池

要刷新 Postgres 缓冲池，请重新启动 Postgres。

要分析当前缓冲池的状态，可以使用 [pg_buffercache](https://postgresql.org/docs/current/pgbuffercache.html)。

## 刷新操作系统页缓存

刷新 Linux 页缓存：

```bash
sync
echo 3 > /proc/sys/vm/drop_caches
```

查看 Linux 中当前 RAM 的使用情况 (MiB)：

```bash
free -m
```

在 macOS 上，刷新页缓存：

```bash
sync
sudo purge
```

查看 macOS 上当前 RAM 的状态：

```bash
vm_stat
```

## 我见

在 17 版本中，`pg_buffercache` 新增 `pg_buffercache_evict`，顾名思义，排除指定缓冲区，这样进行实验就要方便多了。

~~~sql
postgres=# \dx+ pg_buffercache 
 Objects in extension "pg_buffercache"
           Object description           
----------------------------------------
 function pg_buffercache_evict(integer)
 function pg_buffercache_pages()
 function pg_buffercache_summary()
 function pg_buffercache_usage_counts()
 type pg_buffercache
 type pg_buffercache[]
 view pg_buffercache
(7 rows)
~~~

另外，对于 page cache，我们还可以借助 fincore/pcstat 这类工具进行观察。