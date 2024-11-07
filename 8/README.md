# Задание 8
## Работа с журналами
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 15 версии

Дописываем в *postgresql.conf* строку:
```
checkpoint_timeout = 30s
```
и перезапускаем СУБД. Далее инициализируем тестовую базу, бенчмаркинг:
```
# psql -U postgres
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# CREATE DATABASE benchmark;
CREATE DATABASE
postgres=# \q
# pgbench -i -U postgres benchmark
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.05 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.25 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.14 s, vacuum 0.03 s, primary keys 0.07 s).
# pgbench -c 50 -j 2 -P 60 -T 600 -U postgres benchmark
pgbench (15.8 (Debian 15.8-1.pgdg120+1))
starting vacuum...end.
progress: 60.0 s, 582.1 tps, lat 85.617 ms stddev 157.137, 0 failed
progress: 120.0 s, 583.9 tps, lat 85.287 ms stddev 156.309, 0 failed
progress: 180.0 s, 578.4 tps, lat 86.870 ms stddev 160.102, 0 failed
progress: 240.0 s, 592.1 tps, lat 84.404 ms stddev 174.457, 0 failed
progress: 300.0 s, 571.0 tps, lat 87.174 ms stddev 191.836, 0 failed
progress: 360.0 s, 581.9 tps, lat 86.241 ms stddev 160.555, 0 failed
progress: 420.0 s, 570.5 tps, lat 87.371 ms stddev 158.820, 0 failed
progress: 480.0 s, 578.1 tps, lat 86.633 ms stddev 143.777, 0 failed
progress: 540.0 s, 600.0 tps, lat 83.373 ms stddev 179.548, 0 failed
progress: 600.0 s, 592.2 tps, lat 84.588 ms stddev 159.811, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 349862
number of failed transactions: 0 (0.000%)
latency average = 85.745 ms
latency stddev = 164.780 ms
initial connection time = 43.002 ms
tps = 583.045866 (without initial connection time)
```
> [!IMPORTANT]
> Намерили 583.045866 tps

Смотрим размер журнала:
```
# cd /var/lib/postgresql/data
# cd pg_wal
# ls -l --block-size=M
total 65M
-rw------- 1 postgres postgres 16M Nov  7 04:03 00000001000000000000001F
-rw------- 1 postgres postgres 16M Nov  7 04:01 000000010000000000000020
-rw------- 1 postgres postgres 16M Nov  7 04:01 000000010000000000000021
-rw------- 1 postgres postgres 16M Nov  7 04:01 000000010000000000000022
drwx------ 2 postgres postgres  1M Nov  7 03:44 archive_status
```
> [!IMPORTANT]
> 64 Мб нагенерировали в итоге. Если исходить из того что тестирование шло 600 сек., должно было создаться 20 контрольных точек. Т.е. одна точка "весит" около 3.2 Мб.

Проверим статистику:
```
postgres=# select * from pg_stat_bgwriter;
 checkpoints_timed | checkpoints_req | checkpoint_write_time | checkpoint_sync_time | buffers_checkpoint | buffers_clean | maxwritten_clean | buffers_backend | buffers_backend_fsync | buffers_alloc |         stats_reset          
-------------------+-----------------+-----------------------+----------------------+--------------------+---------------+------------------+-----------------+-----------------------+---------------+------------------------------
                72 |               2 |                592601 |                  459 |              50092 |             0 |                0 |            4737 |                     0 |          6812 | 2024-11-07 03:44:04.37381+00
(1 row)
```
> [!IMPORTANT]
> Видим, что *checkpoints_timed* не ноль, было 2 контрольных точки не по расписанию. Т.к. по умолчанию у меня в файле стоит размер WAL файла - 1Гб, его размер не был превышен, и можно сделать вывод что 2 точки просто не успели создаться, т.к. не завершилось создание прыдедущей точки. Вообще по мануалу 30 сек. - это очень мало для интервала создания точек.

Включаем асинхронный commit в конфигурационном файле:
```
synchronous_commit = off
```
и снова бенчмаркинг:
```
# pgbench -c 50 -j 2 -P 60 -T 600 -U postgres benchmark
pgbench (15.8 (Debian 15.8-1.pgdg120+1))
starting vacuum...end.
progress: 60.0 s, 3605.1 tps, lat 13.850 ms stddev 16.884, 0 failed
progress: 120.0 s, 3570.4 tps, lat 14.000 ms stddev 17.383, 0 failed
progress: 180.0 s, 3555.6 tps, lat 14.056 ms stddev 17.149, 0 failed
progress: 240.0 s, 3657.7 tps, lat 13.665 ms stddev 16.635, 0 failed
progress: 300.0 s, 3647.0 tps, lat 13.704 ms stddev 16.827, 0 failed
progress: 360.0 s, 3494.3 tps, lat 14.302 ms stddev 17.985, 0 failed
progress: 420.0 s, 3509.9 tps, lat 14.239 ms stddev 17.577, 0 failed
progress: 480.0 s, 3644.8 tps, lat 13.715 ms stddev 16.956, 0 failed
progress: 540.0 s, 3693.9 tps, lat 13.530 ms stddev 16.453, 0 failed
progress: 600.0 s, 3584.6 tps, lat 13.944 ms stddev 17.193, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2157849
number of failed transactions: 0 (0.000%)
latency average = 13.897 ms
latency stddev = 17.105 ms
initial connection time = 40.138 ms
tps = 3596.417511 (without initial connection time)
```
> [!IMPORTANT]
> Намерили 3596.417511 tps, что сильно выше чем предыдущий результат. Рост производительности связан с тем что сервер теперь не ожидает завершения сброса на диск и работает дальше.

Создаем теперь контейнер с опцией
```
-e POSTGRES_INITDB_ARGS="--data-checksums"
```
Далее сделаем табличку с данными:
```
# psql -U postgres
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# CREATE TABLE test(code varchar(10), name varchar(255));
CREATE TABLE
postgres=# INSERT INTO test (code, name) VALUES('code', 'name');
INSERT 0 1
postgres=# select pg_relation_filepath('test');
pg_relation_filepath 
----------------------
 base/5/16384
(1 row)
```
Получив директорию файла таблицы, исправляем в нем пару символов и перезапустив кластер, пытаемся сделать SELECT:
```
postgres=# SELECT * FROM test;
WARNING:  page verification failed, calculated checksum 35316 but expected 564
ERROR:  invalid page in block 0 of relation base/5/16384
```
> [!IMPORTANT]
> Таблица "сломалась".

Можно "починить":
```
postgres=# SET ignore_checksum_failure TO on;
SET
postgres=# SHOW ignore_checksum_failure;
 ignore_checksum_failure 
-------------------------
 on
(1 row)

postgres=# SELECT * FROM test;
WARNING:  page verification failed, calculated checksum 35316 but expected 564
 code | name 
------+------
(0 rows)
```
> [!IMPORTANT]
> Однако получить что-то внятное не вышло...
