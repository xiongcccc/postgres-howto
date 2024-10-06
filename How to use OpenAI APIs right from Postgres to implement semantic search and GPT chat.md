# How to use OpenAI APIs right from Postgres to implement semantic search and GPT chat

> 我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

今天，我们将在 Postgres 中实现 RAG ([Retrieval Augmented Generation](https://en.wikipedia.org/wiki/Prompt_engineering#Retrieval-augmented_generation])) ：

1. 将完整的 Postgres 提交历史加载到 Postgres 表中。
2. 使用 `plpython3u` (某些托管服务上不可用，如 RDS)，从 Postgres 中直接调用 OpenAI API。

> ⚠️ 注意：这种方法不能很好地扩展，因此不推荐用于大型生产集群。请将其视为一种娱乐或用于小型项目/服务的方式。

3. 对于每个提交，生成 OpenAI 嵌入并将其存储为"vector"格式 (`pgvector`)。 
4. 使用语义搜索查找提交，通过 `pgvector` 的 HNSW 索引加速搜索。 
5. 最后，通过 OpenAI GPT4 实现"与提交历史对话"。

(Inspired by: [@jonatasdp's Tweet](https://twitter.com/jonatasdp/status/1714711585191596419))

## 准备工作

首先，安装扩展：

```sql
create extension if not exists plpython3u;
create extension if not exists vector;
```

然后，我们需要一个 [OpenAI API key](https://help.openai.com/en/articles/4936850-where-do-i-find-my-secret-api-key)。我们将其存储为当前数据库定义的自定义 GUC。

> ⚠️ 注意：这种存储方式不够安全；我们仅为简化操作使用此方法：

```sql
do $$ begin
 execute(
   format(
     'alter database %I set opanai.api_key = %L',
     current_database(),
     'sk-XXXXXXXXXXXX'
   )
 );
end $$;
```

------

## 从Git导入提交历史

创建一个表来存储 Git 提交记录：

```bash
psql -X \
  -c "create table commit_logs (
    id bigserial primary key,
    hash text not null,
    created_at timestamptz not null,
    authors text,
    message text,
    embedding vector(1536)
  )" \
  -c "create unique index on commit_logs(hash)"
```

现在，克隆 Postgres 代码库并将完整的提交历史导出到 CSV 文件 (注意处理提交消息中的双引号转义)：

```bash
git clone https://gitlab.com/postgres/postgres.git
cd postgres

git log --all --pretty=format:'%H,%ad,%an,🐘%s %b🐘🐍' --date=iso \
 | tr '\n' ' ' \
 | sed 's/"/""/g' \
 | sed 's/🐘/"/g' \
 | sed 's/🐍/\n/g' \
 > commit_logs.csv
```

将提交历史从 CSV 文件加载到表中：

```bash
psql -Xc "copy commit_logs(hash, created_at, authors, message)
   from stdin
   with delimiter ',' csv" \
 < commit_logs.csv

psql -X \
 -c "update commit_logs set hash = trim(hash)" \
 -c "vacuum full analyze commit_logs"
```

截至 2023 年 10 月，这将生成约 88,000 行，涵盖 Postgres 开发历史的近 10,000 天 (超过 27 年 — 第一次提交是在 1996-07-09)。

## 创建并存储嵌入

以下是用于从 OpenAI API 获取每个提交条目的向量的函数，使用 `plpython3u`（`u` 表示"不受信任"，允许该函数与外部世界通信)：

```sql
create or replace function openai_get_embedding(
 content text,
 api_key text default current_setting('opanai.api_key', true)
) returns vector(1536)
as $$
 import requests

 response = requests.post(
   'https://api.openai.com/v1/embeddings',
   headers={ 'Authorization': f'Bearer {api_key}' },
   json={
     'model': 'text-embedding-ada-002',
     'input': content.replace("\n", " ")
   }
 )

 if response.status_code >= 400:
   raise Exception(f"Failed. Code: {response.status_code}")

 return response.json()['data'][0]['embedding']
$$ language plpython3u;
```

函数创建之后，开始获取和存储向量。我们将分批处理，避免长时间运行的事务，万一发生故障，不会丢失大量数据，也不会阻塞并发会话 (如果有的话)：

```sql
with scope as (
 select hash
 from commit_logs
 where embedding is null
 order by id
 limit 5
), upd as (
 update commit_logs
 set embedding = openai_get_embedding(
   content := format(
     'Date: %s. Hash: %s. Message: %s',
     created_at,
     hash,
     message
   )
 )
 where hash in (
   select hash
   from scope
 )
 returning *
)
select
 count(embedding) as cnt_vectorized,
 max(http://upd.id) as latest_upd_id,
 round(
   max(http://upd.id)::numeric * 100 /
     (select max(http://c.id) from commit_logs as c),
   2
 )::text || '%' as progress,
 1 / count(*) as err_when_done
from upd
\watch .1
```

这个过程可能需要一些时间，可能会超过一个小时，所以需要耐心等待。此外，我们在此过程中开始使用 OpenAI 的 API (虽然嵌入创建的成本非常低，估计大约为 1 美元，详见[定价](https://openai.com/api/pricing/))。

一旦完成 (或者在部分结果生成后)，就可以开始使用它。

## 语义搜索

以下是使用语义搜索查找最相关提交的简单示例：

这个概念很简单：

1. 首先调用 OpenAI API，将查询的文本向量化。
2. 然后使用 `pgvector` 的相似性搜索查找 K 个最近邻。

我们将使用 `HNSW` 索引，这被认为是目前最好的方法之一 (尽管最早是在 [2016 ](https://arxiv.org/abs/1603.09320)年描述的)；已被许多数据库管理系统添加。`pgvector` 在 0.5.0 版本中添加了该索引。需要注意的是，这是一个 `ANN` 索引 — "近似最近邻"，为了速度，它允许产生不严格的结果，不像 Postgres 中的常规索引。

创建索引：

```bash
psql -Xc "create index on commit_logs
 using hnsw (embedding vector_cosine_ops)"
```

现在，在 `psql` 中执行搜索：

```
select
 created_at,
 format(
   'https://gitlab.com/postgres/postgres/-/commit/%s',
   left(hash, 8)
 ),
 left(message, 150),
 authors,
 1 - (embedding <-> :'q_vector') as similarity
from commit_logs
order by embedding <-> :'q_vector'
limit 5 \gx
```

如果索引已创建完成，第二个查询应该非常快。你可以使用 `EXPLAIN (ANALYZE, BUFFERS)` 检查计划和执行详情。我们的数据集很小 (< 100 k)，所以搜索速度大约为 1 毫秒，缓冲区命中/读取数量小于 1000。pgvector 提供了一些针对索引的调整选项，请查看其 [README](https://github.com/pgvector/pgvector/blob/master/README.md)。

以下是使用 `\watch` 限制循环查询结果的示例：

```sql
-[ RECORD 1 ]------------------------------------------------------------------------------------------------------------------------------------------------------
created_at | 2023-04-06 17:18:14+00
format     | https://gitlab.com/postgres/postgres/-/commit/00beecfe
left       | psql: add an optional execution-count limit to \watch. \watch can now be told to stop after N executions of the query.  With the idea that we might wa
authors    | Tom Lane
similarity | 0.4774958262998097

-[ RECORD 2 ]------------------------------------------------------------------------------------------------------------------------------------------------------
created_at | 2023-09-18 14:19:25+00
format     | https://gitlab.com/postgres/postgres/-/commit/d726897c
left       | Fix psql's \? output for \watch It was reported as misaligned by Kyotaro, but it also needed to be turned into a single translatable phrase (like the
authors    | Alvaro Herrera
similarity | 0.4347320155373213

-[ RECORD 3 ]------------------------------------------------------------------------------------------------------------------------------------------------------
created_at | 2021-07-12 23:13:48+00
format     | https://gitlab.com/postgres/postgres/-/commit/7c09d279
left       | Add PSQL_WATCH_PAGER for psql's \watch command. Allow a pager to be used by the \watch command.  This works but isn't very useful with traditional pag
authors    | Thomas Munro
similarity | 0.4234620303691825
```

------

## 在Postgres中通过GPT4与"Git历史对话"

最后，创建两个函数，并提出有关 Postgres 提交历史的问题：

```sql
create or replace function openai_gpt_call(
 question text,
 data_to_embed text,
 model text default 'gpt-4',
 token_limit int default 4096,
 api_key text default current_setting('opanai.api_key', true)
) returns text
as $$
 import requests, json

 prompt = """Be terse. Discuss only Postgres and it's commits.
For commits, mention timestamp and hash.
CONTEXT (git commits):
---
%s
---
QUESTION: %s
""" % (data_to_embed[:2000], question)

 ### W: this code lacks error handling
 response = http://requests.post(
   'https://api.openai.com/v1/chat/completions',
   headers={ 'Authorization': f'Bearer {api_key}' },
   json={
     'model': model,
     'messages': [
         {"role": "user", "content": prompt}
     ],
     'max_tokens': token_limit,
     'temperature': 0
   }
 )

 if response.status_code >= 400:
   raise Exception(f"Failed. Code: {response.status_code}. Response: {response.text}")

 return response.json()['choices'][0]['message']['content']
$$ language plpython3u;

create or replace function openai_chat(
 in question text,
 in model text default 'gpt-4',
 out answer text
) as $$
 with q as materialized (
   select openai_get_embedding(
     question
   ) as emb
 ), find_enries as (
   select
     format(
       e'Created: %s, hash: %s, message: %s, committer: %s\n',
       created_at,
       left(hash, 8),
       message,
       authors
     ) as entry
   from commit_logs
   where embedding <=> (select emb from q) < 0.8
   order by embedding <=> (select emb from q)
   limit 10 -- adjust if needed
 )
 select openai_gpt_call(
   question := openai_chat.question,
   data_to_embed := string_agg(entry, e'\n'),
   model := openai_chat.model
 )
 from find_enries;
$$ language sql;
```

现在，可以通过 `openai_chat(...)` 提问，例如：

```sql
nik=# select openai_chat('tell me about fixes of CREATE INDEX CONCURRENTLY – when, who, etc.');
openai_chat
There are two notable fixes for CREATE INDEX CONCURRENTLY. The first one was made on 2016-02-16 18:43:03+00 by Tom Lane, with the commit hash a65313f2. The commit improved the documentation about CREATE INDEX CONCURRENTLY, clarifying the description of which transactions will block a CREATE INDEX CONCURRENTLY command from proceeding.

The second fix was made on 2012-11-29 02:25:27+00, with the commit hash 3c840464. This commit fixed assorted bugs in CREATE/DROP INDEX CONCURRENTLY. The issue was that the pg_index state for an index that's reached the final pre-drop stage was the same as the state for an index just created by CREATE INDEX CONCURRENTLY. This was fixed by adding an additional boolean column "indislive" to pg_index.
```

请注意，它默认使用模型 "gpt-4"，它比 "gpt-3.5-turbo-16k" 更慢且更昂贵 (请参考[定价](https://openai.com/pricing))。

## 一些注意事项

1. 正如前面提到的，从 Postgres 中调用外部 API 的这种方法扩展性不佳。它适合快速原型开发，但不应在 TPS 预计会显著增长的项目中使用 (否则，随着规模增长，CPU 饱和、空闲事务高峰等问题可能导致严重的性能问题甚至宕机)。
2. 另一个缺点是 `plpython3u` 在某些 Postgres 服务上不可用 (例如 RDS)。
3. 最后，在 SQL 上下文中工作时，很容易无意中在循环中调用 API。这可能导致过度开销。为避免此问题，我们需要仔细检查执行计划。
4. 对于某些人来说，这类代码也可能更难调试。

尽管如此，上述概念在某些情况下可能非常有效 — 只需记住这些细节，避免低效操作。如果问题过大，可以将 API 调用代码从 `plpython3u` 移到 Postgres 之外。