# Задание 6
## Настройка autovacuum с учетом особеностей производительности
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 15.8

Для начала запускаем контейнер и вводим ему ограничения на ресурсы:
```
docker run --env=POSTGRES_PASSWORD=password --volume=D:\psql:/win_share --network=bridge -p 5432:5432 -d --memory=4g --cpus="2.0" postgres:15.8
```
Переходим в интекрактивный режим и инициализируем бенчмарки:
```
pgbench -U postgres -i postgres
pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.8 (Debian 15.8-1.pgdg120+1))
starting vacuum...end.
progress: 6.0 s, 574.0 tps, lat 13.861 ms stddev 10.798, 0 failed
progress: 12.0 s, 605.8 tps, lat 13.204 ms stddev 9.819, 0 failed
progress: 18.0 s, 671.0 tps, lat 11.925 ms stddev 9.349, 0 failed
progress: 24.0 s, 587.3 tps, lat 13.604 ms stddev 10.740, 0 failed
progress: 30.0 s, 602.5 tps, lat 13.269 ms stddev 10.619, 0 failed
progress: 36.0 s, 570.0 tps, lat 14.014 ms stddev 11.117, 0 failed
progress: 42.0 s, 616.3 tps, lat 12.967 ms stddev 10.154, 0 failed
progress: 48.0 s, 314.7 tps, lat 25.323 ms stddev 23.863, 0 failed
progress: 54.0 s, 205.0 tps, lat 39.095 ms stddev 25.066, 0 failed
progress: 60.0 s, 205.5 tps, lat 38.885 ms stddev 24.164, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 29721
number of failed transactions: 0 (0.000%)
latency average = 16.142 ms
latency stddev = 15.314 ms
initial connection time = 13.382 ms
tps = 495.131490 (without initial connection time)
```
Меняем конфигурацию:
```
max_connections = 40
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 65536kB
min_wal_size = 4GB
max_wal_size = 16GB
```
И заново запускаем тесты
```
pgbench (15.8 (Debian 15.8-1.pgdg120+1))
starting vacuum...end.
progress: 6.0 s, 528.0 tps, lat 15.087 ms stddev 11.018, 0 failed
progress: 12.0 s, 558.8 tps, lat 14.302 ms stddev 11.297, 0 failed
progress: 18.0 s, 628.5 tps, lat 12.717 ms stddev 9.248, 0 failed
progress: 24.0 s, 612.0 tps, lat 13.058 ms stddev 9.944, 0 failed
progress: 30.0 s, 611.3 tps, lat 13.083 ms stddev 10.212, 0 failed
progress: 36.0 s, 631.7 tps, lat 12.650 ms stddev 10.004, 0 failed
progress: 42.0 s, 639.1 tps, lat 12.492 ms stddev 9.433, 0 failed
progress: 48.0 s, 465.3 tps, lat 17.139 ms stddev 17.030, 0 failed
progress: 54.0 s, 206.5 tps, lat 38.787 ms stddev 24.674, 0 failed
progress: 60.0 s, 201.2 tps, lat 39.755 ms stddev 27.730, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 30503
number of failed transactions: 0 (0.000%)
latency average = 15.728 ms
latency stddev = 14.748 ms
initial connection time = 13.473 ms
tps = 508.139824 (without initial connection time)
#
```
> [!IMPORTANT]
> Немного выросла производительность, вероятно из-за shared_buffers

Создаем таблицу с одним текстовым полем и заполняем его миллионом строк:
```
postgres=#CREATE TABLE testTable(column1 varchar(255));
CREATE TABLE
postgres=# INSERT INTO testTable SELECT md5(random()::text) FROM generate_series(1, 1000000);
INSERT 0 1000000
```
Смотрим размер таблицы:
```
postgres=# \d+
                                         List of relations
 Schema |       Name       | Type  |  Owner   | Persistence | Access method |  Size   | Description 
--------+------------------+-------+----------+-------------+---------------+---------+-------------
 public | pgbench_accounts | table | postgres | permanent   | heap          | 13 MB   | 
 public | pgbench_branches | table | postgres | permanent   | heap          | 40 kB   | 
 public | pgbench_history  | table | postgres | permanent   | heap          | 1624 kB | 
 public | pgbench_tellers  | table | postgres | permanent   | heap          | 80 kB   | 
 public | testtable        | table | postgres | permanent   | heap          | 65 MB   | 
(5 rows)
```
Обновляем строки 5 раз добавляя по символу:
```
postgres=# UPDATE testTable SET column1 = column1 || 'x';
UPDATE 1000000
postgres=# UPDATE testTable SET column1 = column1 || 'x';
UPDATE 1000000
postgres=# UPDATE testTable SET column1 = column1 || 'x';
UPDATE 1000000
postgres=# UPDATE testTable SET column1 = column1 || 'x';
UPDATE 1000000
postgres=# UPDATE testTable SET column1 = column1 || 'x';
UPDATE 1000000
postgres=# SELECT relname, n_live_tup, n_dead_tup, last_vacuum, last_autovacuum FROM pg_stat_user_tables;
     relname      | n_live_tup | n_dead_tup |          last_vacuum          |        last_autovacuum        
------------------+------------+------------+-------------------------------+-------------------------------
 testtable        |    1000000 |    4999814 |                               | 
 pgbench_history  |      30503 |          0 | 2024-10-27 08:03:45.829534+00 | 2024-10-27 08:16:39.279742+00
 pgbench_accounts |     100000 |       4359 | 2024-10-27 08:03:45.806474+00 | 2024-10-27 08:03:56.311509+00
 pgbench_tellers  |         10 |          0 | 2024-10-27 08:15:39.163414+00 | 2024-10-27 08:16:39.265756+00
 pgbench_branches |          1 |          0 | 2024-10-27 08:15:39.162463+00 | 2024-10-27 08:17:34.822207+00
(5 rows)
```
> [!IMPORTANT]
> Вышло не с первого раза, иногда автовакуум успевает подчистить таблицу *testtable*

Подождем (попьем чаю) и получаем:
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, last_vacuum, last_autovacuum FROM pg_stat_user_tables;
     relname      | n_live_tup | n_dead_tup |          last_vacuum          |        last_autovacuum        
------------------+------------+------------+-------------------------------+-------------------------------
 testtable        |    1000000 |          0 |                               | 2024-10-27 11:48:55.041631+00
 pgbench_history  |      30503 |          0 | 2024-10-27 08:03:45.829534+00 | 2024-10-27 08:16:39.279742+00
 pgbench_accounts |     100000 |       4359 | 2024-10-27 08:03:45.806474+00 | 2024-10-27 08:03:56.311509+00
 pgbench_tellers  |         10 |          0 | 2024-10-27 08:15:39.163414+00 | 2024-10-27 08:16:39.265756+00
 pgbench_branches |          1 |          0 | 2024-10-27 08:15:39.162463+00 | 2024-10-27 08:17:34.822207+00
(5 rows)
```
> [!IMPORTANT]
> Все, автовакуум уже раз сходил в таблицу и почистил "хвосты".

Обновляем строки 5 раз добавляя по символу:
```
postgres=# UPDATE testTable SET column1 = column1 || 'x';
UPDATE 1000000
postgres=# UPDATE testTable SET column1 = column1 || 'x';
UPDATE 1000000
postgres=# UPDATE testTable SET column1 = column1 || 'x';
UPDATE 1000000
postgres=# UPDATE testTable SET column1 = column1 || 'x';
UPDATE 1000000
postgres=# UPDATE testTable SET column1 = column1 || 'x';
UPDATE 1000000
postgres=# \d+
                                         List of relations
 Schema |       Name       | Type  |  Owner   | Persistence | Access method |  Size   | Description 
--------+------------------+-------+----------+-------------+---------------+---------+-------------
 public | pgbench_accounts | table | postgres | permanent   | heap          | 13 MB   | 
 public | pgbench_branches | table | postgres | permanent   | heap          | 40 kB   | 
 public | pgbench_history  | table | postgres | permanent   | heap          | 1624 kB | 
 public | pgbench_tellers  | table | postgres | permanent   | heap          | 80 kB   | 
 public | testtable        | table | postgres | permanent   | heap          | 414 MB  | 
(5 rows)
```
Отключаем автовакуум на *testtable*, запускаем анонимный блок кода чтобы обновить все строки по 10 раз и смотрим на итоговый размер:
```
postgres=# ALTER TABLE testTable SET (autovacuum_enabled = false);
ALTER TABLE
postgres=# DO $$BEGIN 
   FOR i IN 1..10 LOOP
      UPDATE testTable SET column1 = column1 || 'x';
   END LOOP;
END$$;
DO
postgres=# \d+
                                         List of relations
 Schema |       Name       | Type  |  Owner   | Persistence | Access method |  Size   | Description 
--------+------------------+-------+----------+-------------+---------------+---------+-------------
 public | pgbench_accounts | table | postgres | permanent   | heap          | 13 MB   | 
 public | pgbench_branches | table | postgres | permanent   | heap          | 40 kB   | 
 public | pgbench_history  | table | postgres | permanent   | heap          | 1624 kB | 
 public | pgbench_tellers  | table | postgres | permanent   | heap          | 80 kB   | 
 public | testtable        | table | postgres | permanent   | heap          | 1378 MB | 
(5 rows)
```
> [!IMPORTANT]
> Бесконечные апдейты и отсутвие VACUUM FULL привели к тому что таблица уже неприлично "распухла".
