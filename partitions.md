
***
## 1. Выбор таблицы для секционирования
***



Выброр мой пал на одну из самых больших таблиц bookings.ticket_flights.
Почему её надо секционировать, потому что с увеличением количества данных, производительность запросов будет снижаться.
Также часть данных из этой таблицы со временем можно будет отключить или перенести на медленные СХД.
Так как основным значением в этой таблице является ticket_no (номер билета), решил секционировать по нему.
Номер билета по сути является автоинкрементным, возрастает с количеством продаж.
Старые номера билетов можно сказать со временем будут не актуальны и их можно архивировать.

Провёл анализ запроса по обычной таблице. Индекс используется. Планировщик показал Index Scan при различных запросах к таблице:

```postgresql
explain analyze
SELECT ticket_no, flight_id, fare_conditions, amount
FROM bookings.ticket_flights
where ticket_no like '0005430727805';
```

```
Index Scan using ticket_flights_pkey on ticket_flights  (cost=0.43..16.45 rows=3 width=32) (actual time=9.100..9.101 rows=0 loops=1)
  Index Cond: (ticket_no = '0005430727805'::bpchar)
  Filter: (ticket_no ~~ '0005430727805'::text)
Planning Time: 0.139 ms
Execution Time: 9.139 ms
```

```
Index Scan using ticket_flights_pkey on ticket_flights  (cost=0.43..16.46 rows=3 width=32) (actual time=0.051..0.058 rows=4 loops=1)
  Index Cond: (ticket_no = '0005433569575'::bpchar)
  Filter: (ticket_no ~~ '0005433569575'::text)
Planning Time: 0.563 ms
Execution Time: 0.086 ms
```


***
## 2. Создание секционированной таблицы
***


Решил секционировать таблицу по диапазону. Как уже сказал, секционировать буду по ticket_no. Выбрал небольшие диапазоны:
по первым 7 знакам (0005430). Если бы тип данных для хранения ticket_no был числовым, было бы проще. 
Создал секционированную таблицу bookings.ticket_flights_partitioned:

```postgresql
CREATE TABLE bookings.ticket_flights_partitioned (
	ticket_no bpchar(13) NOT NULL,
	flight_id int4 NOT NULL,
	fare_conditions varchar(10) NOT NULL,
	amount numeric(10, 2) NOT NULL,
	CONSTRAINT ticket_flights_amount_check CHECK ((amount >= (0)::numeric)),
	CONSTRAINT ticket_flights_fare_conditions_check CHECK (((fare_conditions)::text = ANY (ARRAY[('Economy'::character varying)::text, ('Comfort'::character varying)::text, ('Business'::character varying)::text]))),
	CONSTRAINT ticket_flights_partitioned_pkey PRIMARY KEY (ticket_no, flight_id)
) partition by range (ticket_no);
```

Добавил ограничения по внешним ключам:

```postgresql
ALTER TABLE bookings.ticket_flights_partitioned ADD CONSTRAINT ticket_flights_flight_id_fkey FOREIGN KEY (flight_id) REFERENCES bookings.flights(flight_id);
ALTER TABLE bookings.ticket_flights_partitioned ADD CONSTRAINT ticket_flights_ticket_no_fkey FOREIGN KEY (ticket_no) REFERENCES bookings.tickets(ticket_no);
```

Создал секции для каждого диапазона значений ticket_no:

```postgresql
CREATE TABLE bookings.ticket_flights_partitioned_0005430 PARTITION OF bookings.ticket_flights_partitioned FOR VALUES FROM ('0005430000000') TO ('0005431000000');
CREATE TABLE bookings.ticket_flights_partitioned_0005431 PARTITION OF bookings.ticket_flights_partitioned FOR VALUES FROM ('0005431000000') TO ('0005432000000');
CREATE TABLE bookings.ticket_flights_partitioned_0005432 PARTITION OF bookings.ticket_flights_partitioned FOR VALUES FROM ('0005432000000') TO ('0005433000000');
CREATE TABLE bookings.ticket_flights_partitioned_0005433 PARTITION OF bookings.ticket_flights_partitioned FOR VALUES FROM ('0005433000000') TO ('0005434000000');
CREATE TABLE bookings.ticket_flights_partitioned_0005434 PARTITION OF bookings.ticket_flights_partitioned FOR VALUES FROM ('0005434000000') TO ('0005435000000');
CREATE TABLE bookings.ticket_flights_partitioned_0005435 PARTITION OF bookings.ticket_flights_partitioned FOR VALUES FROM ('0005435000000') TO ('0005436000000');
CREATE TABLE bookings.ticket_flights_partitioned_0005436 PARTITION OF bookings.ticket_flights_partitioned FOR VALUES FROM ('0005436000000') TO ('0005437000000');
CREATE TABLE bookings.ticket_flights_partitioned_0005437 PARTITION OF bookings.ticket_flights_partitioned FOR VALUES FROM ('0005437000000') TO ('0005438000000');
```


***
## 3. Миграция данных
***


Вставил данные в секционированную таблицу:

```postgresql
INSERT INTO bookings.ticket_flights_partitioned
(ticket_no, flight_id, fare_conditions, amount)
SELECT ticket_no, flight_id, fare_conditions, amount
FROM bookings.ticket_flights;
```


***
## 4. Оптимизация запросов
***


Провёл анализ запросов:

```postgresql
explain analyze
SELECT ticket_no, flight_id, fare_conditions, amount
FROM bookings.ticket_flights_partitioned
where ticket_no like '0005433569575';
```

```
Append  (cost=0.00..65.30 rows=16 width=52) (actual time=0.134..0.215 rows=4 loops=1)
  ->  Seq Scan on ticket_flights_partitioned_0005430 ticket_flights_partitioned_1  (cost=0.00..0.00 rows=1 width=114) (actual time=0.038..0.038 rows=0 loops=1)
        Filter: (ticket_no ~~ '0005433569575'::text)
  ->  Seq Scan on ticket_flights_partitioned_0005431 ticket_flights_partitioned_2  (cost=0.00..0.00 rows=1 width=114) (actual time=0.009..0.009 rows=0 loops=1)
        Filter: (ticket_no ~~ '0005433569575'::text)
  ->  Bitmap Heap Scan on ticket_flights_partitioned_0005432 ticket_flights_partitioned_3  (cost=4.45..16.23 rows=3 width=32) (actual time=0.050..0.050 rows=0 loops=1)
        Filter: (ticket_no ~~ '0005433569575'::text)
        ->  Bitmap Index Scan on ticket_flights_partitioned_0005432_pkey  (cost=0.00..4.45 rows=3 width=0) (actual time=0.040..0.040 rows=0 loops=1)
              Index Cond: (ticket_no = '0005433569575'::bpchar)
  ->  Index Scan using ticket_flights_partitioned_0005433_pkey on ticket_flights_partitioned_0005433 ticket_flights_partitioned_4  (cost=0.42..16.34 rows=3 width=32) (actual time=0.035..0.043 rows=4 loops=1)
        Index Cond: (ticket_no = '0005433569575'::bpchar)
        Filter: (ticket_no ~~ '0005433569575'::text)
  ->  Index Scan using ticket_flights_partitioned_0005434_pkey on ticket_flights_partitioned_0005434 ticket_flights_partitioned_5  (cost=0.42..16.39 rows=3 width=32) (actual time=0.022..0.022 rows=0 loops=1)
        Index Cond: (ticket_no = '0005433569575'::bpchar)
        Filter: (ticket_no ~~ '0005433569575'::text)
  ->  Bitmap Heap Scan on ticket_flights_partitioned_0005435 ticket_flights_partitioned_6  (cost=4.45..16.25 rows=3 width=32) (actual time=0.029..0.029 rows=0 loops=1)
        Filter: (ticket_no ~~ '0005433569575'::text)
        ->  Bitmap Index Scan on ticket_flights_partitioned_0005435_pkey  (cost=0.00..4.45 rows=3 width=0) (actual time=0.019..0.020 rows=0 loops=1)
              Index Cond: (ticket_no = '0005433569575'::bpchar)
  ->  Seq Scan on ticket_flights_partitioned_0005436 ticket_flights_partitioned_7  (cost=0.00..0.00 rows=1 width=114) (actual time=0.009..0.009 rows=0 loops=1)
        Filter: (ticket_no ~~ '0005433569575'::text)
  ->  Seq Scan on ticket_flights_partitioned_0005437 ticket_flights_partitioned_8  (cost=0.00..0.00 rows=1 width=114) (actual time=0.010..0.010 rows=0 loops=1)
        Filter: (ticket_no ~~ '0005433569575'::text)
Planning Time: 0.931 ms
Execution Time: 0.292 ms
```

```
Append  (cost=0.00..65.32 rows=16 width=52) (actual time=0.152..0.244 rows=4 loops=1)
  ->  Seq Scan on ticket_flights_partitioned_0005430 ticket_flights_partitioned_1  (cost=0.00..0.00 rows=1 width=114) (actual time=0.042..0.042 rows=0 loops=1)
        Filter: (ticket_no ~~ '0005433569575'::text)
  ->  Seq Scan on ticket_flights_partitioned_0005431 ticket_flights_partitioned_2  (cost=0.00..0.00 rows=1 width=114) (actual time=0.010..0.010 rows=0 loops=1)
        Filter: (ticket_no ~~ '0005433569575'::text)
  ->  Bitmap Heap Scan on ticket_flights_partitioned_0005432 ticket_flights_partitioned_3  (cost=4.45..16.24 rows=3 width=32) (actual time=0.053..0.054 rows=0 loops=1)
        Filter: (ticket_no ~~ '0005433569575'::text)
        ->  Bitmap Index Scan on ticket_flights_partitioned_0005432_pkey  (cost=0.00..4.45 rows=3 width=0) (actual time=0.042..0.043 rows=0 loops=1)
              Index Cond: (ticket_no = '0005433569575'::bpchar)
  ->  Index Scan using ticket_flights_partitioned_0005433_pkey on ticket_flights_partitioned_0005433 ticket_flights_partitioned_4  (cost=0.42..16.33 rows=3 width=32) (actual time=0.043..0.050 rows=4 loops=1)
        Index Cond: (ticket_no = '0005433569575'::bpchar)
        Filter: (ticket_no ~~ '0005433569575'::text)
  ->  Index Scan using ticket_flights_partitioned_0005434_pkey on ticket_flights_partitioned_0005434 ticket_flights_partitioned_5  (cost=0.42..16.40 rows=3 width=32) (actual time=0.031..0.031 rows=0 loops=1)
        Index Cond: (ticket_no = '0005433569575'::bpchar)
        Filter: (ticket_no ~~ '0005433569575'::text)
  ->  Bitmap Heap Scan on ticket_flights_partitioned_0005435 ticket_flights_partitioned_6  (cost=4.45..16.26 rows=3 width=32) (actual time=0.028..0.028 rows=0 loops=1)
        Filter: (ticket_no ~~ '0005433569575'::text)
        ->  Bitmap Index Scan on ticket_flights_partitioned_0005435_pkey  (cost=0.00..4.45 rows=3 width=0) (actual time=0.019..0.019 rows=0 loops=1)
              Index Cond: (ticket_no = '0005433569575'::bpchar)
  ->  Seq Scan on ticket_flights_partitioned_0005436 ticket_flights_partitioned_7  (cost=0.00..0.00 rows=1 width=114) (actual time=0.012..0.012 rows=0 loops=1)
        Filter: (ticket_no ~~ '0005433569575'::text)
  ->  Seq Scan on ticket_flights_partitioned_0005437 ticket_flights_partitioned_8  (cost=0.00..0.00 rows=1 width=114) (actual time=0.010..0.011 rows=0 loops=1)
        Filter: (ticket_no ~~ '0005433569575'::text)
Planning Time: 2.741 ms
Execution Time: 0.364 ms
```

И анализ запроса к одной секции:

```postgresql
explain analyze
SELECT ticket_no, flight_id, fare_conditions, amount
FROM bookings.ticket_flights_partitioned_0005433
where ticket_no like '0005433569575';
```

```
Index Scan using ticket_flights_partitioned_0005433_pkey on ticket_flights_partitioned_0005433  (cost=0.42..16.34 rows=3 width=32) (actual time=0.054..0.061 rows=4 loops=1)
  Index Cond: (ticket_no = '0005433569575'::bpchar)
  Filter: (ticket_no ~~ '0005433569575'::text)
Planning Time: 0.150 ms
Execution Time: 0.086 ms
```

В результате анализа видно, что секционированная таблица немного уступает по производительности обычной таблице.
Но со временем проще будет оптимизировать хранение этих данных именно в секционированной таблице, путём отключения секции и т.п. 


***
## 5. Тестирование решения
***


Проверил операции вставки, обновления и удаления. Всё работает:

```postgresql
delete from bookings.ticket_flights_partitioned
where amount < 5000;
```

```
Updated Rows	73274
Execute time	1s
Start time	Thu Mar 27 15:53:44 MSK 2025
Finish time	Thu Mar 27 15:53:46 MSK 2025
```

```postgresql
update bookings.ticket_flights_partitioned
set amount=amount+200
where amount = 9800;
```

```
Updated Rows	28774
Execute time	3s
Start time	Thu Mar 27 15:52:58 MSK 2025
Finish time	Thu Mar 27 15:53:01 MSK 2025
```

```postgresql
insert into bookings.ticket_flights_partitioned(ticket_no, flight_id, fare_conditions, amount)
SELECT distinct '0005436' || substring(ticket_no,8,6) as ticket_no, flight_id, fare_conditions, amount
FROM bookings.ticket_flights
where ticket_no like '0005432______';
```

```
Updated Rows	502272
Execute time	54s
Start time	Thu Mar 27 16:00:51 MSK 2025
Finish time	Thu Mar 27 16:01:45 MSK 2025
```

```postgresql
insert into bookings.ticket_flights_partitioned(ticket_no, flight_id, fare_conditions, amount)
SELECT distinct '0005437' || substring(ticket_no,8,6) as ticket_no, flight_id, fare_conditions, amount
FROM bookings.ticket_flights
where ticket_no like '0005432______';
```

```
Updated Rows	502272
Execute time	58s
Start time	Thu Mar 27 16:02:49 MSK 2025
Finish time	Thu Mar 27 16:03:47 MSK 2025
```

Однако, при вставке новых значений в секционированную таблицу, которые вышли за границы созданного диапазона, получил ошибку:

```postgresql
insert into bookings.ticket_flights_partitioned(ticket_no, flight_id, fare_conditions, amount)
SELECT distinct '0005438' || substring(ticket_no,8,6) as ticket_no, flight_id, fare_conditions, amount
FROM bookings.ticket_flights
where ticket_no like '0005432______';
```

```
SQL Error [23514]: ОШИБКА: для строки не найдена секция в отношении "ticket_flights_partitioned"
  Подробности: Ключ секционирования для неподходящей строки содержит (ticket_no) = (0005438522123).
```
  
Чтобы этого избежать, необходимо было своевременно создать новую секцию для последующего диапазона значений ticket_no.

