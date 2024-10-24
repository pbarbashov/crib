# Analize slow queries

Top slow queries

```SQL
--postgres
SELECT queryid,
datname,
query
as short_query,
sum(calls)
as calls,
sum(total_time)
as total_time,
min(min_time)
as min time,
max(max_time)
as max time,
sum(mean_time * calls) / sum(calls) as mean_time
FROM pg_stat_statements
JOIN pg_database ON pg_stat_statements.dbid = pg_database.oid
where datname = 'db_name'
group by queryid, short_query, datname
order by max_time desc;


--postgres with pgpro extension
SELECT queryid,
datname,
query
as short_query,
sum(calls) sum(total_exec_time) min(min_exec_time) max(max_exec_time)
as calls,
as total_time,
as min_time, as max_time,
sum(mean_exec_time * calls) / sum(calls) as mean_time
FROM pgpro_stats_statements
JOIN pg_database ON pgpro_stats_statements.dbid = pg_database.oid
where datname - 'db_name' group by queryid, short_query, datname order by max_time desc;
```

# Select analyze plan
```SQL
explain (analyze, verbose, buffers) select --...

```

# UPDATE INSERT DELETE analyze
```SQL
begin;
explain (analyze, verbose, buffers) update --...
rollback;
```

# Other useful queries
Show requests that are stuck due to blocking
```SQL
select pid, query, state, wait_event, wait_event_type, pg_blocking_pids(pid) from pg_stat_activity where cardinality(pg_blocking_pids(pid)) > 0;
```

Show statistics on the table (number of rows, vacuum condition, etc.)

```SQL
select * from pg_stat_user_tables where relname = 'table_name' and schemaname = 'schema_name';
```

The full size of the tables, only the size of the table, only the size of the indexes on the table:
```SQL
select pg_size_pretty(pg_total_relation_size('schema.table'));
select pg_size_pretty(pg_table_size('schema.table'));
select pg_size_pretty(pg_indexes_size('schema.table'));
```

Sometimes pg_indexes_size returns 0 size, in such cases, you can run a query to collect the sizes of indexes/tables:

```SQL
SELECT
t.schemaname,
t.tablename,
c.reltuples::bigint
AS num_rows,
pg_size_pretty(pe_relation_size(c.oid))
AS table_size,
psai.indexrelname
AS index_name,
pg_size_pretty(pg_relation_size(i.indexrelid)) CASE WHEN i.indisunique THEN 'Y' ELSE 'N' END
AS index_size, AS "unique", AS number_of_scans, AS tuples read,
psai.idx_scan psai.idx_tup_read psai.idx_tup_fetch
AS tuples_fetched
FROM
pg_tables t
LEFT JOIN pg_class c ON t.tablename = c.relname 
LEFT JOIN pg_index i ON c.oid - i.indrelid 
LEFT JOIN pg_stat_all_indexes psai oN i.indexrelid = psai.indexrelid
WHERE
t.schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY 1, 2;


```

Cluster information and settings
```SQL
select * from pg configuration;
select * from pg settings;
```

Check how actively one or another index is used in the table

```SQL
select * from pg_statio_user_indexes where relname like '%table_name%';
```

Statistics on the distribution of values in the columns of the table. This information should be taken into account when creating indexes

```SQL
select * from pg stats where tablename like "%table_name%";
```


# Проектирование реляционной БД: основные понятия, (анти)рекомендации
## Целостность
Основные виды:
1. Сущностная целостность. $\forall$ кортеж можно отличить от $\forall$ другого кортежа рассматриваемого отношения. Иными словами гарантия уникальности каждой строки в таблице.
Обеспечивается Primary Key. Но, если в качестве PK назначен суррогатный(искусственный ключ), то полезно иметь и естественный ключ для ассоциации технической идентификации с реальным миром.

1. Доменная целостность. Обеспечивает корректность компонента любого кортежа отношения в соответствии с заданными правилами. Примеры: проверка NOT NULL, проверка на тип данных, на форматирование, на принадлежность диапазону.
1. Ссылочная целостность. Ссылка на кортеж в родительской таблице либо указывает на конкретный кортеж либо это ссылка равна NULL. Обеспечивается ограничением Foreign Key.

## Антирекомендации по выбору PK

1. Может быть нарушена уникальность значений ключа (например, № федерального закона, исходящий номер письма).
1. Возможно пустое значение хотя бы одного атрибута, входящего в ключ (например, ISBN, серия и номер документа).
1. Возможно изменение значения атрибутов, входящих в ключ (например, URL, INN, номер телефона).
1. Большое текстовое значение (например, описание товара).
1. Слишком много атрибутов (например, почтовый адрес).
1. Вы не имеете возможность контролировать изменения значений ключа (любой "чужой" ключ).