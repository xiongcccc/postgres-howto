# postgres-howtos
postgres-howtos in chinese，original repo🔗：https://gitlab.com/postgres-ai/postgresql-consulting/postgres-howtos

This project has been started by [@NikolayS]() on 2023-09-26 https://twitter.com/samokhvalov/status/1706748070967624174:

> I'm going to start a PostgreSQL marathon: each day I'll be posting a new "howto" recipe. Today is the day zero, and the first post is here.

> My goal is to create at least 365 posts 😎

> Why am I doing it?
>
> 1. Postgres docs are awesome but often lack practical pieces of advice (howtos)
> 2. 20+ years of database experience, from small startups to giants like Chewy, GitLab, Miro - always have a feeling that I need to share
> 3. eventually I aim to have a structured set of howtos, constantly improving it - and make the systems we develop at [Postgres.ai](https://Postgres.ai) / [Database_Lab](https://twitter.com/Database_Lab) better and more helpful.

[Subscribe](https://twitter.com/samokhvalov/status/1706748070967624174), like, share, and wish me luck with this -- and let's go! 🏊

一个很不错的学习资源，都是总结的一些实践经验，我会不定期翻译一篇，并添加笔者自己的理解。

觉得项目不错，不妨点个 ⭐️ 再走 ~

## 目录

- 0001 [EXPLAIN ANALYZE or EXPLAIN (ANALYZE, BUFFERS)?](https://github.com/xiongcccc/postgres-howto/blob/master/EXPLAIN%20ANALYZE%20or%20EXPLAIN%20(ANALYZE%2C%20BUFFERS).md)
- 0002 [How to troubleshoot and speed up Postgres stop and restart attempts](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20troubleshoot%20and%20speed%20up%20Postgres%20stop%20and%20restart%20attempts.md)
- 0003 [How to troubleshoot long Postgres startup](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20troubleshoot%20long%20Postgres%20startup.md)
- 0004 [Understanding how sparsely tuples are stored in a table](https://github.com/xiongcccc/postgres-howto/blob/master/Understanding%20how%20sparsely%20tuples%20are%20stored%20in%20a%20table.md)
- 0005 [How to work with pg_stat_statments, part 1](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20work%20with%20pg_stat_statements%2C%20part%201.md)
- 0006 [How to work with pg_stat_statments, part 2](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20work%20with%20pg_stat_statements%2C%20part%202.md)
- 0007 [How to work with pg_stat_statments, part 3](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20work%20with%20pg_stat_statements%2C%20part%203.md)
- 0008 [How to speed up pg_dump when dumping large databases](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20speed%20up%20pg_dump%20when%20dumping%20large%20databases.md)
- 0009 [How to understand LSN values and WAL filenames](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20understand%20LSN%20values%20and%20WAL%20filenames.md)
- 0010 [How to troubleshoot Postgres performance using FlameGraphs and eBPF (or perf)](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20troubleshoot%20Postgres%20performance%20using%20FlameGraphs%20and%20eBPF%20(or%20perf).md)
- 0011 [Ad-hoc monitoring](https://github.com/xiongcccc/postgres-howto/blob/master/Ad-hoc%20monitoring.md)
- 0012 [How to find query examples for problematic pg_stat_statements records](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20find%20query%20examples%20for%20problematic%20pg_stat_statements%20records.md)
- 0013 [How to benchmark](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20benchmark.md)
- 0014 [How to decide when a query is too slow and needs optimization](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20decide%20when%20a%20query%20is%20too%20slow%20and%20needs%20optimization.md)
- 0015 [How to monitor CREATE INDEX / REINDEX progress in Postgres 12+](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20monitor%20CREATE%20INDEX%20%3A%20REINDEX%20progress%20in%20Postgres%2012%2B.md)
- 0016 [How to get into trouble using some Postgres features](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20get%20into%20trouble%20using%20some%20Postgres%20features.md)
- 0017 [How to determine the replication lag](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20determine%20the%20replication%20lag.md)
- 0018 [Over-indexing](https://github.com/xiongcccc/postgres-howto/blob/master/Over-indexing.md)
- 0019 [How to import CSV to Postgres](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20import%20CSV%20to%20Postgres.md)
- 0020 [How to use pg_restore](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20use%20pg_restore.md)
- 0021 [How to set application_name without extra queries](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20set%20application_name%20without%20extra%20queries.md)
- 0022 [How to analyze heavyweight locks, part 1](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20analyze%20heavyweight%20locks%2C%20part%201.md)
- 0023 [How to use OpenAI APIs right from Postgres to implement semantic search and GPT chat](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20use%20OpenAI%20APIs%20right%20from%20Postgres%20to%20implement%20semantic%20search%20and%20GPT%20chat.md)
- 0024 [How to work with metadata](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20work%20with%20metadata.md)
- 0025 [How to quit from psql](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20quit%20from%20psql.md)
- 0026 [How to check btree indexes for corruption](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20check%20btree%20indexes%20for%20corruption.md)
- 0027 [How to compile Postgres on Ubuntu 22.04](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20compile%20Postgres%20on%20Ubuntu%2022.04.md)
- 0028 [How to work with arrays, part 1](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20work%20with%20arrays%2C%20part%201.md)
- 0029 [How to work with arrays, part 2](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20work%20with%20arrays%2C%20part%202.md)
- 0030 [How to deal with long-running transactions (OLTP)](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20deal%20with%20long-running%20transactions%20(OLTP).md)
- 0031 [How to troubleshoot a growing pg_wal directory](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20troubleshoot%20a%20growing%20pg_wal%20directory.md)
- 0032 [How to speed up bulk load](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20speed%20up%20bulk%20load.md)
- 0033 [How to redefine a PK without downtime](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20redefine%20a%20PK%20without%20downtime.md)
- 0034 [How to perform initial / rough Postgres tuning](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20perform%20initial%20%20rough%20Postgres%20tuning.md)
- 0035 [How to use subtransactions in Postgres](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20use%20subtransactions%20in%20Postgres.md)
- 0036 ["Find-or-insert" using a single query](https://github.com/xiongcccc/postgres-howto/blob/master/Find-or-insert%20using%20a%20single%20query.md)
- 0037 [How to enable data checksums without downtime](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20enable%20data%20checksums%20without%20downtime.md)
- 0038 [How to NOT get screwed as a DBA (DBRE)](https://github.com/xiongcccc/postgres-howto/blob/master/How%20to%20NOT%20get%20screwed%20as%20a%20DBA%20(DBRE).md)
