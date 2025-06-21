
## Виды и устройство репликации в PostgreSQL

1. Создал ВМ №3 на ЯО, otus-vm-1, otus-vm-2, otus-vm-3.
2. Ставим на каждую ВМ postgresql

# ВМ №1

```
otus@otus-vm-1:~$ sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
otus@otus-vm-1:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

# ВМ №2

```
otus@otus-vm-2:~$ sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
otus@otus-vm-2:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

# ВМ №3

```
otus@otus-vm-3:~$ sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15
otus@otus-vm-3:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

# В конфигурации ставим параметр сервера wal_level, устанавливаем logical и перезапускаем кластеры на всех. Проверяем:

```sql
postgres=# show wal_level;
 wal_level
-----------
 replica
(1 row)
```

```sql
postgres=# alter system set wal_level = logical;
ALTER SYSTEM
postgres=# \q
```

```
otus@otus-vm-1:~$ sudo pg_ctlcluster 15 main restart
```

```sql
postgres=# show wal_level;
 wal_level
-----------
 logical
(1 row)
```

* Аналогично на 2 и ВМ №3


## На ВМ №1 создаем таблицы test для записи, test2 для запросов на чтение
# На всех ВМ создал по 2 таблицы test, test2 сразу

*ВМ №1*

```sql
postgres=# create table test (i int);
CREATE TABLE
postgres=#  create table test2 (i int);
CREATE TABLE
postgres=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)
```

*ВМ №2*

```sql
postgres=# create table test (i int);
CREATE TABLE
postgres=#  create table test2 (i int);
CREATE TABLE
postgres=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)
```

*ВМ №3*

```sql
postgres=# create table test (i int);
CREATE TABLE
postgres=# create table test2 (i int);
CREATE TABLE
postgres=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)
```

## Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2

*ВМ №1*

```sql
postgres=# create publication test_pub for table test;
CREATE PUBLICATION
postgres=# \dRp+
                            Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test"
```

*ВМ №2*

```sql
postgres=# create publication test_pub for table test2;
CREATE PUBLICATION
postgres=# \dRp+
                            Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"
```

## Создаём подписки. На ВМ №1 на публикацию test_pub ВМ №2. На ВМ №2 на публикацию test_pub ВМ №1. На ВМ №3 на публикацию test_pub ВМ №1 и test_pub ВМ №2.

*ВМ №1*

```sql
postgres=# create subscription test_sub
connection 'host=158.160.86.74 port=5432 user=postgres dbname=postgres'
publication test_pub with (copy_data = false);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
postgres=# \dRs+
List of subscriptions
   Name   |  Owner   | Enabled | Publication | Binary | Streaming | Two-phase commit | Disable on error | Synchronous commit |                          Conninfo                          | Skip LSN
----------+----------+---------+-------------+--------+-----------+------------------+------------------+--------------------+------------------------------------------------------------+----------
 test_sub | postgres | t       | {test_pub}  | f      | f         | d                | f                | off                | host=158.160.86.74 port=5432 user=postgres dbname=postgres | 0/0
(1 row)
```

*ВМ №2*

```sql
postgres=# create subscription test_sub
connection 'host=158.160.67.175 port=5432 user=postgres dbname=postgres'
publication test_pub with (copy_data = false);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
postgres=# \dRs+
List of subscriptions
   Name   |  Owner   | Enabled | Publication | Binary | Streaming | Two-phase commit | Disable on error | Synchronous commit |
       Conninfo                           | Skip LSN
----------+----------+---------+-------------+--------+-----------+------------------+------------------+--------------------+-------------------------------------------------------------+----------
 test_sub | postgres | t       | {test_pub}  | f      | f         | d                | f                | off                | host=158.160.67.175 port=5432 user=postgres dbname=postgres | 0/0
(1 row)
```

*ВМ №3*

```sql
postgres=# create subscription test_sub1
connection 'host=158.160.67.175 port=5432 user=postgres dbname=postgres'
publication test_pub with (copy_data = false);
NOTICE:  created replication slot "test_sub1" on publisher
CREATE SUBSCRIPTION
postgres=# create subscription test_sub2
connection 'host=158.160.86.74 port=5432 user=postgres dbname=postgres'
publication test_pub with (copy_data = false);
NOTICE:  created replication slot "test_sub2" on publisher
CREATE SUBSCRIPTION
postgres=# \dRs+
List of subscriptions
   Name    |  Owner   | Enabled | Publication | Binary | Streaming | Two-phase commit | Disable on error | Synchronous commit |
        Conninfo                           | Skip LSN
-----------+----------+---------+-------------+--------+-----------+------------------+------------------+--------------------+-------------------------------------------------------------+----------
 test_sub1 | postgres | t       | {test_pub}  | f      | f         | d                | f                | off                | host=158.160.67.175 port=5432 user=postgres dbname=postgres | 0/0
 test_sub2 | postgres | t       | {test_pub}  | f      | f         | d                | f                | off                | host=158.160.86.74 port=5432 user=postgres dbname=postgres  | 0/0
(2 rows)
```


## Проверяем. Bставим данные на ВМ №1 в таблицу test и данные на ВМ №2 в таблицу test2

*ВМ №1*

```sql
postgres=# INSERT INTO test(i) SELECT random() FROM generate_series(1, 10);
INSERT 0 10
postgres=# select * from test;
 i
---
 0
 0
 1
 1
 0
 1
 0
 0
 0
 0
(10 rows)
```

```sql
postgres=# INSERT INTO test(i) SELECT generate_series(1, 10);
INSERT 0 10
postgres=# select * from test;
 i
----
  0
  0
  1
  1
  0
  1
  0
  0
  0
  0
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
(20 rows)
```

*ВМ №2*

```sql
postgres=# INSERT INTO test2(i) SELECT generate_series(11, 20);
INSERT 0 10
postgres=# select * from test2;
 i
----
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
(10 rows)
```

## На ВМ №1 смотрим test2 со второй, на 2-й test с первой, на 3-й test с 1-й и test2 со 2-й

*ВМ №1*

```sql
postgres=# select * from public.test2;
 i
----
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
(10 rows)
```

*ВМ №2*

```sql
postgres=# select * from public.test;
 i
----
  0
  0
  1
  1
  0
  1
  0
  0
  0
  0
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
(20 rows)
```

*ВМ №3*

```sql
postgres=# select * from public.test;
 i
----
  0
  0
  1
  1
  0
  1
  0
  0
  0
  0
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
(20 rows)
```

```sql
postgres=# select * from public.test2;
 i
----
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
(10 rows)
```

*Все работает*

