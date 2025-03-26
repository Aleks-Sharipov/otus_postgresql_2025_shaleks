

***
## 1. Тестируем план выполнения запроса для таблицы без индексов
***

Загрузили данные в таблицу ~ 500.000 строк 
Выполнили для неё EXPLAIN

```postgresql
EXPLAIN
SELECT * FROM products
    WHERE product_id = 1;
```
```
Gather  (cost=1000.00..6789.67 rows=5 width=14)
  Workers Planned: 2
  ->  Parallel Seq Scan on products  (cost=0.00..5789.17 rows=2 width=14)
        Filter: (product_id = 1)
```

```postgresql
EXPLAIN
SELECT * FROM products
    WHERE product_id < 100;
```

```
Gather  (cost=1000.00..6837.77 rows=486 width=14)
  Workers Planned: 2
  ->  Parallel Seq Scan on products  (cost=0.00..5789.17 rows=202 width=14)
        Filter: (product_id < 100)
```


***
## 2. Создать индекс к какой-либо из таблиц 
***

Создадим обычный индекс на таблице:

```postgresql
CREATE INDEX
    idx_products_product_id
    ON products(product_id);
```

Выполнили для неё EXPLAIN:

```postgresql
EXPLAIN
SELECT * FROM products
    WHERE product_id = 1;
```

```
Bitmap Heap Scan on products  (cost=4.46..23.93 rows=5 width=14)
  Recheck Cond: (product_id = 1)
  ->  Bitmap Index Scan on idx_products_product_id  (cost=0.00..4.46 rows=5 width=0)
        Index Cond: (product_id = 1)
```

```postgresql
EXPLAIN
SELECT * FROM products
    WHERE product_id < 100;
```

```
Bitmap Heap Scan on products  (cost=8.41..1369.06 rows=515 width=14)
  Recheck Cond: (product_id < 100)
  ->  Bitmap Index Scan on idx_products_product_id  (cost=0.00..8.29 rows=515 width=0)
        Index Cond: (product_id < 100)
```

В результате плана выполнения запросов видим, что индекс работает, используется Bitmap Index Scan.


***
## 3. Реализовать индекс для полнотекстового поиска 
***

Создали таблицу с текстовым полем и загрузили в неё данные:

```postgresql
CREATE TABLE documents (
    title    varchar(64),
    metadata jsonb,
    contents text
);
```

Создали GIN индекс для полнотекстового поиска:

```postgresql
CREATE INDEX
    idx_documents_contents
    ON documents
    USING GIN(to_tsvector('english', contents));
```

Выполнили для неё EXPLAIN:
```sql
EXPLAIN
SELECT * FROM documents
    WHERE to_tsvector('english', contents) @@ 'node';
```

```
Seq Scan on documents  (cost=0.00..6.51 rows=1 width=210)
  Filter: (to_tsvector('english'::regconfig, contents) @@ '''node'''::tsquery)
```

В результате плана выполнения запроса видим, что используется последовательное сканирование таблицы. Индекс не используется из-за малого числа строк в таблице. 
Выключили принудительно Seq Scan:

```postgresql
SET enable_seqscan = OFF;
```

Запускаем EXPLAIN:

```
Bitmap Heap Scan on documents  (cost=8.54..12.80 rows=1 width=210)
  Recheck Cond: (to_tsvector('english'::regconfig, contents) @@ '''node'''::tsquery)
  ->  Bitmap Index Scan on idx_documents_contents  (cost=0.00..8.54 rows=1 width=0)
        Index Cond: (to_tsvector('english'::regconfig, contents) @@ '''node'''::tsquery)
```

Всё, индекс работает, используется Bitmap Index Scan.


***
## 4. Реализовать индекс на часть таблицы или индекс на поле с функцией 
***

Создадим таблицу и наполним её записями ~ 1 млн строк

```postgresql
CREATE TABLE cities (
    code CHAR(13) PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);
```

Запускаем EXPLAIN по неиднексированной таблице на поиск по части поля code:

```postgresql
explain
SELECT * FROM cities
    WHERE SUBSTRING(code, 1, 3) = '250';
```

```
Gather  (cost=1000.00..14197.67 rows=5000 width=16)
  Workers Planned: 2
  ->  Parallel Seq Scan on cities  (cost=0.00..12697.67 rows=2083 width=16)
        Filter: ("substring"((code)::text, 1, 3) = '250'::text)
```

Как видим, индекс не используется. Идёт Parallel Seq Scan

Проиндексируем часть столбца code:

```postgresql
CREATE INDEX
    func_idx_cities_region
    ON cities ((SUBSTRING(code, 1, 3)));
```

Выполнили EXPLAIN:

```
Bitmap Heap Scan on cities  (cost=59.17..5665.65 rows=5000 width=16)
  Recheck Cond: ("substring"((code)::text, 1, 3) = '250'::text)
  ->  Bitmap Index Scan on func_idx_cities_region  (cost=0.00..57.92 rows=5000 width=0)
        Index Cond: ("substring"((code)::text, 1, 3) = '250'::text)
```

Всё, индекс работает, используется Bitmap Heap Scan.
Попробуем выключим Bitmap Index Scan и запустим EXPLAIN

```postgresql
SET enable_bitmapscan = off;
```

Запрос в итоге также использует индекс, но подругому через полное его сканирование:

```
Index Scan using func_idx_cities_region on cities  (cost=0.42..13783.92 rows=5000 width=16)
  Index Cond: ("substring"((code)::text, 1, 3) = '250'::text)
```

А если выключим и Index Scan:

```postgresql
SET enable_indexscan = off;
```

Тогда получим тот же план выполнения, что и без использования индекса:

```
Gather  (cost=1000.00..14197.67 rows=5000 width=16)
  Workers Planned: 2
  ->  Parallel Seq Scan on cities  (cost=0.00..12697.67 rows=2083 width=16)
        Filter: ("substring"((code)::text, 1, 3) = '250'::text)
```

***
## 5. Создать индекс на несколько полей 
***


Удалил индекс в таблице products:

```postgresql
drop index idx_products_product_id ;
```

Запустил EXPLAIN:

```
Gather  (cost=1000.00..7327.30 rows=173 width=14)
  Workers Planned: 2
  ->  Parallel Seq Scan on products  (cost=0.00..6310.00 rows=72 width=14)
        Filter: ((product_id <= 3344) AND (brand = 'r'::bpchar))
```

Индекс не используется, т.к. его нет. Оптимизатор выполяет параллельное сканирование таблицы Parallel Seq Scan.
Создадим индекс по двум полям brand, product_id:

```postgresql
CREATE INDEX
    idx_products_brand_product_id
    ON products(brand, product_id);
```

Запустил EXPLAIN:

```
Bitmap Heap Scan on products  (cost=6.20..568.00 rows=173 width=14)
  Recheck Cond: ((brand = 'r'::bpchar) AND (product_id <= 3344))
  ->  Bitmap Index Scan on idx_products_brand_product_id  (cost=0.00..6.15 rows=173 width=0)
        Index Cond: ((brand = 'r'::bpchar) AND (product_id <= 3344))
```

Всё, индекс работает, используется Bitmap Index Scan.


