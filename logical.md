
Для начала создадим новый кластер PostgresSQL. У меня 13:

```bash
sudo pg_createcluster 13 main3
```
Стартуем:

```bash
sudo pg_ctlcluster 13 main3 start
```
Проверяем:

```bash
pg_lsclusters
```

Зашли в созданный кластер под пользователем postgres, проверили права:

```sql
postgres=# \du
Список ролей
-[ RECORD 1 ]
Имя роли   | postgres
Атрибуты   | Суперпользователь, Создаёт роли, Создаёт БД, Репликация, Пропускать RLS
Член ролей | {}
```

Создал новую базу данных testdb:

```sql
postgres=# create database testdb;
CREATE DATABASE
```

Зашли в созданную базу данных под пользователем postgres:

```sql
postgres=# \c testdb
psql (13.20 (Debian 13.20-0+deb11u1), сервер 13.19 (Debian 13.19-0+deb11u1))
Вы подключены к базе данных "testdb" как пользователь "postgres".
```

Создал новую схему testnm:

```sql
testdb=# create schema testnm;
CREATE SCHEMA
```

Проверяем:

```sql
testdb=# \dn
    Список схем
  Имя   | Владелец
--------+----------
 public | postgres
 testnm | postgres
(2 строки)
```

Создал новую таблицу t1 с одной колонкой c1 типа integer:

```sql
testdb=# create table t1 (c1 int);
CREATE TABLE
```

Добавили строку со значением c1=1 в таблицу t1:

```sql
testdb=# insert into t1 values(1);
INSERT 0 1
```

Создал новую роль readonly:

```sql
testdb=# create role readonly;
CREATE ROLE
```

Проверяем:

```sql
Список ролей
-[ RECORD 1 ]
Имя роли   | postgres
Атрибуты   | Суперпользователь, Создаёт роли, Создаёт БД, Репликация, Пропускать RLS
Член ролей | {}
-[ RECORD 2 ]
Имя роли   | readonly
Атрибуты   | Вход запрещён
Член ролей | {}
```

Добавил для роли readonly право на подключение к базе данных testdb:

```sql
testdb=# alter role readonly login;
ALTER ROLE
```

Проверим:

```sql
testdb=# \du
Имя роли   | postgres
Атрибуты   | Суперпользователь, Создаёт роли, Создаёт БД, Репликация, Пропускать RLS
Член ролей | {}
-----------+---------
Имя роли   | readonly
Атрибуты   |
Член ролей | {}
```

Добавил для роли readonly право использование схемы testnm
и на select для всех таблиц этой схемы:

```sql
testdb=# grant USAGE on SCHEMA testnm to readonly;
GRANT

testdb=# grant SELECT on all tables in schema testnm to readonly;
GRANT
```

Создал пользователя testread с паролем test123 и дал
роль readonly пользователю testread:

```sql
testdb=# create role testread with password 'test123';
CREATE ROLE
testdb=# grant readonly to testread;
GRANT ROLE
```

Подключился (у-у-х!) под пользователем testread к базе данных testdb:

```sql
testdb=> select current_user, session_user;
 current_user | session_user
--------------+--------------
 testread     | testread
(1 строка)
```

Сделал select * from t1 и получилось, т.к. создал таблицу ранее, чем добавил пользователя (возможно!):

```sql
testdb=> select * from t1;
 c1
----
  1
(1 строка)
```

Подключился к базе данных testdb под пользователем postgres и
удалил таблицу t1:

```sql
testdb=> \c testdb postgres
Пароль пользователя postgres:
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=# drop table t1;
DROP TABLE
```

Создал её заново, но уже с явным указанием имени схемы testnm и вставил пару строк:

```sql
testdb=# create table testnm.t1(c1 int);
CREATE TABLE
testdb=# insert into testnm.t1 values (123);
INSERT 0 1
testdb=# insert into testnm.t1 values (1233);
INSERT 0 1
```

Зашёл под пользователем testread в базу данных testdb:

```sql
testdb=# \c testdb testread
Пароль пользователя testread:
Вы подключены к базе данных "testdb" как пользователь "testread".
testdb=>
```

сделал select * from testnm.t1 и нет доступа!:

```sql
testdb=> select * from testnm.t1;
ОШИБКА:  нет доступа к таблице t1
```

Потому что права не наследовались на вновь созданные таблицы. Необходимо добавить привелегии,
чтобы вновь созданные таблицы также были доступны пользователю:

```sql
testdb=> \c testdb postgres
Пароль пользователя postgres:
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=# ALTER default privileges in SCHEMA testnm grant SELECT on TABLES to readonly;
ALTER DEFAULT PRIVILEGES
```

Добавил привелегии, переподключился к базе под testread и сделал вновь запрос на select:
```sql
testdb=# \c testdb testread
Пароль пользователя testread:
Вы подключены к базе данных "testdb" как пользователь "testread".
testdb=> select * from testnm.t1;
ОШИБКА:  нет доступа к таблице t1
t
```

Увы, опять ошибка! Понял в чём, добавил права на select к схеме testnm для
пользователя testread:

```sql
testdb=# grant select on all tables in schema testnm to testread;
GRANT
testdb=# \c testdb testread
Пароль пользователя testread:
Вы подключены к базе данных "testdb" как пользователь "testread".
testdb=> select * from testnm.t1;
-[ RECORD 1 ]
c1 | 123
-[ RECORD 2 ]
c1 | 1233
```

Выполнил команду create table и вставил в неё данные insert into. Получилось:

```sql
testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
```
Да, по умолчанию, без явного указания схемы в которой будет создаваться таблица,
она создаётся в схеме public. Следовательно, права на создание в схеме public есть
по умолчанию у роли public, а эта роль также по умолчанию добавляется всем новым
пользователям. Для того, чтобы этого не происходило, запретим роли public создавать
объекты базы данных и отзовём все права роли public на базу testdb:

```sql
testdb=> \c testdb postgres
Пароль пользователя postgres:
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=# REVOKE CREATE on SCHEMA public FROM public;
REVOKE
testdb=# REVOKE ALL on DATABASE testdb FROM public;
REVOKE
```
Теперь пробую подключиться под testread, а разрешения нет:

```sql
testdb=# \c testdb testread
Пароль пользователя testread:
ВАЖНО:  доступ к базе "testdb" запрещён
ПОДРОБНОСТИ:  Пользователь не имеет привилегии CONNECT.
Сохранено предыдущее подключение
```
Дал права на CONNECT для роли readonly, и удалось подключиться к базе testdb под testread:

```sql
testdb=# grant CONNECT on DATABASE testdb to readonly;
GRANT
testdb=# \c testdb testread
Пароль пользователя testread:
Вы подключены к базе данных "testdb" как пользователь "testread".
testdb=> 
```

Произошло всё из-за того, что я отозвал все привелегии (REVOKE ALL) для роли public,
а разрешений для роли testread на подключение к testdb отдельно не давал. 
Теперь попробуем выполнить команду create table t3 и вставить в неё записи insert into:

```sql
testdb=> create table t3(c1 integer); insert into t3 values (2);
ОШИБКА:  нет доступа к схеме public
СТРОКА 1: create table t3(c1 integer);
```

Ошибка, нет прав. Для этого необходимо дать права для роли testread на схему public:

```sql
testdb=> \c testdb postgres
Пароль пользователя postgres:
Вы подключены к базе данных "testdb" как пользователь "postgres".
testdb=# GRANT USAGE ON SCHEMA public TO testread;
GRANT
```

А теперь всё получилось:

```sql
testdb=> create table t3(c1 integer); insert into t3 values (2);
CREATE TABLE
INSERT 0 1
testdb=> select * from t3;
-[ RECORD 1 ]
c1 | 2
```
