# Задание 21
## Виды и устройство репликации в PostgreSQL. Практика применения
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 15.8

### Создание контейнеров
Нам понадобится 3 контейнера с СУБД:
```
docker run --hostname=1d3a37668d37 --mac-address=02:42:ac:11:00:02 --env=POSTGRES_PASSWORD=password --env=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/postgresql/15/bin --env=GOSU_VERSION=1.17 --env=LANG=en_US.utf8 --env=PG_MAJOR=15 --env=PG_VERSION=15.8-1.pgdg120+1 --env=PGDATA=/var/lib/postgresql/data --volume=/var/lib/postgresql/data --network=bridge -p 5432:5432 --restart=no --runtime=runc -d postgres:15.8
docker run --hostname=0c06635d9b3f --mac-address=02:42:ac:11:00:03 --env=POSTGRES_PASSWORD=password --env=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/postgresql/15/bin --env=GOSU_VERSION=1.17 --env=LANG=en_US.utf8 --env=PG_MAJOR=15 --env=PG_VERSION=15.8-1.pgdg120+1 --env=PGDATA=/var/lib/postgresql/data --volume=/var/lib/postgresql/data --network=bridge -p 5433:5432 --restart=no --runtime=runc -d postgres:15.8
docker run --hostname=dacb660c33db --mac-address=02:42:ac:11:00:04 --env=POSTGRES_PASSWORD=password --env=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/lib/postgresql/15/bin --env=GOSU_VERSION=1.17 --env=LANG=en_US.utf8 --env=PG_MAJOR=15 --env=PG_VERSION=15.8-1.pgdg120+1 --env=PGDATA=/var/lib/postgresql/data --volume=/var/lib/postgresql/data --network=bridge -p 5444:5432 --restart=no --runtime=runc -d postgres:15.8
```
Для того чтобы не поднимаеть кубер, решил просто завести машины на разных портах. Хотя конечно, корректнее и красивее было бы через кубер это сделать (но т.к. не девопс - решил не заморачиваться).
### Настройка СУБД в контейнерах
На контейнере с портом 5432:
```
# psql -U postgres
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# CREATE TABLE test(code varchar(10) PRIMARY KEY, name varchar(255));
CREATE TABLE
postgres=# CREATE TABLE test2(code varchar(10) PRIMARY KEY, name varchar(255));
CREATE TABLE
postgres=# CREATE PUBLICATION test_pub FOR TABLE test;
CREATE PUBLICATION
postgres=# \q
```
На контейнере 5433
```
# psql -U postgres
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# CREATE TABLE test(code varchar(10) PRIMARY KEY, name varchar(255));
CREATE TABLE
postgres=# CREATE TABLE test2(code varchar(10) PRIMARY KEY, name varchar(255));
CREATE TABLE
postgres=# CREATE PUBLICATION test_pub FOR TABLE test2;
CREATE PUBLICATION
postgres=# CREATE SUBSCRIPTION test_sub CONNECTION 'host=192.168.88.126 port=5432 user=postgres password=password dbname=postgres' PUBLICATION test_pub WITH(copy_data = false);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
```
Теперь в первом контейнере добавляем подписку на второй:
```
# psql -U postgres
psql (15.8 (Debian 15.8-1.pgdg120+1))
Type "help" for help.

postgres=# CREATE SUBSCRIPTION test_sub CONNECTION 'host=192.168.88.126 port=5433 user=postgres password=password dbname=postgres' PUBLICATION test_pub WITH(copy_data = false);
NOTICE:  created replication slot "test_sub" on publisher
CREATE SUBSCRIPTION
```
### Проверка
На контейнере 5432 добавим строку в таблицу *test*
```
postgres=# INSERT INTO test VALUES('code1', 'testName');
INSERT 0 1
```
Далее на контейнере с 5433 проверяем содержимое таблицы
```
postgres=# SELECT * FROM test;
 code  |   name   
-------+----------
 code1 | testName
(1 row)
```
Зеркальные действия с *test2*
```
postgres=# INSERT INTO test2 VALUES ('code2', 'name2');
INSERT 0 1
```
```
postgres=# SELECT * FROM test2;
 code  | name  
-------+-------
 code2 | name2
(1 row)
```
> [!NOTE]
> Синхронизация налажена.

### Создание 3-й реплики
Подписываемся в контейнере 5444 на таблицу *test* с первого контейнера и *test2* со второго
```
postgres=# CREATE TABLE test(code varchar(10) PRIMARY KEY, name varchar(255));
CREATE TABLE
postgres=# CREATE TABLE test2(code varchar(10) PRIMARY KEY, name varchar(255));
CREATE TABLE
postgres=# CREATE SUBSCRIPTION test_sub3 CONNECTION 'host=192.168.88.126 port=5432 user=postgres password=password dbname=postgres' PUBLICATION test_pub WITH(copy_data = false);
NOTICE:  created replication slot "test_sub3" on publisher
postgres=# CREATE SUBSCRIPTION test_sub4 CONNECTION 'host=192.168.88.126 port=5433 user=postgres password=password dbname=postgres' PUBLICATION test_pub WITH(copy_data = false);
NOTICE:  created replication slot "test_sub4" on publisher
```
> [!WARNING]
> Тут оказалось что при логической репликации уже ранее созданные записи в нее не попадают. И пришлось еще добавить записей в исходные таблицы.

После добавления записи стали появляться:
```
postgres=# SELECT * FROM test;
 code  |   name    
-------+-----------
 code5 | testName5
(1 row)
```
> [!WARNING]
> Выходит, если нужна реплика прямо для бэкапов гарантированных, надо бы ее делать физической. Но при этом физическая реплика не позволяет скажем брать таблицу 1 с сервера 1 а таблицу 2 с сервера 2. А логика слегка работает неудобно.
