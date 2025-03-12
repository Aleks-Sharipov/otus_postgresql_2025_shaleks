Работаю с локальным кластером на старом ноуте с 3Гб ОЗУ, 900 Гб HDD и 2х ядерным CPU Intel Pentium B960 2.2ГГц, Deb11:)

1.
***
## shared_buffers
***
Выставил значение 512Мб.

Запустил загрузку тестовых данных:
```sql
pgbench -h 192.168.31.56 -p 5433 -U postgres -i -s 150 otus
```

Память съел почти всю после выполнения заливки тестовых данных:
до
```sql
               total        used        free      shared  buff/cache   available
Mem:            3186         487        1784         160         915        2242
Swap:              0           0           0
```
после
```sql
               total        used        free      shared  buff/cache   available
Mem:            3186         556         246         227        2383        2076
Swap:              0           0           0
```

Весь процесс занял 4 мин +\-:
```sql
done in 244.07 s (drop tables 0.00 s, create tables 0.62 s, client-side generate 129.19 s, vacuum 84.84 s, primary keys 29.42 s).
```
Запустил тестирование:
```sql
pgbench -p 5433 -c 50 -j 2 -P 10 -T 60 otus
```

Получил результат, tps ~ 114:
```sql
scaling factor: 150
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 6978
latency average = 429.282 ms
latency stddev = 280.191 ms
tps = 114.397761 (including connections establishing)
tps = 114.404917 (excluding connections establishing)
```

Поменял shared_buffers на 1024Мб. Перезапустился. Повтор тестирования. Результат по tps стал лучше ~ 102:
```sql
number of transactions actually processed: 6230
latency average = 479.861 ms
latency stddev = 298.598 ms
tps = 102.478047 (including connections establishing)
tps = 102.484637 (excluding connections establishing)
```

Пробую shared_buffers на 2048 Гб. Перезапустился. Повтор тестирования. Результат по tps стал лучше ~ 82:
```sql
number of transactions actually processed: 4940
latency average = 604.706 ms
latency stddev = 400.493 ms
tps = 82.030043 (including connections establishing)
tps = 82.035737 (excluding connections establishing)
```
Но свободной памяти при этом оставалось мало при тестировании:
```sql
               total        used        free      shared  buff/cache   available
Mem:            3186         617         166         354        2402        1909
Swap:              0           0           0
```
Т.о. при увеличении shared_buffers улучшается производительность tps.

2.
***
## max_connections
***

Вернул назад значение shared_buffers = 512MB, выставил max_connections = 50. Запустил тест с теми же параметрами (-c 50)
Результат показал хорошие tps~ 73:
```sql
scaling factor: 150
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 4507
latency average = 666.202 ms
latency stddev = 425.585 ms
tps = 73.039226 (including connections establishing)
tps = 73.043983 (excluding connections establishing)
```
Выставил max_connections = 250. Запустил тест с параметрами (-c 250)
Результат показал tps~ 108:
```sql
scaling factor: 150
query mode: simple
number of clients: 250
number of threads: 2
duration: 60 s
number of transactions actually processed: 7078
latency average = 2145.944 ms
latency stddev = 1759.268 ms
tps = 108.727045 (including connections establishing)
tps = 108.733389 (excluding connections establishing)
```

Т.о. при увеличении количества подключенных пользователей падает производительность.

3.
***
## effective_cache_size
***
Вернул назад значение max_connections = 50. Включил effective_cache_size = 2GB:
```sql
scaling factor: 150
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 4299
latency average = 696.025 ms
latency stddev = 417.894 ms
tps = 70.942418 (including connections establishing)
tps = 70.947445 (excluding connections establishing)
```
Прогресс не особо заметен. При выкл. значении такие же показатели tps

4. 
***
## work_mem
***
Включил параметр work_mem = 16MB. Запустил тест:
```sql
scaling factor: 150
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 4661
latency average = 642.345 ms
latency stddev = 408.471 ms
tps = 76.741487 (including connections establishing)
tps = 76.746749 (excluding connections establishing)
```
Показатели tps стали хуже. Увеличиваю work_mem = 32MB
```sql
number of transactions actually processed: 5634
latency average = 536.582 ms
latency stddev = 373.273 ms
tps = 90.474527 (including connections establishing)
tps = 90.480277 (excluding connections establishing)
```
Ещё хуже :( 
Уменьшил work_mem = 512kB
Показатели вернулись в прежние диапазоны:
```sql
number of transactions actually processed: 4547
latency average = 660.788 ms
latency stddev = 443.070 ms
tps = 73.791488 (including connections establishing)
tps = 73.796293 (excluding connections establishing)
```
Уменьшил work_mem = 128kB:
```sql
number of transactions actually processed: 4709
latency average = 660.325 ms
latency stddev = 582.422 ms
tps = 73.857386 (including connections establishing)
tps = 73.862033 (excluding connections establishing)

```
Не особо изменилось.

5.
***
## maintenance_work_mem
***
Тестировать не стал, .т.к. не делал VACUUM, CREATE INDEX, CREATE FOREIGN KEY.

6.
***
## wal_buffers
***

Выключил параметры work_mem и effective_cache_size. Включил параметр wal_buffers = 16MB. Тестирую
```sql
scaling factor: 150
query mode: simple
number of clients: 50
number of threads: 2
duration: 60 s
number of transactions actually processed: 4223
latency average = 706.957 ms
latency stddev = 456.125 ms
tps = 69.855002 (including connections establishing)
tps = 69.859844 (excluding connections establishing)
```

Уменьшил до wal_buffers = 8MB
```sql
number of transactions actually processed: 5751
latency average = 519.873 ms
latency stddev = 536.095 ms
tps = 92.524913 (including connections establishing)
tps = 92.530781 (excluding connections establishing)
```
Показатели ухудшились. Увеличил до wal_buffers = 32MB
```sql
number of transactions actually processed: 4743
latency average = 632.450 ms
latency stddev = 450.108 ms
tps = 76.548479 (including connections establishing)
tps = 76.553779 (excluding connections establishing)
```
Показатели далеки от первоначальных, т.о. лучший вариант был при wal_buffers = 16MB, что соответвует моим 1/32 от shared_buffers

7.
***
## max_wal_size
***
Тестирование параметров max_wal_size и checkpoint_timeout не проводил. Не стал тестить сбой БД

