# Задание 5
## Нагрузочное тестирование и тюнинг PostgreSQL
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 15.8

Для начала создадим базу для бенчмаркинга:
```
# psql -h localhost -p 5432 -U postgres 
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# CREATE DATABASE benchmark;
CREATE DATABASE
```
Инициализируем ее для *pgbench*:
```
# pgbench -i -U postgres benchmark
```
Запустим тест на базовом конфиге, чтобы было с чем сравнивать:
```
pgbench -c 50 -j 2 -P 60 -T 600 -U postgres benchmark
pgbench (15.8 (Debian 15.8-1.pgdg120+1))
starting vacuum...end.
progress: 60.0 s, 497.3 tps, lat 100.020 ms stddev 171.629, 0 failed
progress: 120.0 s, 186.5 tps, lat 266.725 ms stddev 305.752, 0 failed
progress: 180.0 s, 196.4 tps, lat 255.163 ms stddev 280.590, 0 failed
progress: 240.0 s, 198.7 tps, lat 251.284 ms stddev 289.711, 0 failed
progress: 300.0 s, 197.1 tps, lat 254.871 ms stddev 305.648, 0 failed
progress: 360.0 s, 198.6 tps, lat 252.130 ms stddev 276.843, 0 failed
progress: 420.0 s, 201.2 tps, lat 248.257 ms stddev 305.560, 0 failed
progress: 480.0 s, 200.7 tps, lat 249.380 ms stddev 287.567, 0 failed
progress: 540.0 s, 202.4 tps, lat 246.335 ms stddev 286.637, 0 failed
progress: 600.0 s, 205.1 tps, lat 244.274 ms stddev 330.586, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 137076
number of failed transactions: 0 (0.000%)
latency average = 218.874 ms
latency stddev = 281.761 ms
initial connection time = 46.000 ms
tps = 228.390137 (without initial connection time)
```
В первую очередь, изменяем *fsync* на off в postgresql.conf, жертвуя надежностью (дописываем в конец файла):
```
#fsync = off
```
Повторный запуск:
```
psql -h localhost -p 5432 -U postgres
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# show fsync;
 fsync 
-------
 off
(1 row)

postgres=# \q
# pgbench -c 50 -j 2 -P 60 -T 600 -U postgres benchmark
pgbench (15.8 (Debian 15.8-1.pgdg120+1))
starting vacuum...end.
progress: 60.0 s, 3674.0 tps, lat 13.590 ms stddev 16.975, 0 failed
progress: 120.0 s, 3781.4 tps, lat 13.218 ms stddev 16.243, 0 failed
progress: 180.0 s, 3785.4 tps, lat 13.204 ms stddev 16.478, 0 failed
progress: 240.0 s, 3736.3 tps, lat 13.376 ms stddev 16.260, 0 failed
progress: 300.0 s, 3727.0 tps, lat 13.409 ms stddev 16.599, 0 failed
progress: 360.0 s, 3678.6 tps, lat 13.587 ms stddev 16.660, 0 failed
progress: 420.0 s, 3564.9 tps, lat 14.019 ms stddev 17.306, 0 failed
progress: 480.0 s, 3672.2 tps, lat 13.609 ms stddev 16.823, 0 failed
progress: 540.0 s, 3737.0 tps, lat 13.376 ms stddev 16.330, 0 failed
progress: 600.0 s, 3706.5 tps, lat 13.484 ms stddev 16.524, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2223859
number of failed transactions: 0 (0.000%)
latency average = 13.484 ms
latency stddev = 16.620 ms
initial connection time = 43.201 ms
tps = 3706.448438 (without initial connection time)
```
> [!IMPORTANT]
> Условный score вырос более чем в 10 раз, но конечно *fsync* отключать не рекомендуется, см. комментарий в стартовом конфиге postgresql. Еще почему-то если раньше короткий тест был сильно быстрее длинного, то теперь показатели сравнялись.

Далее, воспользуемся pgtune (недоступно в РФ). Аппаратная конфигурация уже задана виртуалкой, на нее повлиять нельзя, но попробуем поиграться с типом приложения. Для начала выбрали **Mixed type**, в итоге для моей конфигурации железа получилось:
```
max_connections = 50
shared_buffers = 1536MB
effective_cache_size = 4608MB
maintenance_work_mem = 384MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 5242kB
huge_pages = off
min_wal_size = 1GB
max_wal_size = 4GB
max_worker_processes = 6
max_parallel_workers_per_gather = 3
max_parallel_workers = 6
max_parallel_maintenance_workers = 3
```
Повторный запуск:
```
# pgbench -c 50 -j 2 -P 60 -T 600 -U postgres benchmark
pgbench (15.8 (Debian 15.8-1.pgdg120+1))
starting vacuum...end.
progress: 60.0 s, 3678.9 tps, lat 13.572 ms stddev 16.778, 0 failed
progress: 120.0 s, 3661.4 tps, lat 13.650 ms stddev 17.027, 0 failed
progress: 180.0 s, 3622.3 tps, lat 13.796 ms stddev 17.268, 0 failed
progress: 240.0 s, 3606.8 tps, lat 13.858 ms stddev 17.257, 0 failed
progress: 300.0 s, 3642.6 tps, lat 13.722 ms stddev 17.131, 0 failed
progress: 360.0 s, 3685.1 tps, lat 13.562 ms stddev 16.684, 0 failed
progress: 420.0 s, 3627.2 tps, lat 13.780 ms stddev 17.136, 0 failed
progress: 480.0 s, 3690.3 tps, lat 13.544 ms stddev 16.632, 0 failed
progress: 540.0 s, 3717.9 tps, lat 13.441 ms stddev 16.649, 0 failed
progress: 600.0 s, 3710.1 tps, lat 13.473 ms stddev 16.685, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2198611
number of failed transactions: 0 (0.000%)
latency average = 13.639 ms
latency stddev = 16.925 ms
initial connection time = 42.201 ms
tps = 3664.345004 (without initial connection time)
```
> [!IMPORTANT]
> Изменения в рамках погрешности, как мне кажется, хотя видно что результаты стабильно чуть-чуть похуже.

Tеперь **Desktop application**:
```
max_connections = 50
shared_buffers = 384MB
effective_cache_size = 1536MB
maintenance_work_mem = 384MB
checkpoint_completion_target = 0.9
wal_buffers = 11796kB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 2184kB
huge_pages = off
min_wal_size = 100MB
max_wal_size = 2GB
max_worker_processes = 6
max_parallel_workers_per_gather = 3
max_parallel_workers = 6
max_parallel_maintenance_workers = 3
wal_level = minimal
max_wal_senders = 0
```
Повторный запуск:
```
# pgbench -c 50 -j 2 -P 60 -T 600 -U postgres benchmark
pgbench (15.8 (Debian 15.8-1.pgdg120+1))
starting vacuum...end.
progress: 60.0 s, 3607.0 tps, lat 13.841 ms stddev 17.204, 0 failed
progress: 120.0 s, 3732.9 tps, lat 13.390 ms stddev 16.640, 0 failed
progress: 180.0 s, 3796.3 tps, lat 13.166 ms stddev 16.424, 0 failed
progress: 240.0 s, 3774.5 tps, lat 13.242 ms stddev 16.338, 0 failed
progress: 300.0 s, 3706.9 tps, lat 13.482 ms stddev 16.805, 0 failed
progress: 360.0 s, 3761.6 tps, lat 13.286 ms stddev 16.420, 0 failed
progress: 420.0 s, 3767.2 tps, lat 13.268 ms stddev 16.225, 0 failed
progress: 480.0 s, 3768.0 tps, lat 13.264 ms stddev 16.256, 0 failed
progress: 540.0 s, 3763.6 tps, lat 13.280 ms stddev 16.288, 0 failed
progress: 600.0 s, 3760.0 tps, lat 13.292 ms stddev 16.473, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2246337
number of failed transactions: 0 (0.000%)
latency average = 13.349 ms
latency stddev = 16.508 ms
initial connection time = 42.585 ms
tps = 3743.868468 (without initial connection time)
```
> [!IMPORTANT]
> Опять таки, разница в процентах, но формально чуть побыстрее стало.

Tеперь **Data warehouse**:
```
max_connections = 50
shared_buffers = 1536MB
effective_cache_size = 4608MB
maintenance_work_mem = 768MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 5242kB
huge_pages = off
min_wal_size = 4GB
max_wal_size = 16GB
max_worker_processes = 6
max_parallel_workers_per_gather = 3
max_parallel_workers = 6
max_parallel_maintenance_workers = 3
```
Повторный запуск:
```
# pgbench -c 50 -j 2 -P 60 -T 600 -U postgres benchmark
pgbench (15.8 (Debian 15.8-1.pgdg120+1))
starting vacuum...end.
progress: 60.0 s, 3667.4 tps, lat 13.614 ms stddev 16.914, 0 failed
progress: 120.0 s, 3776.2 tps, lat 13.234 ms stddev 16.305, 0 failed
progress: 180.0 s, 3779.8 tps, lat 13.224 ms stddev 16.139, 0 failed
progress: 240.0 s, 3796.7 tps, lat 13.164 ms stddev 15.945, 0 failed
progress: 300.0 s, 3735.5 tps, lat 13.380 ms stddev 16.484, 0 failed
progress: 360.0 s, 3768.4 tps, lat 13.260 ms stddev 16.520, 0 failed
progress: 420.0 s, 3750.4 tps, lat 13.327 ms stddev 16.397, 0 failed
progress: 480.0 s, 3739.2 tps, lat 13.368 ms stddev 16.642, 0 failed
progress: 540.0 s, 3653.0 tps, lat 13.680 ms stddev 17.088, 0 failed
progress: 600.0 s, 3695.8 tps, lat 13.525 ms stddev 17.082, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2241795
number of failed transactions: 0 (0.000%)
latency average = 13.376 ms
latency stddev = 16.553 ms
initial connection time = 44.467 ms
tps = 3736.363908 (without initial connection time)
```
> [!IMPORTANT]
> Изменения в рамках погрешности, как мне кажется.

Tеперь **Online transaction processing**:
```
max_connections = 50
shared_buffers = 1536MB
effective_cache_size = 4608MB
maintenance_work_mem = 768MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 5242kB
huge_pages = off
min_wal_size = 4GB
max_wal_size = 16GB
max_worker_processes = 6
max_parallel_workers_per_gather = 3
max_parallel_workers = 6
max_parallel_maintenance_workers = 3
```
Повторный запуск:
```
# pgbench -c 50 -j 2 -P 60 -T 600 -U postgres benchmark
pgbench (15.8 (Debian 15.8-1.pgdg120+1))
starting vacuum...end.
progress: 60.0 s, 3583.8 tps, lat 13.933 ms stddev 17.207, 0 failed
progress: 120.0 s, 3707.8 tps, lat 13.479 ms stddev 16.708, 0 failed
progress: 180.0 s, 3697.3 tps, lat 13.519 ms stddev 16.822, 0 failed
progress: 240.0 s, 3763.0 tps, lat 13.279 ms stddev 16.162, 0 failed
progress: 300.0 s, 3762.1 tps, lat 13.288 ms stddev 16.340, 0 failed
progress: 360.0 s, 3768.9 tps, lat 13.260 ms stddev 16.548, 0 failed
progress: 420.0 s, 3744.2 tps, lat 13.350 ms stddev 16.631, 0 failed
progress: 480.0 s, 3757.4 tps, lat 13.303 ms stddev 16.121, 0 failed
progress: 540.0 s, 3748.6 tps, lat 13.330 ms stddev 16.346, 0 failed
progress: 600.0 s, 3755.3 tps, lat 13.311 ms stddev 16.317, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2237353
number of failed transactions: 0 (0.000%)
latency average = 13.403 ms
latency stddev = 16.521 ms
initial connection time = 42.875 ms
tps = 3728.942101 (without initial connection time)

```
> [!IMPORTANT]
> Тоже в рамках погрешности.

> [!WARNING]
> WebApplication не стал тестировать, т.к. настройки pgtune выдал такие же как для **Online transaction processing**.

> [!IMPORTANT]
> К сожалению, все мои попытки поиграться с конфигом не дали ничего лучше чем выдал pgtune. Ощущение что то ли *fsync* так влияет и не дает получить разницу, или бенчмарк так хорош что даже под разные цели базу хорошо оценивает.
