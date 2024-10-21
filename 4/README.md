# Задание 4
## Работа с базами данных, пользователями и правами
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 14 версии
### Создание и настройка объектов
После старта контейнера подключаемся к СУБД:
```
psql -U postgres
psql (14.13 (Debian 14.13-1.pgdg120+1))
```
Далее, настраиваем объекты согласно заданию:
```
postgres=# CREATE DATABASE testdb;
CREATE DATABASE
postgres=# \c testdb
You are now connected to database "testdb" as user "postgres".
testdb=# CREATE SCHEMA testnm;
CREATE SCHEMA
testdb=# CREATE TABLE testnm.t1 (c1 INTEGER);
CREATE TABLE
testdb=# INSERT INTO testnm.t1 VALUES(1);
INSERT 0 1
testdb=# CREATE ROLE readonly;
CREATE ROLE
testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
GRANT
testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
GRANT
testdb=# GRANT SELECT ON  ALL TABLES IN SCHEMA testnm TO readonly;
GRANT
testdb=# CREATE USER testread WITH PASSWORD 'test123';
CREATE ROLE
testdb=# GRANT readonly TO testread;
GRANT ROLE
testdb=# \q
```
> [!IMPORTANT]
> В этом месте осознал что в задании не указали что при создании таблицы надо указать схему (возможно специально), а я на автомате ожидая что раз речь шла про схему вбил ее.

Заходим в СУБД уже под пользователем **testread**
```
# psql -U testread -d testdb
```
Пытаемся получить данные из таблицы согласно готовому запросу:
```
testdb=> select * from t1
testdb-> ;
ERROR:  relation "t1" does not exist
LINE 1: select * from t1
                      ^
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

testdb=> SHOW search_path;
   search_path
-----------------
 "$user", public
(1 row)
```
> [!IMPORTANT]
> Не вышло, все потому что таблица создана в схеме, а мы ее не указали, поиск идет в *public*. Почему *public*? Потому что search_path у данного пользователя не знает про схему *testnm*.

Правим запрос и заодно проверяем *search_path*:
```
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)

testdb=> SHOW search_path;
   search_path
-----------------
 "$user", public
(1 row)
```
> [!IMPORTANT]
> Вышесказанное подтверждено.

Теперь попытка создать таблицу:
```
testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
```
> [!IMPORTANT]
> Получилось, оно и понятно - *search_path* выводили выше и мы знаем что таблица создастся в *public*, а права мы резали на *testnm*. Если бы мы тут явным образом указали схему, то получили бы ошибку.

Нужно исправить пользователя, чтобы он не лазил в *public* по дефолту:
```
# psql -U postgres
psql (14.13 (Debian 14.13-1.pgdg120+1))
Type "help" for help.

postgres=# alter user testread set SEARCH_PATH = 'testnm';
ALTER ROLE
```
А теперь снова пытаемся что-то создать:
```
# psql -U testread -d testdb
psql (14.13 (Debian 14.13-1.pgdg120+1))
Type "help" for help.

testdb=> create table t3(c1 integer); insert into t2 values (2);
ERROR:  permission denied for schema testnm
LINE 1: create table t3(c1 integer);
                     ^
ERROR:  relation "t2" does not exist
LINE 1: insert into t2 values (2);
```
> [!IMPORTANT]
> Все теперь такая команда уже не работает, схемой по умолчанию стала *testnm* а на нее прав никто не давал.
