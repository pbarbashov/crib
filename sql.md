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

## Нормализация БД
Цель нормализации создание наиболее подходящей структуры сущностей и их связей для решения конкретной задачи. Свойства полученной структуры: уменьшение избыточности и вероятности неконсистентности в данных(избыточными остаются как правило только внешние ключи), наиболее связанные друг с другом атрибуты хранятся вместе. Недостатки ненормализованной базы на примере
Staff(staffNo, sName, position, salary, branchNo)  
Branch(branchNo, bAddress)  
StaffBranch(staffNo, sName, position , salary, branchNo, bAddress)  

1. Проблема вставки. Для того, чтобы вставить нового работника в существующее подразделение в StaffBranch необходимо отдублировать адрес подразделения. Это может привести к некосистентности адресов.
1. Проблема удаления. При удалении из StaffBranch последнего сотрудника в подразделении удаляется и информация о подразделении.
1. Проблема обновления. Для изменения адреса в общем случае необходимо поменять несколько записей StaffBranch.
Нормализация основана на понятии функцональной зависимости между атрибутами сущностей.  
Атрибут(ы) B функционально зависит от A (A→B), если для любого потенциального значения атрибута(ов) A существует единственное значение B.  
Атрибут B полностью функционально зависит от A, если A→B и не существует собственного подмножества атрибутов S ⊂ A, такого что S→B.  
Если ∃ A→B и B→C и B↛A и C↛A, то A→C - транзитивная зависимость через B.  
Детерминант функциональной зависимости A→B - это набор атрибутов A.  

### UNF
Это таблица с повтрояющимися значениями атрибутов.
Например, 
| clientNo | cName         | propertyNo | pAddress                      | rentStart  | rentFinish  | rent | ownerNo | oName        |
|----------|---------------|------------|-------------------------------|------------|-------------|------|---------|--------------|
| CR76     | John Kay      | PG4        | 6 Lawrence St, Glasgow        | 1-Jul-07   | 31-Aug-08   | 350  | CO40    | Tina Murphy  |
|          |               | PG16       | 5 Novar Dr, Glasgow           | 1-Sep-08   | 1-Sep-09    | 450  | CO93    | Tony Shaw    |
| CR56     | Aline Stewart | PG4        | 6 Lawrence St, Glasgow        | 1-Sep-06   | 10-Jun-07   | 350  | CO40    | Tina Murphy  |
|          |               | PG36       | 2 Manor Rd, Glasgow           | 10-Oct-07  | 1-Dec-08    | 375  | CO93    | Tony Shaw    |
|          |               | PG16       | 5 Novar Dr, Glasgow           | 1-Nov-09   | 10-Aug-10   | 450  | CO93    | Tony Shaw    |


### 1NF

Для того, чтобы таблица находилась в 1NF необходимо и достаточно, чтобы на пересечении любых столбца и строки находилось одно значение.
Пример:  

| clientNo | propertyNo | cName         | pAddress                      | rentStart  | rentFinish  | rent | ownerNo | oName        |
|----------|------------|---------------|-------------------------------|------------|-------------|------|---------|--------------|
| CR76     | PG4        | John Kay      | 6 Lawrence St, Glasgow        | 1-Jul-07   | 31-Aug-08   | 350  | CO40    | Tina Murphy  |
| CR76     | PG16       | John Kay      | 5 Novar Dr, Glasgow           | 1-Sep-08   | 1-Sep-09    | 450  | CO93    | Tony Shaw    |
| CR56     | PG4        | Aline Stewart | 6 Lawrence St, Glasgow        | 1-Sep-06   | 10-Jun-07   | 350  | CO40    | Tina Murphy  |
| CR56     | PG36       | Aline Stewart | 2 Manor Rd, Glasgow           | 10-Oct-07  | 1-Dec-08    | 375  | CO93    | Tony Shaw    |
| CR56     | PG16       | Aline Stewart | 5 Novar Dr, Glasgow           | 1-Nov-09   | 10-Aug-10   | 450  | CO93    | Tony Shaw    |


### 2NF
Для того, чтобы таблица находилась в 2NF необходимо и достаточно, чтобы она была в 1NF и каждый не состоящий в потенциальном ключе атрибут полностью функционально зависит от любого потенциального ключа.  

Можно использовать частную формулировку, заменив "потенциальный ключ" на "первичный ключ" и не рассматривать потенциальные ключи.

#### Client

| clientNo | cName         |
|----------|---------------|
| CR76     | John Kay      |
| CR56     | Aline Stewart |

#### Rental

| clientNo | propertyNo | rentStart | rentFinish |
|----------|------------|-----------|------------|
| CR76     | PG4        | 1-Jul-07  | 31-Aug-08  |
| CR76     | PG16       | 1-Sep-08  | 1-Sep-09   |
| CR56     | PG4        | 1-Sep-06  | 10-Jun-07  |
| CR56     | PG36       | 10-Oct-07 | 1-Dec-08   |
| CR56     | PG16       | 1-Nov-09  | 10-Aug-10  |

#### PropertyOwner

| propertyNo | pAddress               | rent | ownerNo | oName       |
|------------|-------------------------|------|---------|-------------|
| PG4        | 6 Lawrence St, Glasgow  | 350  | CO40    | Tina Murphy |
| PG16       | 5 Novar Dr, Glasgow     | 450  | CO93    | Tony Shaw   |
| PG36       | 2 Manor Rd, Glasgow     | 375  | CO93    | Tony Shaw   |


### 3NF
Для того, чтобы таблица находилась в 3NF необходимо и достаточно, чтобы она была в 1NF и 2NF и НИ ОДИН не состоящий в потенциальном ключе атрибут не зависел транзитивно от любого потенциального ключа.  

Можно использовать частную формулировку, заменив "потенциальный ключ" на "первичный ключ" и не рассматривать потенциальные ключи.

PropertyOwner не находится в 3NF из-за транзитивной зависимости:
fd1: propertyNo → pAddress, rent, ownerNo, oName (Primary Key)
fd2: ownerNo → oName (Transitive dependency)

#### PropertyForRent

| propertyNo | pAddress               | rent | ownerNo |
|------------|-------------------------|------|---------|
| PG4        | 6 Lawrence St, Glasgow  | 350  | CO40    |
| PG16       | 5 Novar Dr, Glasgow     | 450  | CO93    |
| PG36       | 2 Manor Rd, Glasgow     | 375  | CO93    |

#### Owner

| ownerNo | oName        |
|---------|--------------|
| CO40    | Tina Murphy  |
| CO93    | Tony Shaw    |

### BCNF
BCNF является более строгой формой, чем 3NF.
Для приведения к BCNF необходимо составить минимальное множество функциональных зависимостей X таблицы, т.е. с выполнением следующих условий:
1. В множестве остаются только зависимости вида A,B,C→D (справа только 1 атрибут).
1. Не должно быть тривиальных зависимостей. Т.е. таких A → D, что D ⊂ A.
1. Мы не можем убрать ни одну зависимость, чтобы не потерять эквивалентность изначальному множеству зависимостей.

Для того, чтобы таблица находилась в BCNF необходимо и достаточно, чтобы в любой зависимости из X детерминант являлся одним из потенциальных ключей таблицы.  

Отличие BCNF от 3NF заключается в том, что если в некоторой A → B получится так, что B - атрибут потенциального ключа, а A - НЕ потенциальный ключ, то такая зависимость позволяет таблице быть в 3NF (так как в формулировке 3NF идет речь о не состоящих в потенциальном ключе атрибутах). Но так как A - НЕ потенциальный ключ, то это противоречит определению BCNF, и следовательно таблица с такой зависимостью не будет в BCNF.

Потенциальное нарушение BCNF может возникнуть, когда:
1. отношение содержит два (или более) составных потенциального ключа ИЛИ
1. потенциальные ключи перекрываются, то есть имеют по крайней мере один общий атрибут.

В примере ниже представлена таблица интервью персонала с клиентами в привязке с помещением.
Сотрудники, участвующие в собеседовании с клиентами, в день собеседования размещаются в определенной комнате. Однако, по мере необходимости в течение рабочего дня комната может быть выделена нескольким сотрудникам. Клиент проходит собеседование только один раз в определенный день, но его могут попросить посетить дополнительные собеседования в более поздние сроки.

#### ClientInterview

| clientNo | interviewDate | interviewTime | staffNo | roomNo |
|----------|---------------|---------------|---------|--------|
| CR76     | 13-May-09     | 10.30         | SG5     | G101   |
| CR56     | 13-May-09     | 12.00         | SG5     | G101   |
| CR74     | 13-May-09     | 12.00         | SG37    | G102   |
| CR56     | 1-Jul-09      | 10.30         | SG5     | G102   |

Тут имеются следующие функциональные зависимости:  
fd1: clientNo, interviewDate → interviewTime, staffNo, roomNo (Primary key)  
fd2: staffNo, interviewDate, interviewTime → clientNo (Candidate key)  
fd3: roomNo, interviewDate, interviewTime → staffNo, clientNo (Candidate key)  
fd4: staffNo, interviewDate → roomNo  

В fd4 детерминант не является потенциальным ключом, что противоречит BCNF.
Для соответствия нужно разбить таблички.

#### Interview

| clientNo | interviewDate | interviewTime | staffNo |
|----------|---------------|---------------|---------|
| CR76     | 13-May-09     | 10.30         | SG5     |
| CR56     | 13-May-09     | 12.00         | SG5     |
| CR74     | 13-May-09     | 12.00         | SG37    |
| CR56     | 1-Jul-09      | 10.30         | SG5     |

#### StaffRoom

| staffNo | interviewDate | roomNo |
|---------|---------------|--------|
| SG5     | 13-May-09     | G101   |
| SG37    | 13-May-09     | G102   |
| SG5     | 1-Jul-09      | G102   |


