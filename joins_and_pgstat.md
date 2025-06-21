## Сбор и использование статистики

## Развернем демо базу для работы

```
wget https://edu.postgrespro.ru/demo_small.zip && unzip demo_small.zip && PGPASSWORD=pgpass psql -U postgres -h 192.168.31.56 -d postgres -f ./demo_small.sql -c 'alter database demo set search_path to bookings';
```

# В демо базе используются таблицы bookings.aircrafts, bookings.airports, bookings.flights. Их структура ниже

```sql
CREATE TABLE bookings.aircrafts (
	aircraft_code bpchar(3) NOT NULL,
	model text NOT NULL,
	"range" int4 NOT NULL,
	CONSTRAINT aircrafts_pkey PRIMARY KEY (aircraft_code),
	CONSTRAINT aircrafts_range_check CHECK ((range > 0))
);

CREATE TABLE bookings.airports (
    airport_code bpchar(3) NOT NULL,
    airport_name text NOT NULL,
    city text NOT NULL,
    longitude float8 NOT NULL,
    latitude float8 NOT NULL,
    timezone text NOT NULL,
    CONSTRAINT airports_pkey PRIMARY KEY (airport_code)
);

CREATE TABLE bookings.flights (
    flight_id serial4 NOT NULL,
    flight_no bpchar(6) NOT NULL,
    scheduled_departure timestamptz NOT NULL,
    scheduled_arrival timestamptz NOT NULL,
    departure_airport bpchar(3) NOT NULL,
    arrival_airport bpchar(3) NOT NULL,
    status varchar(20) NOT NULL,
    aircraft_code bpchar(3) NOT NULL,
    actual_departure timestamptz NULL,
    actual_arrival timestamptz NULL,
    CONSTRAINT flights_check CHECK ((scheduled_arrival > scheduled_departure)),
    CONSTRAINT flights_check1 CHECK (((actual_arrival IS NULL) OR ((actual_departure IS NOT NULL) AND (actual_arrival IS NOT NULL) AND (actual_arrival > actual_departure)))),
    CONSTRAINT flights_flight_no_scheduled_departure_key UNIQUE (flight_no, scheduled_departure),
    CONSTRAINT flights_pkey PRIMARY KEY (flight_id),
    CONSTRAINT flights_status_check CHECK (((status)::text = ANY (ARRAY[('On Time'::character varying)::text, ('Delayed'::character varying)::text, ('Departed'::character varying)::text, ('Arrived'::character varying)::text, ('Scheduled'::character varying)::text, ('Cancelled'::character varying)::text])))
);
```

# Реализовать прямое соединение двух или более таблиц

```sql
select f.flight_no,f.departure_airport,f.arrival_airport,dep.airport_name,f.actual_arrival
from flights f
inner join airports dep on f.departure_airport = dep.airport_code
limit 10;
```
count | 33121

# Реализовать левостороннее (или правостороннее) соединение двух или более таблиц

```sql
select f.departure_airport,f.arrival_airport,dep.airport_name,f.actual_arrival
from flights f
left outer join airports dep on f.departure_airport = dep.airport_code
limit 10;
```
count | 33121

# Реализовать кросс соединение двух или более таблиц

```sql
select distinct f.departure_airport,f.arrival_airport,dep.airport_name
from flights f
cross join airports dep;
```
count | 3444584

# Реализовать полное соединение двух или более таблиц

```sql
select f.*,dep.*
from flights f
full join airports dep on f.departure_airport = dep.airport_code
limit 10;
```
count | 33121

# Реализовать запрос, в котором будут использованы разные типы соединений

```sql
select f.*,dep.*,air.*
from flights f
left outer join airports dep on f.departure_airport = dep.airport_code
full join aircrafts air on f.aircraft_code = air.aircraft_code
limit 10;
```
count | 33122


По количеству строк при join, left outer join и full join разницы нет - все запросы выводят по 33121 строк, т.е. равное количеству записей в таблице flights.
При cross join происходит полное соединение всех записей (каждая с каждой) и соответственно выдает 3444584 строк (33121 * 104) (count(flight) * count(airports)).


## Работа со статистикой

# серверные процесс, находящиеся в состоянии idle in transaction

```sql
SELECT pid, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
AND xact_start < NOW() - INTERVAL '2 seconds';
```

# использование временных файлов базой данных

```sql
SELECT temp_files, temp_bytes FROM pg_stat_database
WHERE datname = current_database();
```
