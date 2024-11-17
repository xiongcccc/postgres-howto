# How to tune Linux parameters for OLTP Postgres

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

以下是一些 Linux 基础调优的通用建议，以便在高并发 OLTP (比如 Web/移动应用) 负载下运行 Postgres。这些参数大部分都是 [postgresql_cluster](https://github.com/vitabaks/postgresql_cluster) 中的默认设置。

请将下列参数视为进一步研究的切入点，文中提供的值仅作为粗略的调优参考，需根据具体情况进行审查。

大部分参数可在 `sysctl.conf` 中修改，更改后需要调用：

```bash
sysctl -p /etc/sysctl.conf
```

临时更改 (以 `vm.swappiness` 为例)：

```bash
sudo sysctl -w vm.swappiness=1
```

或者：

```bash
echo 1 | sudo tee /proc/sys/vm/swappiness
```

## 内存管理

1. `vm.overcommit_memory = 2`

   避免内存过度分配，防止 OOM killer 影响 Postgres。

2. `vm.swappiness = 1`

   最小化 swap 使用，避免完全禁用。

   > 💀这是一个具有争议的话题。我个人在关键任务系统中在高负载下使用 0 来禁用 swap，承担OOM 风险。不过，许多专家建议不要完全关闭，可以使用较低值—1 或 10。

**相关优质文章：**

- [Deep PostgreSQL Thoughts: The Linux Assassin](https://crunchydata.com/blog/deep-postgresql-thoughts-the-linux-assassin) (2021，k8s环境)，作者 [@josepheconway](https://twitter.com/josepheconway)
- [PostgreSQL load tuning on Red Hat Enterprise Linux](https://redhat.com/en/blog/postgresql-load-tuning-red-hat-enterprise-linux) (2022)

3. `vm.min_free_kbytes = 102400`

   确保在内存分配高峰期间为 Postgres 预留可用内存。

4. `transparent_hugepage/enabled=never`，`transparent_hugepage/defrag=never`

   禁用透明大页 (THP)，以防止不适合 Postgres OLTP 工作负载的延迟和碎片。一般建议在 OLTP 系统中禁用 THP (例如 Oracle)。

   - [Ubuntu/Debian](https://stackoverflow.com/questions/44800633/how-to-disable-transparent-huge-pages-thp-in-ubuntu-16-04lts)
   - [Red Hat](https://access.redhat.com/solutions/46111)

## I/O管理

5. `vm.dirty_background_bytes = 67108864`

6. `vm.dirty_bytes = 536870912`

   调整 [pdflush](https://lwn.net/Articles/326552/) 以防止 IO 延迟峰值，详见 [@grayhemp](https://twitter.com/grayhemp) 的 [PgCookbook - a PostgreSQL documentation project](https://github.com/grayhemp/pgcookbook/blob/master/database_server_configuration.md)。

## 网络配置

> 📝 以下是 ipv4 配置；🎯 **TODO：**ipv6 配置待补充

7. `net.ipv4.ip_local_port_range = 10000 65535`

   允许处理更多客户端连接。

8. `net.core.netdev_max_backlog = 10000`

   应对网络流量高峰，避免丢包。

9. `net.ipv4.tcp_max_syn_backlog = 8192`

   适应高并发连接请求。

10. `net.core.somaxconn = 65535`

    增加套接字连接队列的上限。

11. `net.ipv4.tcp_tw_reuse = 1`

    减少高吞吐量 OLTP 应用的连接建立时间。

## NUMA配置

12. `vm.zone_reclaim_mode = 0`

    避免跨 NUMA 节点的内存回收对 Postgres 造成性能影响。

13. `kernel.numa_balancing = 0`

    禁用自动 NUMA 平衡，以提高 Postgres 的 CPU 缓存效率。

14. `kernel.sched_autogroup_enabled = 0`

    改善 Postgres 进程调度的延迟。

## 文件系统和文件句柄

15. `fs.file-max = 262144`

    设置 Linux 内核可以分配的最大文件句柄数量。在运行像 Postgres 这样的数据库服务器时，足够的文件描述符对于处理大量连接和文件至关重要。

> 🎯 TODO：根据不同主流操作系统进行进一步审查和调整。

## 我见

文中提到了几个不错的工具：

- [PgCookbook - a PostgreSQL documentation project](https://github.com/grayhemp/pgcookbook)

  > The project is a continuously updating set of articles, scripts and configuration files made to help with PostgreSQL maintenance. The articles and files might be modified as new versions of software or new ways of doing things appear. Stay tuned.

- [PgToolkit - tools for PostgreSQL maintenance](https://github.com/grayhemp/pgtoolkit)

  >Currently the package contains the only tool `pgcompact`, we are planning to add much more in the future. Stay tuned.
  >
  >The list of changes can be found in [CHANGES.md](https://github.com/grayhemp/pgtoolkit/blob/master/CHANGES.md). The To-Do List is in [TODO.md](https://github.com/grayhemp/pgtoolkit/blob/master/TODO.md).