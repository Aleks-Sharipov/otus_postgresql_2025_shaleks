
# Резервное копирование и восстановление
## Создаем ВМ/докер c ПГ. Для работы использую ранее развёрнутую ВМ в WSL

```
alex@Alex-PC-Win:~$ sudo -u postgres psql
psql (16.8 (Ubuntu 16.8-0ubuntu0.24.04.1))
Type "help" for help.
postgres=#
```

## Создаю БД, схему и таблицу в ней

```
postgres=# create database otus;
CREATE DATABASE
postgres=# \c otus
You are now connected to database "otus" as user "postgres".
otus=# CREATE SCHEMA postgresotus;
CREATE SCHEMA
otus=# SET search_path TO postgresotus,public;
SET
otus=# SHOW search_path;
     search_path
---------------------
 postgresotus, public
(1 row)
```

## Заполним таблицы автосгенерированными 100 записями
```
otus=# SET search_path TO postgresotus;
SET
otus=# create table students as select generate_series(1, 100) as id, md5(random()::text)::char(10) as fio;
SELECT 100
otus=# \dt
             List of relations
    Schema    |   Name   | Type  |  Owner
--------------+----------+-------+----------
 postgresotus | students | table | postgres
(1 row)
```

## Под линукс пользователем postgres создадим каталог для бэкапов

```
alex@Alex-PC-Win:sudo mkdir -p /tmp/postgresql/backups
alex@Alex-PC-Win:cd /tmp/postgresql/backups
alex@Alex-PC-Win:/tmp/postgresql/backups$ sudo chown -R postgres:postgres /tmp/postgresql/backups
```

## Сделал логический бэкап используя утилиту COPY

```
otus=# \copy students to '/tmp/postgresql/backups/backup_copy.sql';
COPY 100
```

## Проверил наличие файла в директории и что в нём находится

```
alex@Alex-PC-Win:/tmp/postgresql/backups$ cat backup_copy.sql
1       fa428b257e
2       b43f61e213
3       a379472bc2
4       78ee5c11f9
5       426c8ac821
6       756ee77db7
7       c853dc719a
8       1091a567d7
9       71b4de238f
10      36e6cca332
11      1b2949b1b8
12      c9f2b815e9
13      7895483f5e
14      8568f9a471
15      1ad506d0f1
16      8766936cbd
17      db3f68cc94
18      f96b96b841
19      08cc5166b8
20      6bfe558b5d
21      47d12b84ae
22      c2cfb52401
23      84edfa5a97
24      2775bc1015
25      7ff61a3aa6
26      f9cd40b096
27      f1cc06d87e
28      4aee25f8e9
29      fd3ea9aaa3
30      861f3d17ae
31      f44673a719
32      192f1a22d9
33      58ea1f7831
34      2f3ee6d1db
35      dd29284c71
36      f85ba3be5b
37      76f0f0104c
38      4b8bb48769
39      fa70a13ef5
40      3271070938
41      64147c5f2b
42      a8c65f49ed
43      df20e003ea
44      f77cbd0d83
45      4bb4deefce
46      1dc69edeac
47      11ca6860ac
48      931ffaa5c7
49      642f3c1500
50      dcf05e9889
51      65fd0a9a09
52      feaad78a83
53      3f3b2754df
54      903de0f99c
55      7d5fbe2fb3
56      cb24bc35ee
57      8565794ac8
58      d9d3d0f203
59      f10d28df52
60      c781fc097c
61      eefbafd904
62      6f7e2e7643
63      6207462e1e
64      38e5a498e1
65      d83710d767
66      8c7027d58e
67      db73c5bd39
68      771be2cd86
69      c57c4f48e4
70      ba46cfb540
71      e0d8543911
72      a1e996dd4d
73      d4fe9d4dde
74      61b147aa3d
75      e61e8ed43b
76      1b073aa064
77      10c190f049
78      0cb2faffe1
79      c605759bce
80      e2583f0f50
81      33712cfaf2
82      a00aae2228
83      4a4eaf5203
84      4b313e2cd7
85      11fe3a4051
86      0657fae61d
87      614cc6ea34
88      f262c465ab
89      96221c982c
90      87dddaa589
91      09f865608c
92      f42a5dc24f
93      c1841c61c1
94      c4aae0f936
95      ac62cb6e6f
96      edcf20efa5
97      78e1771c3f
98      5e294590f7
99      68898bc665
100     01782db899
```

## Восстановим во 2 таблицу данные из бэкапа

```
otus=# create table students2 (id int, fio char(10));
CREATE TABLE
otus=# \copy students2 from '/tmp/postgresql/backups/backup_copy.sql';
COPY 100
otus=# select * from students2;
otus=# select * from students2 limit 10;
 id |    fio
----+------------
  1 | fa428b257e
  2 | b43f61e213
  3 | a379472bc2
  4 | 78ee5c11f9
  5 | 426c8ac821
  6 | 756ee77db7
  7 | c853dc719a
  8 | 1091a567d7
  9 | 71b4de238f
 10 | 36e6cca332
(10 rows)
```

## Используя утилиту pg_dump создал бэкап в кастомном сжатом формате двух таблиц

```
postgres@Alex-PC-Win:/tmp/postgresql/backups$ sudo -u postgres pg_dump -Fc -t postgresotus.students -t postgresotus.students2 otus > postgresotus_schema.dmp
```

## Используя утилиту pg_restore восстановим в новую БД только вторую таблицу

```
postgres=# create database otus_new;
CREATE DATABASE
postgres=# \c otus_new
You are now connected to database "otus_new" as user "postgres".
otus_new=# CREATE SCHEMA postgresotus;
CREATE SCHEMA
otus_new=# \dt
Did not find any relations.
```

```
postgres@Alex-PC-Win:/tmp/postgresql/backups$ sudo -u postgres pg_restore --table=students2 -d otus_new /tmp/postgresql/backups/postgresotus_schema.dmp
otus_new=# \dt
              List of relations
    Schema    |   Name    | Type  |  Owner
--------------+-----------+-------+----------
 postgresotus | students2 | table | postgres
(1 row)

otus_new=# select * from students2;
```
