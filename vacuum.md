Работаю с локальным кластером на старом ноуте с 3Гб ОЗУ, 900 Гб HDD и 2х ядерным CPU Intel Pentium B960 2.2ГГц, Deb11:)

***
## 1. pgbench
***


Запустил загрузку тестовыми данными:
```sql
pgbench -h 192.168.31.56 -p 5433 -U postgres -i otus
```

```sql
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.12 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 4.17 s (drop tables 1.11 s, create tables 0.45 s, client-side generate 0.43 s, vacuum 0.45 s, primary keys 1.72 s).
```


Запустил тест:
```sql
alex@alex-debian:~$ pgbench -h 192.168.31.56 -p 5433 -c8 -P 6 -T 60 -U postgres otus
Password:
starting vacuum...end.
progress: 6.0 s, 15.7 tps, lat 455.003 ms stddev 193.546
progress: 12.0 s, 11.2 tps, lat 722.612 ms stddev 438.122
progress: 18.0 s, 16.2 tps, lat 504.774 ms stddev 222.540
progress: 24.0 s, 13.2 tps, lat 597.941 ms stddev 389.683
progress: 30.0 s, 18.2 tps, lat 441.957 ms stddev 178.824
progress: 36.0 s, 15.0 tps, lat 452.230 ms stddev 241.242
progress: 42.0 s, 15.0 tps, lat 588.960 ms stddev 505.813
progress: 48.0 s, 14.5 tps, lat 579.599 ms stddev 217.867
progress: 54.0 s, 13.0 tps, lat 615.714 ms stddev 398.765
progress: 60.0 s, 16.7 tps, lat 481.070 ms stddev 152.119
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 60 s
number of transactions actually processed: 899
latency average = 533.313 ms
latency stddev = 316.630 ms
tps = 14.879514 (including connections establishing)
tps = 14.888454 (excluding connections establishing)
```

Сменил настройки конф. файла по инструкции и запустил тест ещё раз:

```sql
starting vacuum...end.
progress: 6.0 s, 16.0 tps, lat 452.774 ms stddev 174.387
progress: 12.0 s, 16.8 tps, lat 476.193 ms stddev 189.926
progress: 18.0 s, 14.5 tps, lat 499.716 ms stddev 228.046
progress: 24.0 s, 14.3 tps, lat 591.555 ms stddev 344.637
progress: 30.0 s, 16.7 tps, lat 500.726 ms stddev 185.560
progress: 36.0 s, 16.8 tps, lat 472.887 ms stddev 185.999
progress: 42.0 s, 16.3 tps, lat 489.626 ms stddev 205.087
progress: 48.0 s, 13.8 tps, lat 546.888 ms stddev 317.082
progress: 54.0 s, 13.2 tps, lat 604.666 ms stddev 418.589
progress: 60.0 s, 14.2 tps, lat 595.909 ms stddev 303.566
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 60 s
number of transactions actually processed: 924
latency average = 519.706 ms
latency stddev = 266.234 ms
tps = 15.220005 (including connections establishing)
tps = 15.229125 (excluding connections establishing)
```

Стало хуже, немного. Почему? Хороший вопрос, может из-за wal_buffers, он из всех параметров стал меньше. Не знаю.


***
## 2. заполняем таблицу данными
***

Создал таблицу с стекст полем и наполнил её данными:

```sql
INSERT INTO otus_table (id,client)
SELECT g.id, 'Maumussx'
FROM generate_series(1, 20000000) AS g (id) ;
INSERT 0 20000000
```

Посмотрим её размер:

```sql
select pg_total_relation_size('otus_table');
-[ RECORD 1 ]----------+----------
pg_total_relation_size | 885899264

select pg_size_pretty( pg_total_relation_size('otus_table'));
-[ RECORD 1 ]--+-------
pg_size_pretty | 845 MB
```


***
## 3. тестируем с автовакуумом вкл.
***

Запускаем update:

```sql
update otus_table set client = replace(client,'M','H');
UPDATE 20000000
```

Проверяем размер:

```sql
select pg_size_pretty( pg_total_relation_size('otus_table'));
-[ RECORD 1 ]--+--------
pg_size_pretty | 1690 MB
```

Проверяем мёртвые тюплы:

```sql
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_TABLEs WHERE relname = 'otus_table';
-[ RECORD 1 ]---+------------------------------
relname         | otus_table
n_live_tup      | 20000000
n_dead_tup      | 20000000
ratio%          | 99
last_autovacuum | 2025-03-17 19:41:25.846241+03
```

Ещё несколько обновлений и проверка:

```sql
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_TABLEs WHERE relname = 'otus_table';
-[ RECORD 1 ]---+------------------------------
relname         | otus_table
n_live_tup      | 19983456
n_dead_tup      | 81911867
ratio%          | 314
last_autovacuum | 2025-03-17 19:55:08.045589+03
```

```sql
select pg_size_pretty( pg_total_relation_size('otus_table'));
-[ RECORD 1 ]--+--------
pg_size_pretty | 4342 MB
```

В итоге, после всех обновлений размер табл увеличился в 5 раз. Прошёл автовакуум и мёртвые тюплы пропали

```sql
-[ RECORD 1 ]---+------------------------------
relname         | otus_table
n_live_tup      | 19983456
n_dead_tup      | 0
ratio%          | 314
last_autovacuum | 2025-03-17 20:05:56.843747+03
```


***
## 4. тестируем с автовакуумом выкл.
***

Выключил автовакуум:

```sql
ALTER TABLE otus_table SET (autovacuum_enabled = off);
```

Обновлял несколько раз данные, проверяю статистику по тюплам и размер:

```sql
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_TABLEs WHERE relname = 'otus_table';
-[ RECORD 1 ]---+------------------------------
relname         | otus_table
n_live_tup      | 19983456
n_dead_tup      | 121911424
ratio%          | 610
last_autovacuum | 2025-03-17 20:05:56.843747+03
```

```sql
SELECT pg_relation_filepath('otus_table');
 -[ RECORD 1 ]--------+-----------------
pg_relation_filepath | base/16384/16387
```

```sql
select pg_size_pretty( pg_total_relation_size('otus_table'));
 -[ RECORD 1 ]--+--------
pg_size_pretty | 5875 MB
```

После всех обновлений размер табл увеличился ещё. Размер не уменьшался и мёртвые тюплы не исчезали, т.к. автовакуум у нас выключен.
Из-за этого и рост размеров табл.
Запустил очистку

```sql
vacuum full otus_table ;
```

В результате данные сжались, размер табл вернулся в исходное состояние, а мёртвые тюплы не очистились:

```sql
-[ RECORD 1 ]---+------------------------------
relname         | otus_table
n_live_tup      | 19856025
n_dead_tup      | 122991146
ratio%          | 619
last_autovacuum | 2025-03-17 20:05:56.843747+03
```

```sql
select pg_size_pretty( pg_total_relation_size('otus_table'));
 -[ RECORD 1 ]--+-------
pg_size_pretty | 766 MB
```

Включил автовакуум, всё очистилось:

```sql
ALTER TABLE student SET (autovacuum_enabled = on);

-[ RECORD 1 ]---+------------------------------
relname         | otus_table
n_live_tup      | 20000034
n_dead_tup      | 0
ratio%          | 0
last_autovacuum | 2025-03-18 23:04:55.788039+03

select pg_size_pretty( pg_total_relation_size('otus_table'));            -[ RECORD 1 ]--+-------
pg_size_pretty | 766 MB
```

