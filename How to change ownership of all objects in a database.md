# How to change ownership of all objects in a database

>我每天都会发布一篇新的 PostgreSQL "howto" 文章。加入我的旅程吧 — 订阅、提供反馈、分享！

如果需要更改当前数据库中所有对象的所有权，可以使用以下匿名 DO 块 (或从[此处](https://gitlab.com/postgres-ai/database-lab/-/snippets/2075222)粘贴)：

~~~sql
do $$
declare
  new_owner text := 'NEW_OWNER_ROLE_NAME';
  object_type record;
  r record;
  sql text;
begin
  -- New owner for all schemas
  for r in select * from pg_namespace loop
    sql := format(
      'alter schema %I owner to %I;',
      r.nspname,
      new_owner
    );

    raise debug 'Execute SQL: %', sql;

    execute sql;
  end loop;

  -- Various DB objects
  -- c: composite type
  -- p: partitioned table
  -- i: index
  -- r: table
  -- v: view
  -- m: materialized view
  -- S: sequence
  for object_type in
    select
      unnest('{type,table,table,view,materialized view,sequence}'::text[]) type_name,
      unnest('{c,p,r,v,m,S}'::text[]) code
  loop
    for r in
      execute format(
        $sql$
          select n.nspname, c.relname
          from pg_class c
          join pg_namespace n on
            n.oid = c.relnamespace
            and not n.nspname in ('pg_catalog', 'information_schema')
            and c.relkind = %L
          order by c.relname
        $sql$,
        object_type.code
      )
    loop
      sql := format(
        'alter %s %I.%I owner to %I;',
        object_type.type_name,
        r.nspname,
        r.relname,
        new_owner
      );

      raise debug 'Execute SQL: %', sql;

      execute sql;
    end loop;
  end loop;

  -- Functions, procedures
  for r in 
    select
      p.proname,
      n.nspname,
      pg_catalog.pg_get_function_identity_arguments(p.oid) as args
    from pg_catalog.pg_namespace as n
    join pg_catalog.pg_proc as p on p.pronamespace = n.oid
    where
      not n.nspname in ('pg_catalog', 'information_schema')
      and p.proname not ilike 'dblink%' -- We do not want dblink to be involved (exclusion)
  loop
    sql := format(
      'alter function %I.%I(%s) owner to %I', -- todo: check support CamelStyle r.args
      r.nspname,
      r.proname,
      r.args,
      new_owner
    );

    raise debug 'Execute SQL: %', sql;

    execute sql;
  end loop;

  -- Full text search dictionary
  -- TODO: text search configuration
  for r in 
    select * 
    from pg_catalog.pg_namespace n
    join pg_catalog.pg_ts_dict d on d.dictnamespace = n.oid
    where not n.nspname in ('pg_catalog', 'information_schema')
  loop
    sql := format(
      'alter text search dictionary %I.%I owner to %I',
      r.nspname,
      r.dictname,
      new_owner
    );

    raise debug 'Execute SQL: %', sql;

    execute sql;
  end loop;

  -- Domains
  for r in 
     select typname, nspname
     from pg_catalog.pg_type
     join pg_catalog.pg_namespace on pg_namespace.oid = pg_type.typnamespace
     where typtype = 'd' and not nspname in ('pg_catalog', 'information_schema')
  loop
    sql := format(
      'alter domain %I.%I owner to %I',
      r.nspname,
      r.typname,
      new_owner
    );

    raise debug 'Execute SQL: %', sql;

    execute sql;
  end loop;
end
$$;
~~~

不要忘记更改 `new_owner` 的值。

该查询不包括模式 `pg_catalog` 和 `information_schema`，它涵盖：模式对象、表、视图、物化视图、函数、文本搜索字典和域。根据你的 PG 版本，可能还有其他对象需要处理。根据需要调整代码。

要查看调试消息，请在运行之前更改 `client_min_messages`：

~~~sql
set client_min_messages to debug;
~~~

