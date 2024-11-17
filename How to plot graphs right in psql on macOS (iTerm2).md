# How to plot graphs right in psql on macOS (iTerm2)

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

如果你像我一样主要在 psql 中使用 Postgres，那么你可能希望能够在不离开 psql 的情况下绘制一个简单的图表。

以下方法最初由 [Alexander Korotkov](https://akorotkov.github.io/blog/2016/06/09/psql-graph/) 描述，我对其进行了[略微调整](https://gist.github.com/NikolayS/d5f1af808f7275dc1491c37fb1e2dd11)以适配Python3。此方法得益于 iTerm2的[内嵌图片协议](https://iterm2.com/documentation-images.html)。对于 Linux，有其他方式可以实现类似效果 (例如，[How do I make my terminal display graphical pictures?](https://askubuntu.com/questions/97542/how-do-i-make-my-terminal-display-graphical-pictures))。

1. 获取绘图脚本并安装 matplotlib：

```bash
wget \
  -O ~/pg_graph.py \
  https://gist.githubusercontent.com/NikolayS/d5f1af808f7275dc1491c37fb1e2dd11/raw/4f19a23222a6f7cf66eead3cae9617dd39bf07a5/pg_graph

pip install matplotlib
```

2. 在 ~/.psqlrc 中定义一个宏 (在 bash、zsh 和 csh 中均可工作)：

```bash
printf "%s %s %s %s %s %s\n" \\set graph \'\\\\g \|
python3 $(pwd)/pg_graph.py\' \
  >> ~/.psqlrc
```

3. 启动 psql 并尝试一下

```sql
nik=# with avg_temp(month, san_diego, vancouver, london) as (
    values
      ('Jan', 15, 4, 5),
      ('Feb', 16, 5, 6),
      ('Mar', 17, 7, 8),
      ('Apr', 18, 10, 11),
      ('May', 19, 14, 15),
      ('Jun', 21, 17, 17),
      ('Jul', 24, 20, 19),
      ('Aug', 25, 21, 20),
      ('Sep', 23, 18, 17),
      ('Oct', 21, 12, 13),
      ('Nov', 18, 8, 8),
      ('Dec', 16, 5, 6)
  )
  select * from avg_temp;

 month | san_diego | vancouver | london
-------+-----------+-----------+--------
 Jan   |        15 |         4 |      5
 Feb   |        16 |         5 |      6
 Mar   |        17 |         7 |      8
 Apr   |        18 |        10 |     11
 May   |        19 |        14 |     15
 Jun   |        21 |        17 |     17
 Jul   |        24 |        20 |     19
 Aug   |        25 |        21 |     20
 Sep   |        23 |        18 |     17
 Oct   |        21 |        12 |     13
 Nov   |        18 |         8 |      8
 Dec   |        16 |         5 |      6
(12 rows)

nik=# :graph
```

结果：

![Graph result](https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos/-/raw/main/files/0081_01.png)