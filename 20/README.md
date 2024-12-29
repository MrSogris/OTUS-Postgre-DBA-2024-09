# Задание 20
## Резервное копирование и восстановление 
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 15.8

Создадим схему и таблицу в ней
```
# psql -U postgres
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# CREATE DATABASE test;
CREATE DATABASE
postgres=# \c test
You are now connected to database "test" as user "postgres".
test=# CREATE SCHEMA homework;
CREATE SCHEMA
test=# CREATE TABLE homework.clients (id SERIAL NOT NULL PRIMARY KEY, name VARCHAR(255) NOT NULL, address VARCHAR(2000) NOT NULL, isImportant BOOLEAN NOT NULL DEFAULT false);
CREATE TABLE
test=# INSERT INTO homework.clients (name, address) SELECT md5(random()::text) "name", md5(random()::text) "address"  FROM generate_series (1, 100);
INSERT 0 100
```
Далее, сделаем каталог, в который PostgreSQL сможет складывать бэкапы:
```
# cd /var
# mkdir pg_backups
# chmod 777 pg_backups
```
Выгружаем данные из таблицы в бэкап с помощью *COPY*
```
# psql -U postgres
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# \c test
You are now connected to database "test" as user "postgres".
test=# COPY homework.clients TO '/var/pg_backups/logic_backup.csv' WITH (FORMAT CSV, HEADER);
COPY 100
```
Создадим новую таблицу и развернем бэкап в нее
```
test=# CREATE TABLE homework.clients_backup (id SERIAL NOT NULL PRIMARY KEY, name VARCHAR(255) NOT NULL, address VARCHAR(2000) NOT NULL, isImportant BOOLEAN NOT NULL DEFAULT false);
CREATE TABLE
test=# COPY homework.clients_backup FROM '/var/pg_backups/logic_backup.csv' WITH (FORMAT CSV, HEADER);
COPY 100
test=# SELECT * FROM homework.clients_backup LIMIT 10;
 id |               name               |             address              | isimportant 
----+----------------------------------+----------------------------------+-------------
  1 | 7d7f805f4f9dd016a215d6e1b78fd631 | e35414ced76ceff0afa1746087c7ce34 | f
  2 | 475f546828894d456c450a69d7a4560a | 7e5df3ffc6cbd28eea5dd9e335cf0fbc | f
  3 | f24dfd06bdeeb21add780b372199738f | 5ba5df7e9e2442778c7e4ddac709bb1a | f
  4 | 5557bcd33185b0011862cabe97789514 | a4543d25411dd6a1018544a199b2b39c | f
  5 | 4e554da042565cbc0d2c95c186be21f7 | ab15fa2c3fa29c8a2c7e825c53f100d4 | f
  6 | 138a7cf2822d2f0ec816e8344ae2832b | fdb2b94e58c2fc180c2f71650d7db080 | f
  7 | bd3d1a55962022e0bd672b1ef2e36dce | ea49edf4cd106b930bff2b861de78ab1 | f
  8 | 2ecbbba62d61e22870ae04c95669e70c | 871b0fc3fe76418cdf1d3e7a162648b2 | f
  9 | 96640587b121c0d28b81378f16c28b53 | d97acc5ba3c2bb191a0bc53b181821f1 | f
 10 | 674293fc0e16723d32f1d74720767ffd | f091891635d1ef0b2e998152bdf5bdbc | f
(10 rows)
```
> [!NOTE]
> Во второй таблице появились данные.

Далее, воспользуемся утилитой *pg_dump*, чтобы сделать бэкап
```
# pg_dump -U postgres --schema=homework --format=custom -t homework.clients -t homework.clients_backup test > /var/pg_backups/backup.dump
```
Теперь восстановим **clients_backup**. Для того чтобы точно увидеть что бэкап сработал, в этой таблице изменим поле **isImportant**
```
# psql -U postgres
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# \c test
You are now connected to database "test" as user "postgres".
test=# UPDATE homework.clients_backup SET isImportant = true;
UPDATE 100
test=# \q
# pg_restore -v -U postgres --clean --dbname=test --format=custom -t clients_backup /var/pg_backups/backup.dump
pg_restore: connecting to database for restore
pg_restore: dropping TABLE clients_backup
pg_restore: creating TABLE "homework.clients_backup"
pg_restore: processing data for table "homework.clients_backup"
# psql -U postgres
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# \c test
You are now connected to database "test" as user "postgres".
test=# SELECT COUNT(*) FROM homework.clients_backup WHERE isImportant = true;
 count 
-------
     0
(1 row)
```
> [!NOTE]
> Строк, где **isImportant** = true не нашлось.

> [!WARNING]
> О проблемах
> 1. *pg_restore* лучше запускать с флагом *-v*, иначе невозможно будет понять что он там восстановил вообще и восстановил ли.
> 2. *pg_restore* не понимает название таблицы со схемой
> 3. *pg_restore* если восстанавливает уже существующую таблицу без флага *--clean* как оказалось не в состоянии пересоздать поле с типом *SERIAL*. И без *-v* не говорит что свалился в ошибку.
> В общем, дамп было сделать в сотню раз проще.
