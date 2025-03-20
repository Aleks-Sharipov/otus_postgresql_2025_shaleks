
***
## 1.
***

Сделали настройки:

```sql
locks=# show log_lock_waits ;
-[ RECORD 1 ]--+----
log_lock_waits | off
```

```sql
ALTER SYSTEM SET log_lock_waits to 'ON';
```

```sql
locks=# show log_lock_waits ;
-[ RECORD 1 ]--+----
log_lock_waits | off
```

```sql
locks=# select pg_reload_conf();
-[ RECORD 1 ]--+--
pg_reload_conf | t
```

```sql
locks=# show log_lock_waits ;
-[ RECORD 1 ]--+---
log_lock_waits | on
```

```sql
ALTER SYSTEM SET deadlock_timeout = '200ms';
```

Запустили транзакции в 3х сессиях:

```sql
SELECT * FROM locks_v WHERE pid = 26985;
SELECT * FROM locks_v WHERE pid = 27004;
SELECT * FROM locks_v WHERE pid = 27022;
```

В 1й сессии pid = 26985 открыли транзакцию и запустили update строки, в результате появилась блокировка RowExclusiveLock
Во 2й сессии pid = 27004 также открыли транзакцию и запустили update строки, в результате появилась блокировка RowExclusiveLock и взаимная блокировка на уровне строк (tuple) 1й транзакции с pid = 26985 с типом ExclusiveLock и блокировка ShareLock, которая ещё не наложена, т.к. 1я транзакция update ещё не завершена.
В 3й сессии pid = 27022 также открыли транзакцию и запустили update строки, в ней также есть блокировка RowExclusiveLock, а также взаимная блокировка на уровне строк (tuple) от 2й транзакции с pid = 27004, но в данном случае она ещё ожидающая, не наложена.

```sql
SELECT * FROM locks_v WHERE pid = 26985;
  pid  |   locktype    |   lockid   |       mode       | granted
-------+---------------+------------+------------------+---------
 26985 | relation      | otus_table | RowExclusiveLock | t
 26985 | transactionid | 515        | ExclusiveLock    | t
(2 строки)
```

```sql
SELECT * FROM locks_v WHERE pid = 27004;
  pid  |   locktype    |    lockid     |       mode       | granted
-------+---------------+---------------+------------------+---------
 27004 | relation      | otus_table    | RowExclusiveLock | t
 27004 | tuple         | otus_table:20 | ExclusiveLock    | t
 27004 | transactionid | 516           | ExclusiveLock    | t
 27004 | transactionid | 515           | ShareLock        | f
(4 строки)
```

```sql
SELECT * FROM locks_v WHERE pid = 27022;
  pid  |   locktype    |    lockid     |       mode       | granted
-------+---------------+---------------+------------------+---------
 27022 | relation      | otus_table    | RowExclusiveLock | t
 27022 | tuple         | otus_table:20 | ExclusiveLock    | f
 27022 | transactionid | 517           | ExclusiveLock    | t
(3 строки)
```

```sql
 locktype |  relation  |       mode       | granted |  pid  | wait_for
----------+------------+------------------+---------+-------+----------
 relation | otus_table | RowExclusiveLock | t       | 26985 | {}
 relation | otus_table | RowExclusiveLock | t       | 27004 | {26985}
 tuple    | otus_table | ExclusiveLock    | t       | 27004 | {26985}
 relation | otus_table | RowExclusiveLock | t       | 27022 | {27004}
 tuple    | otus_table | ExclusiveLock    | f       | 27022 | {27004}
(5 строк)
```


Завершили 1ю транзакцию и получили:

```sql
SELECT * FROM locks_v WHERE pid = 26985;
pid | locktype | lockid | mode | granted
-----+----------+--------+------+---------
(0 строк)
```

```sql
SELECT * FROM locks_v WHERE pid = 27004;
  pid  |   locktype    |   lockid   |       mode       | granted
-------+---------------+------------+------------------+---------
 27004 | relation      | otus_table | RowExclusiveLock | t
 27004 | transactionid | 516        | ExclusiveLock    | t
(2 строки)
```

```sql
SELECT * FROM locks_v WHERE pid = 27022;
  pid  |   locktype    |   lockid   |       mode       | granted
-------+---------------+------------+------------------+---------
 27022 | relation      | otus_table | RowExclusiveLock | t
 27022 | transactionid | 516        | ShareLock        | f
 27022 | transactionid | 517        | ExclusiveLock    | t
(3 строки)
```

Завершили 2ю транзакцию и получили:

```sql
SELECT * FROM locks_v WHERE pid = 27004;
 pid | locktype | lockid | mode | granted
-----+----------+--------+------+---------
(0 строк)
```

```sql
SELECT * FROM locks_v WHERE pid = 27022;
  pid  |   locktype    |   lockid   |       mode       | granted
-------+---------------+------------+------------------+---------
 27022 | relation      | otus_table | RowExclusiveLock | t
 27022 | transactionid | 517        | ExclusiveLock    | t
(2 строки)
```



***
## 2.
***

Да, можно изучая журнал транзакции посмотреть, какие были блокировки. Единственное, нет понятия что была взаимная блокировка на уровне строки.


2025-03-20 19:09:00.353 MSK [27004] postgres@locks СООБЩЕНИЕ:  процесс 27004 продолжает ожидать в режиме ShareLock блокировку "транзакция 515" в течение 200.348 мс
2025-03-20 19:09:00.353 MSK [27004] postgres@locks ПОДРОБНОСТИ:  Process holding the lock: 26985. Wait queue: 27004.
2025-03-20 19:09:00.353 MSK [27004] postgres@locks КОНТЕКСТ:  при изменении кортежа (54108,20) в отношении "otus_table"
2025-03-20 19:09:00.353 MSK [27004] postgres@locks ОПЕРАТОР:  update otus_table set client = '2' where id = 10000;
2025-03-20 19:09:01.449 MSK [27022] postgres@locks СООБЩЕНИЕ:  процесс 27022 продолжает ожидать в режиме ExclusiveLock блокировку "кортеж (54108,20) отношения 16385 базы данных 16384" в течение 200.346 мс
2025-03-20 19:09:01.449 MSK [27022] postgres@locks ПОДРОБНОСТИ:  Process holding the lock: 27004. Wait queue: 27022.
2025-03-20 19:09:01.449 MSK [27022] postgres@locks ОПЕРАТОР:  update otus_table set client = '3' where id = 10000;
2025-03-20 20:16:23.170 MSK [27004] postgres@locks СООБЩЕНИЕ:  процесс 27004 получил в режиме ShareLock блокировку "транзакция 515" через 4043017.866 мс
2025-03-20 20:16:23.170 MSK [27004] postgres@locks КОНТЕКСТ:  при изменении кортежа (54108,20) в отношении "otus_table"
2025-03-20 20:16:23.170 MSK [27004] postgres@locks ОПЕРАТОР:  update otus_table set client = '2' where id = 10000;
2025-03-20 20:16:23.171 MSK [27022] postgres@locks СООБЩЕНИЕ:  процесс 27022 получил в режиме ExclusiveLock блокировку "кортеж (54108,20) отношения 16385 базы данных 16384" через 4041921.844 мс
2025-03-20 20:16:23.171 MSK [27022] postgres@locks ОПЕРАТОР:  update otus_table set client = '3' where id = 10000;
2025-03-20 20:16:23.371 MSK [27022] postgres@locks СООБЩЕНИЕ:  процесс 27022 продолжает ожидать в режиме ShareLock блокировку "транзакция 516" в течение 200.246 мс
2025-03-20 20:16:23.371 MSK [27022] postgres@locks ПОДРОБНОСТИ:  Process holding the lock: 27004. Wait queue: 27022.
2025-03-20 20:16:23.371 MSK [27022] postgres@locks КОНТЕКСТ:  при перепроверке изменённого кортежа (1,4) в отношении "otus_table"
2025-03-20 20:16:23.371 MSK [27022] postgres@locks ОПЕРАТОР:  update otus_table set client = '3' where id = 10000;
2025-03-20 20:19:41.817 MSK [27022] postgres@locks СООБЩЕНИЕ:  процесс 27022 получил в режиме ShareLock блокировку "транзакция 516" через 198645.697 мс
2025-03-20 20:19:41.817 MSK [27022] postgres@locks КОНТЕКСТ:  при перепроверке изменённого кортежа (1,4) в отношении "otus_table"
2025-03-20 20:19:41.817 MSK [27022] postgres@locks ОПЕРАТОР:  update otus_table set client = '3' where id = 10000;



***
## 3.
***

Запустили в трёх сессиях update таблицы. Во 2й сессии возникла ExclusiveLock, но через 200мс процесс 27004 завершился из-за взаимоблокировки (deadlock_timeout=200), а 3я транзакция осталась, т.к. в момент запуска 1й транзакции, она ещё не была активирована. По завершении 1й транзакции. запустилась 3я транзакция.
1я и 2я транзакции не заблокировали друг друга.

```sql
SELECT * FROM locks_v WHERE pid = 26985;
  pid  |   locktype    |    lockid    |       mode       | granted
-------+---------------+--------------+------------------+---------
 26985 | relation      | otus_table   | RowExclusiveLock | t
 26985 | tuple         | otus_table:1 | ExclusiveLock    | t
 26985 | transactionid | 518          | ExclusiveLock    | t
 26985 | transactionid | 519          | ShareLock        | f
(4 строки)
```

```sql
SELECT * FROM locks_v WHERE pid = 27004;
  pid  |   locktype    |   lockid   |       mode       | granted
-------+---------------+------------+------------------+---------
 27004 | relation      | otus_table | RowExclusiveLock | t
 27004 | transactionid | 519        | ExclusiveLock    | t
(2 строки)
```

```sql
SELECT * FROM locks_v WHERE pid = 27022;
  pid  |   locktype    |    lockid    |       mode       | granted
-------+---------------+--------------+------------------+---------
 27022 | relation      | otus_table   | RowExclusiveLock | t
 27022 | transactionid | 520          | ExclusiveLock    | t
 27022 | tuple         | otus_table:1 | ExclusiveLock    | f
(3 строки)
```

```sql
 locktype |  relation  |       mode       | granted |  pid  | wait_for
----------+------------+------------------+---------+-------+----------
 relation | otus_table | RowExclusiveLock | t       | 26985 | {27004}
 tuple    | otus_table | ExclusiveLock    | t       | 26985 | {27004}
 relation | otus_table | RowExclusiveLock | t       | 27004 | {}
 relation | otus_table | RowExclusiveLock | t       | 27022 | {26985}
 tuple    | otus_table | ExclusiveLock    | f       | 27022 | {26985}
(5 строк)
```

И данные логов говорят об аналогичном:

2025-03-20 20:33:11.126 MSK [26985] postgres@locks СООБЩЕНИЕ:  процесс 26985 продолжает ожидать в режиме ShareLock блокировку "транзакция 519" в течение 200.146 мс
2025-03-20 20:33:11.126 MSK [26985] postgres@locks ПОДРОБНОСТИ:  Process holding the lock: 27004. Wait queue: 26985.
2025-03-20 20:33:11.126 MSK [26985] postgres@locks КОНТЕКСТ:  при изменении кортежа (54054,1) в отношении "otus_table"
2025-03-20 20:33:11.126 MSK [26985] postgres@locks ОПЕРАТОР:  update otus_table set client = '1';
2025-03-20 20:33:11.140 MSK [27022] postgres@locks СООБЩЕНИЕ:  процесс 27022 продолжает ожидать в режиме ExclusiveLock блокировку "кортеж (54054,1) отношения 16385 базы данных 16384" в течение 200.070 мс
2025-03-20 20:33:11.140 MSK [27022] postgres@locks ПОДРОБНОСТИ:  Process holding the lock: 26985. Wait queue: 27022.
2025-03-20 20:33:11.140 MSK [27022] postgres@locks ОПЕРАТОР:  update otus_table set client = '3';
2025-03-20 20:35:51.923 MSK [27004] postgres@locks СООБЩЕНИЕ:  процесс 27004 обнаружил взаимоблокировку, ожидая в режиме ShareLock блокировку "транзакция 518" в течение 200.193 мс
2025-03-20 20:35:51.923 MSK [27004] postgres@locks ПОДРОБНОСТИ:  Process holding the lock: 26985. Wait queue: .
2025-03-20 20:35:51.923 MSK [27004] postgres@locks КОНТЕКСТ:  при изменении кортежа (0,1) в отношении "otus_table"
2025-03-20 20:35:51.923 MSK [27004] postgres@locks ОПЕРАТОР:  update otus_table set client = '2';
2025-03-20 20:35:51.974 MSK [27004] postgres@locks ОШИБКА:  обнаружена взаимоблокировка
2025-03-20 20:35:51.974 MSK [27004] postgres@locks ПОДРОБНОСТИ:  Процесс 27004 ожидает в режиме ShareLock блокировку "транзакция 518"; заблокирован процессом 26985.
        Процесс 26985 ожидает в режиме ShareLock блокировку "транзакция 519"; заблокирован процессом 27004.
        Процесс 27004: update otus_table set client = '2';
        Процесс 26985: update otus_table set client = '1';
2025-03-20 20:35:51.974 MSK [27004] postgres@locks ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
2025-03-20 20:35:51.974 MSK [27004] postgres@locks КОНТЕКСТ:  при изменении кортежа (0,1) в отношении "otus_table"
2025-03-20 20:35:51.974 MSK [27004] postgres@locks ОПЕРАТОР:  update otus_table set client = '2';
2025-03-20 20:35:51.974 MSK [26985] postgres@locks СООБЩЕНИЕ:  процесс 26985 получил в режиме ShareLock блокировку "транзакция 519" через 161048.629 мс
2025-03-20 20:35:51.974 MSK [26985] postgres@locks КОНТЕКСТ:  при изменении кортежа (54054,1) в отношении "otus_table"
2025-03-20 20:35:51.974 MSK [26985] postgres@locks ОПЕРАТОР:  update otus_table set client = '1';
2025-03-20 20:35:51.977 MSK [27022] postgres@locks СООБЩЕНИЕ:  процесс 27022 получил в режиме ExclusiveLock блокировку "кортеж (54054,1) отношения 16385 базы данных 16384" через 161036.434 мс
2025-03-20 20:35:51.977 MSK [27022] postgres@locks ОПЕРАТОР:  update otus_table set client = '3';
2025-03-20 20:35:52.177 MSK [27022] postgres@locks СООБЩЕНИЕ:  процесс 27022 продолжает ожидать в режиме ShareLock блокировку "транзакция 518" в течение 200.163 мс
2025-03-20 20:35:52.177 MSK [27022] postgres@locks ПОДРОБНОСТИ:  Process holding the lock: 26985. Wait queue: 27022.
2025-03-20 20:35:52.177 MSK [27022] postgres@locks КОНТЕКСТ:  при изменении кортежа (54054,1) в отношении "otus_table"
2025-03-20 20:35:52.177 MSK [27022] postgres@locks ОПЕРАТОР:  update otus_table set client = '3';
2025-03-20 20:39:57.978 MSK [27022] postgres@locks СООБЩЕНИЕ:  процесс 27022 получил в режиме ShareLock блокировку "транзакция 518" через 246001.505 мс
2025-03-20 20:39:57.978 MSK [27022] postgres@locks КОНТЕКСТ:  при изменении кортежа (54054,1) в отношении "otus_table"
2025-03-20 20:39:57.978 MSK [27022] postgres@locks ОПЕРАТОР:  update otus_table set client = '3';
