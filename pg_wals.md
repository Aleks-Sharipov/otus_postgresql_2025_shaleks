

## Настроил выполнение контрольной точки раз в 30 секунд
```sql
ALTER SYSTEM SET checkpoint_timeout='30s';
SELECT pg_reload_conf();
```

## 10 минут c помощью утилиты pgbench подал нагрузку

# подготовка таблицы
```sql
pgbench -i -h 192.168.31.56 -p 5432 -U postgres test
```

# смотрю lsn

```
test=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/2436F8C0                                  
(1 row)
```

# запускаю pgbench
```
pgbench -h 192.168.31.56 -p 5432 -U postgres -j 2 -P 10 -c 40 -T 600 test
```

# смотрю lsn после pgbench
```
test=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn 
--------------------
 0/493E24F0                                 
(1 row)
```

```
alex@alex-debian:~$ sudo /usr/lib/postgresql/16/bin/pg_waldump -p /var/lib/postgresql/16/main/pg_wal -s 0/2436F8C0 -e 0/493E24F0 |grep CHECKPOINT | wc -l
20                                          
```

# Итого на один checkoint ~ 29МБ


## Проверка данных статистики: все ли контрольные точки выполнялись точно по расписанию
```
test=# select * from pg_stat_bgwriter;
 checkpoints_timed | checkpoints_req | checkpoint_write_time | checkpoint_sync_time | buffers_checkpoint | buffers_clean | maxwritten_clean | buffers_backend | buffers_backend_fsync | buffers_alloc |          stats_reset          
-------------------+-----------------+-----------------------+----------------------+--------------------+---------------+------------------+-----------------+-----------------------+---------------+-------------------------------
                98 |               3 |               1187225 |                  186 |              98962 |             0 |                0 |            8433 |                     0 |         14016 | 2025-05-02 17:35:48.771451+00
(1 row)
```
# buffers_backend_fsync = 0 значит все checkpoints выполнялись по расписанию


## Сравнить tps в синхронном/асинхронном режиме утилитой pgbench. Объяснить полученный результат

# настроил работу в синхронном режиме
```
ALTER SYSTEM SET synchronous_commit = on;
ALTER SYSTEM SET commit_delay = 0;
ALTER SYSTEM SET commit_siblings = 5;
SELECT pg_reload_conf();
```
# запускаю pgbench
```
scaling factor: 1
query mode: simple
number of clients: 40
number of threads: 2
duration: 600 s
number of transactions actually processed: 680857
latency average = 432.455 ms
latency stddev = 301.155 ms
tps = 94.564177 (including connections establishing)
tps = 94.614152 (excluding connections establishing)
```

# настроил работу в асинхронном режиме
```
ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM SET wal_writer_delay = '200ms';
SELECT pg_reload_conf();
```
```
latency average = 479.101 ms
latency stddev = 298.232 ms
tps = 96.102907 (including connections establishing)
tps = 96.122482 (excluding connections establishing)
```

# скорость записи оказалсь +/- одинаковой что в асинхронном режиме, что в синхронном режиме


## Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы

# создаю новый кластер
```
alex@alex-debian:~$ sudo pg_createcluster 16 checksum -- --data-checksum
alex@alex-debian:~$ sudo pg_ctlcluster 16 checksum start
alex@alex-debian:~$ sudo pg_lsclusters 
Ver Cluster  Port Status Owner    Data directory                  Log file
16  checksum 5433 online postgres /var/lib/postgresql/16/checksum /var/log/postgresql/postgresql-16-checksum.log
16  main     5432 online postgres /var/lib/postgresql/16/main     /var/log/postgresql/postgresql-16-main.log
```
# создал таблицу и вставил несколько значений
```
postgres=# create table test(i int);
CREATE TABLE
postgres=# insert into test select s.id from generate_series(1, 100) as s(id); 
postgres=# SELECT pg_relation_filepath('test');
 pg_relation_filepath 
----------------------
 base/5/16384 
(1 row)
```

# выключил кластер
```
postgres=# \q
postgres@alex-debian:~$ exit
logout
alex@alex-debian:~$ sudo pg_ctlcluster 16 checksum stop
```

# изменил пару байт и стартую кластер
```
alex@alex-debian:~$ sudo pg_ctlcluster 16 checksum start
alex@alex-debian:~$ sudo pg_ctlcluster 16 checksum status
pg_ctl: server is running (PID: 31979)
/usr/lib/postgresql/16/bin/postgres "-D" "/var/lib/postgresql/16/checksum" "-c" "config_file=/etc/postgresql/16/checksum/postgresql.conf"
alex@alex-debian:~$ sudo pg_lsclusters 
Ver Cluster  Port Status Owner    Data directory                  Log file
16  checksum 5433 online postgres /var/lib/postgresql/16/checksum /var/log/postgresql/postgresql-16-checksum.log
16  main     5432 online postgres /var/lib/postgresql/16/main     /var/log/postgresql/postgresql-16-main.log
```

# делаю выборку из таблицы
```
postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 31737 but expected 10262
ERROR:  invalid page in block 0 of relation base/5/16384
```
# ошибка, проверяю данные
```
alex@alex-debian:~$ sudo /usr/lib/postgresql/16/bin/pg_checksums -D /var/lib/postgresql/16/checksum
pg_checksums: error: cluster must be shut down
alex@alex-debian:~$ sudo pg_ctlcluster 16 checksum stop
alex@alex-debian:~$ sudo /usr/lib/postgresql/16/bin/pg_checksums -D /var/lib/postgresql/16/checksum
pg_checksums: error: checksum verification failed in file "/var/lib/postgresql/16/checksum/base/5/16384", block 0: calculated checksum 7BFA but block contains 2815
Checksum operation completed
Files scanned:   949
Blocks scanned:  2829
Bad checksums:  1
Data checksum version: 1
```

# отключил checksum и start
```
alex@alex-debian:~$ sudo /usr/lib/postgresql/16/bin/pg_checksums -d -D /var/lib/postgresql/16/checksum
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums disabled in cluster
alex@alex-debian:~$ sudo pg_ctlcluster 16 checksum start
```

# проверяю
```
postgres=# select * from test;
postgres=# select * from test limit 10;
 i  
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
(10 rows)
```
