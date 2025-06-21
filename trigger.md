
# Триггеры, поддержка заполнения витрин
## Создать триггер для поддержки витрины в актуальном состоянии.
### Создал базу

```sql
CREATE DATABASE  otus_new;
```

```sql
postgres=# CREATE DATABASE  otus_new;
CREATE DATABASE
postgres=# \c otus_new
You are now connected to database "otus_new" as user "postgres".
```


### Создадим таблицы с наполнением:

```sql
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);

INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);

SELECT * from goods;
```

```
 goods_id |        good_name         |  good_price
----------+--------------------------+--------------
        1 | Спички хозайственные     |         0.50
        2 | Автомобиль Ferrari FXX K | 185000000.01
(2 rows)
```

```sql
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

SELECT * from sales;
```

```
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        1 |       1 | 2025-06-20 21:12:07.103726+03 |        10
        2 |       1 | 2025-06-20 21:12:07.103726+03 |         1
        3 |       1 | 2025-06-20 21:12:07.103726+03 |       120
        4 |       2 | 2025-06-20 21:12:07.103726+03 |         1
(4 rows)
```

### Выводим отчет:

```sql
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
```

```
        good_name         |     sum
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```

### Решил денормализовать БД, создав таблицу good_sum_mart:

```sql
CREATE TABLE good_sum_mart 
(
 good_name  varchar(63) PRIMARY KEY,
 sum_sale   numeric(16, 2) NOT NULL,
 CONSTRAINT good_name_unique UNIQUE (good_name)
 );

\dt
```

```
otus_new=# \dt
                 List of relations
     Schema      |     Name      | Type  |  Owner
-----------------+---------------+-------+----------
 pract_functions | good_sum_mart | table | postgres
 pract_functions | goods         | table | postgres
 pract_functions | sales         | table | postgres
(3 rows)
```

### Заполняем таблицу-витрину данными

```sql
INSERT INTO good_sum_mart SELECT G.good_name, sum(G.good_price * S.sales_qty) AS sum_sale
  FROM goods G
  INNER JOIN sales S ON S.good_id = G.goods_id
  GROUP BY G.good_name;

SELECT * from good_sum_mart;
```

```sql
otus_new=# SELECT * from good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        65.50
(2 rows)
```

### Создаю триггерные функции:

```sql
CREATE or replace function ft_insert_sales()
RETURNS trigger
AS
$TRIG_FUNC$
DECLARE
  g_name varchar(63);
  g_price numeric(12,2);
BEGIN
   SELECT G.good_name, G.good_price*NEW.sales_qty INTO g_name, g_price
   FROM goods G
   where G.goods_id = NEW.good_id;
   IF EXISTS(select from good_sum_mart T where T.good_name = g_name)
   THEN UPDATE good_sum_mart T SET sum_sale = sum_sale + g_price where T.good_name = g_name;
   ELSE INSERT INTO good_sum_mart (good_name, sum_sale) values(g_name, g_price);
   END IF;
   RETURN NEW;
END;
$TRIG_FUNC$
LANGUAGE plpgsql;
```

```sql
CREATE or replace function ft_delete_sales()
RETURNS trigger
AS
$TRIG_FUNC$
DECLARE
  g_name varchar(63);
  g_price numeric(12,2);
BEGIN
    SELECT G.good_name, G.good_price*OLD.sales_qty INTO g_name, g_price
    FROM goods G where G.goods_id = OLD.good_id;
       IF EXISTS(select from good_sum_mart T where T.good_name = g_name)
    THEN 
       UPDATE good_sum_mart T SET sum_sale = sum_sale - g_price where T.good_name = g_name;
       DELETE FROM good_sum_mart T where T.good_name = g_name and (sum_sale < 0 or sum_sale = 0);
    END IF;
    RETURN NEW;
END;
$TRIG_FUNC$
LANGUAGE plpgsql;
```

```sql
CREATE or replace function ft_update_sales()
RETURNS trigger
AS
$TRIG_FUNC$
DECLARE
   g_name_old varchar(63);
   g_price_old numeric(12,2);
   g_name_new varchar(63);
   g_price_new numeric(12,2);
BEGIN
    SELECT G.good_name, G.good_price*OLD.sales_qty INTO g_name_old, g_price_old
    FROM goods G where G.goods_id = OLD.good_id;
    SELECT G.good_name, G.good_price*NEW.sales_qty INTO g_name_new, g_price_new
    FROM goods G where G.goods_id = NEW.good_id;
    IF EXISTS(select from good_sum_mart T where T.good_name = g_name_new)
      THEN UPDATE good_sum_mart T SET sum_sale = sum_sale + g_price_new where T.good_name = g_name_new;
    ELSE INSERT INTO good_sum_mart (good_name, sum_sale) values(g_name_new, g_price_new);
    END IF;
      IF EXISTS(select from good_sum_mart T where T.good_name = g_name_old)
    THEN 
      UPDATE good_sum_mart T SET sum_sale = sum_sale - g_price_old where T.good_name = g_name_old;
      DELETE FROM good_sum_mart T where T.good_name = g_name_old and (sum_sale < 0 or sum_sale = 0);
    END IF;
    RETURN NEW;
END;
$TRIG_FUNC$
LANGUAGE plpgsql;
```

### Создаю триггеры:

```sql
CREATE TRIGGER tr_insert_sales
AFTER INSERT
ON sales
FOR EACH ROW
EXECUTE PROCEDURE ft_insert_sales();

CREATE TRIGGER tr_delete_sales
AFTER DELETE
ON sales
FOR EACH ROW
EXECUTE PROCEDURE ft_delete_sales();

CREATE TRIGGER tr_update_sales
AFTER UPDATE
ON sales
FOR EACH ROW
EXECUTE PROCEDURE ft_update_sales();
```

### Тестирую
#### Insert

```sql
INSERT INTO sales (good_id, sales_qty) VALUES (1, 10);
SELECT * from good_sum_mart;
```

```sql
otus_new=# SELECT * from good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |        70.50
(2 rows)
```

#### Update
```
UPDATE sales SET sales_qty = 125000000 where sales_id = 1;
SELECT * from good_sum_mart;
```

```sql
otus_new=# SELECT * from good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |  62500065.50
(2 rows)
otus_new=# select * from sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        2 |       1 | 2025-06-20 21:12:07.103726+03 |         1
        3 |       1 | 2025-06-20 21:12:07.103726+03 |       120
        4 |       2 | 2025-06-20 21:12:07.103726+03 |         1
        5 |       1 | 2025-06-20 21:33:08.116473+03 |        10
        1 |       1 | 2025-06-20 21:12:07.103726+03 | 125000000
(5 rows)
```

#### Delete

```sql
DELETE from sales where sales_id = 2;
SELECT * from good_sum_mart;
```

```sql
otus_new=# SELECT * from good_sum_mart;
        good_name         |   sum_sale
--------------------------+--------------
 Автомобиль Ferrari FXX K | 185000000.01
 Спички хозайственные     |  62500065.00
(2 rows)
otus_new=# select * from sales;
 sales_id | good_id |          sales_time           | sales_qty
----------+---------+-------------------------------+-----------
        3 |       1 | 2025-06-20 21:12:07.103726+03 |       120
        4 |       2 | 2025-06-20 21:12:07.103726+03 |         1
        5 |       1 | 2025-06-20 21:33:08.116473+03 |        10
        1 |       1 | 2025-06-20 21:12:07.103726+03 | 125000000
(4 rows)
```

