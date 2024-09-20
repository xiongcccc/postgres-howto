# How to troubleshoot Postgres performance using FlameGraphs and eBPF (or perf)

今天我们将讨论如何在 Linux 机器上了解 Postgres 后台进程在 CPU 内的具体操作 (in-CPU analysis)。

RDS 以及其他托管 Postgres 服务的用户们 — 很遗憾，这个方法不能在你们的环境中使用。建议与服务提供商沟通：你们支付的不仅仅是虚拟机的费用，但他们只提供了 SQL 级的访问 — 这不太公平。

我将尽可能简化这个教程，并演示一个有趣的示例，在某些情况下可能会有帮助 — 我们将尝试了解在查询规划阶段幕后到底发生了什么。

要进行 in-CPU analysis，我们将使用 Brendan Gregg 的 [flamegraph](https://brendangregg.com/FlameGraphs/cpuflamegraphs.html) (火焰图)。

Brendan Gregg 将火焰图描述为"分层数据的可视化，旨在可视化分析软件的堆栈跟踪，以便快速准确地识别最常见的代码路径” (discussed [here](https://brendangregg.com/flamegraphs.html))。

要生成火焰图，我们需要执行几个非常简单的步骤：

1. 安装用于收集分析数据的工具。这里有两个选项：用于 eBPF 的工具或 `perf` 工具。
2. 安装带有 Postgres 调试符号的包，如果需要，还要安装所使用的扩展的调试符号包。
3. 克隆 FlameGraph 的仓库。
4. 获取我们要分析的 Postgres 后台进程的进程 ID。
5. 收集数据。
6. 生成火焰图。

## 示例设置 (表结构和负载)

在讨论分析步骤之前，先介绍一下我们要探索的内容。以下步骤在连接到本地 Postgres 的 psql 中执行。

表结构：

```sql
create table t1 as select i from generate_series(1, 1000000) i;
create table t2 as select i from generate_series(1, 10000000) i;
alter table t1 add primary key (i);
alter table t2 add primary key (i);
vacuum analyze;
```

然后在无限循环中执行一些 Postgres 规划器的活动，并保持其运行：

```sql
select pg_backend_pid(); -- remember it for step 4
(but check again if Postgres is restarted)

explain select from t1
join t2 using (i)
where i between 1000 and 2000 \watch .01
```

现在，开始分析。

## 第一步：安装用于收集数据的工具

我们将安装 `bcc-tools` (eBPF) 和 `linux-tools-***` (`perf`) 来展示其工作原理 — 你可以选择其中之一。

以 eBPF 和 Ubuntu 22.04 为例：

```shell
sudo apt update
apt install -y \
  linux-headers-$(uname -r) \
  bpfcc-tools # 可能叫bcc-tools
```

安装 perf 工具：

```shell
sudo apt update
sudo apt install -y linux-tools-$(uname -r)
```

如果不工作，尝试检查你有哪些可用的工具：

```shell
uname -r
sudo apt search linux-tools | grep $(uname -r)
```

此页面提供了一些关于安装 `perf` 的不错建议，包括如何与容器一起使用：

- [perf, what’s that?](https://www.swift.org/documentation/server/guides/linux-perf.html)

使用 perf 的一个好方法是直接运行 `top` 命令：

```shell
perf top
```

然后观察系统整体的运行情况。注意，许多函数将以难以读取的形式出现，例如：`0x00000000001d6d88` — 某些名称无法解析的函数的内存地址。

![img](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0010_perf_top_wo_debug_symbols.png)

这是因为我们还没有 Postgres 的调试符号。让我们修复这个问题。

## 第二步：安装Postgres调试符号

我推荐在所有生产系统中都安装调试符号。

对于Postgres 16：

```shell
apt install -y postgresql-16-dbgsym
```

你可能需要更多与 Postgres 相关的软件包，例如：

```shell
apt search postgres | grep dbgsym
```

注意，并不是所有与 Postgres 相关的软件包都带有 "`postgres`" 一词 — 例如，对于 `pgBouncer`，你需要安装 `pgbouncer-dbgsym`。

安装好调试符号包之后，别忘了重启 Postgres (以及我们在 psql 中使用 `EXPLAIN.. \watch`  的无限循环)。

现在，`perf top` 将看起来更好 — 所有 Postgres 相关的行都有了函数名称：

![img](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0010_perf_top_w_debug_symbols.png)

### 第三步：获取FlameGraph工具

```shell
git clone https://github.com/brendangregg/FlameGraph
cd FlameGraph
```

## 第四步：获取Postgres的进程ID (PID)

如果你需要知道 Postgres 后台进程的 PID：

```sql
select pg_backend_pid();
```

如果你已有某些进程在运行，可以使用`pg_stat_activity`：

```sql
select pid, query
from pg_stat_activity
where
  application_name = 'psql'
  and pid <> pg_backend_pid();
```

一旦我们知道了 PID，便可以将它保存为一个变量：

```shell
export PGPID=<你的值>
```

## 第五步：收集分析数据

eBPF 版本，收集 PID 为 $`PGPID` 的进程在 10 秒内的数据：

```shell
profile-bpfcc -p $PGPID -F 99 -adf 10 > out
```

如果你更喜欢 `perf`：

```shell
perf record -F 99 -a -g -- sleep 10
perf script | ./stackcollapse-perf.pl > out
```

## 第六步：生成火焰图

```shell
./flamegraph.pl --colors=blue out > profile.svg
```

## 结果

完成了。现在你需要将 `profile.svg` 文件复制到你的机器上，打开它，例如在浏览器中 — 它将显示这个 SVG 文件，而且支持点击各个区域 (对于探索各个区域非常有用)。

以下是我们的进程在无限 EXPLAIN 循环中的结果：

![img](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0010_flamegraph.png)

有趣的是，约 35% 的 CPU 时间用于分析是否值得使用 `Merge Join`，而最后规划器选择了 `Nested Loop`：

```sql
postgres=# explain (costs off) select
from t1 join t2 using (i)
where i between 1000 and 2000;
                    QUERY PLAN
---------------------------------------------------
 Nested Loop
   ->  Index Only Scan using t1_pkey on t1
         Index Cond: ((i >= 1000) AND (i <= 2000))
   ->  Index Only Scan using t2_pkey on t2
         Index Cond: (i = t1.i)
(5 rows)
```

在这个案例中，计划时间非常短，低于毫秒级 — 但我遇到过一些情况，计划阶段非常慢，持续数秒甚至几十秒。最终 (多亏了火焰图！)发现，分析 Merge Join 路径是造成问题的原因，因此禁用 Merge Join (`set enable_mergejoin = off`) 后，计划时间大幅下降，恢复到了合理的水平。但这是另一个故事。

## 推荐阅读

- Brendan Gregg 的书："Systems Performance" and "BPF Performance Tools"
- Brendan Gregg 的演讲 — 例如 ["eBPF: Fueling New Flame Graphs & more • Brendan Gregg"](https://youtube.com/watch?v=HKQR7wVapgk) 
- [Profiling with perf](https://wiki.postgresql.org/wiki/Profiling_with_perf) (Postgres wiki)

---

这篇文章对你有帮助吗？告诉我吧 — 也请与使用#PostgreSQL的同事和朋友分享！

# 我见

一图胜千言

<img width="1225" alt="image" src="https://github.com/user-attachments/assets/53a1c326-458e-4236-876a-2246ad972e7e">
