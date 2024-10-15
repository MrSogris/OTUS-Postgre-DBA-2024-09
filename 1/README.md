# Задание 1
## Работа с уровнями изоляции транзакции в PostgreSQL
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *ubuntu/postgres* 14 версии
### read committed
Открываем **сессию#1** в консоли, после чего запускаем клиентскую утилиту *psql*. 
После открытия соединения с СУБД создаем таблицу **persons** с тестовыми данными, не забыв открыть и закрыть транзакцию:
```
postgres=# begin;
BEGIN
postgres=*# create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
postgres=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```
В **сессии#1** открываем новую транзакцию, но не коммитим:
```
postgres=# begin;
BEGIN
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```
Открываем **сессию#2** в консоли, запускаем *psql* и пытаемся выбрать данные из таблицы **persons**:
```
postgres=# begin;
BEGIN
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
Таким образом в **сессии#2** в таблице **persons** все еще только 2 записи.
> [!IMPORTANT]
> Записи только 2, а не 3 т.к. уровень изоляции read commited делает невозможным т.н. "сырое чтение", таким образом изменения из незавершенной транзакции не видны другим транзакциям.

Завершаем транзакцию в **сессии#1**:
```
postgres=*# commit;
COMMIT
```
Повторяем свой запрос в **сессии#2**:
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
В **сессии#2** появилась 3-я запись.
> [!IMPORTANT]
> После завершения транзакции с уровнем read committed внесенные ею изменения становятся доступны всем другим транзакциям с уровнем read committed.

*Завершаем все незакрытые транзакции в обеих сессиях*
### repeatable read
В **сессии#1** открываем новую транзакцию, но не коммитим:
```
postgres=# begin;
BEGIN
postgres=*# set transaction isolation level repeatable read;
SET
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```
В **сессии#2** пытаемся выбрать данные:
```
postgres=# begin;
BEGIN
postgres=*# set transaction isolation level repeatable read;
SET
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Видим что в **сессии#2** 3 строки.
> [!IMPORTANT]
> Repeatable read - более высокий уровень изоляции, следовательно что было верно для read committed работает и здесь - пока другая транзакция не завершена, ее изменения никому не доступны.
Завершаем транзакцию в **сессии#1** и пытаемся снова сделать выборку в **сессии#2**:
```
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Ничего не изменилось, хотя транзакция и завершилась.
> [!IMPORTANT]
> Здесь явно проявляется тот факт что repeatable read изолирует не только изменения данных, но и результаты чтения. В нашем случае открытие второй транзакции приводит к тому что она постоянно читает данные с одним и тем же результатом, даже если по факту уже другая транзакция успешно завершила их изменение. Такой уровень изоляции гарантирует что результаты чтения конкретной транзакции будут изменяться только в том случае, если сама эта транзакция изменит эти данные.
Завершаем тразнакцию в **сессии#2** и повторно выполняем чтение:
```
postgres=*# commit;
COMMIT
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```
Добавленная в **сессии#1** строка доступна.
> [!IMPORTANT]
> После завершения транзакции изменения становятся доступны. Причину этого легко проверить командой *show transaction isolation level;*, она выведет read committed, следовательно мы читаем все, что изменили другие транзакции, даже если сами мы этого не изменяли.
