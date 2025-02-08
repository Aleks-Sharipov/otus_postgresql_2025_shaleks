Подключился по ssh к серверу PostgreSQL, запустил вторую сессию.
```bash
ssh -i c:\ssh\otus_pgvm otus@158.160.17.212
```

Запустил везде psql под пользователем postgres:
```bash
psql -h 158.160.17.212 -U postgres
```

Выключил auto commit:
```sql
\set AUTOCOMMIT off
```

В первой сессии создал новую таблицу и наполнил её данными:
```sql
create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
```

Посмотрел текущий уровень изоляции транзакций:
```sql
show transaction isolation level;
```

Результат:
```sql
 transaction_isolation
-----------------------
 read committed
(1 строка)
```

Начал новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции.

В первой сессии добавил новую запись:
```sql
insert into persons (first_name, second_name) values('sergey', 'sergeev');
```

Запустил SELECT во второй сессии:
```sql
select * from persons;
```

Результат не содержит новую запись:
```sql
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 строк)
```

Завершил первую транзакцию выражением COMMIT:
```sql
commit;
```

Теперь результат содержит новую запись, так как мы завершили транзакцию в первой сессии и запись стала видна во второй:
```sql
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 строк)
```

завершите транзакцию во второй сессии
начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;
в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
сделать select* from persons во второй сессии*
видите ли вы новую запись и если да то почему?
завершить первую транзакцию - commit;
сделать select from persons во второй сессии
видите ли вы новую запись и если да то почему?
завершить вторую транзакцию
сделать select * from persons во второй сессии
видите ли вы новую запись и если да то почему?
