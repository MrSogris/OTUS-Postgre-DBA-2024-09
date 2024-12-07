# Задание 15
## Секционирование
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 15.8
### Создание тестовой БД и заполнение ее данными
Скачиваем с сайта Postgres Pro демо-БД "Авиаперевозки и развертываем ее:
```
# psql -U postgres -a -f /win_share/demo-medium-20170815.sql
```
Для подбора решения по секционированию обратимся к наглядной структуре БД - [Посмотреть диаграмму](https://postgrespro.ru/docs/postgrespro/15/demodb-schema-diagram)
В общем случае чтобы выбрать что и как будем секционировать, конечно хотелось бы понимать как и для чего используется БД, чтобы правильно "разрезать" данные и как можно чаще попадать в ситуации где нам не потребуется связывать данные между секциями.
Т.к. такой информации нет, то попробуем поработать с самой большой таблицей в БД, это **Ticket_flights**.
```
demo=# SELECT COUNT(*) FROM Ticket_flights;
  count  
---------
 2360335
(1 row)
```
Однако структура ее такая что ее удобно ее секционировать выйдет судя по по всему, только по хэшу.
Еще одна большая таблица - **Bookings**
```
demo=# SELECT COUNT(*) FROM Bookings;
 count  
--------
 593433
(1 row)
```
Ее как раз будет удобно секционировать по датам. Т.к. логично предпололжить что бронирования интересны только в разрезе диапазона дат еще не улетевших рейсов, можем разбить таблицу по колонке *book_date*.
Чтобы секционировать таблицу, создадим ее копию:
```
CREATE TABLE Bookings_partitioned (
    book_ref character(6) NOT NULL,
    book_date timestamp with time zone NOT NULL,
    total_amount numeric(10,2) NOT NULL
) PARTITION BY RANGE (book_date);
```
Определим диапазон дат:
```
demo=# SELECT MIN(book_date),  MAX(book_date) FROM Bookings;
          min           |          max           
------------------------+------------------------
 2017-04-21 11:23:00+00 | 2017-08-15 15:00:00+00
(1 row)
```
Далее, разобьем таблицу по кварталам:
```
CREATE TABLE Bookings_partitioned_2017Q1 PARTITION OF Bookings_partitioned FOR VALUES FROM ('2017-01-01') TO ('2017-04-01');
CREATE TABLE Bookings_partitioned_2017Q2 PARTITION OF Bookings_partitioned FOR VALUES FROM ('2017-04-01') TO ('2017-07-01');
CREATE TABLE Bookings_partitioned_2017Q3 PARTITION OF Bookings_partitioned FOR VALUES FROM ('2017-07-01') TO ('2017-10-01');
INSERT INTO Bookings_partitioned SELECT * FROM Bookings;
```
Для учета будущих периодов, можем запланировать команду вида
```
psql -U postgres -d demo -c "CREATE TABLE Bookings_partitioned_2017Q1 PARTITION OF Bookings_partitioned FOR VALUES FROM ('2017-01-01') TO ('2017-04-01');"
```
Теперь "подменяем" старую таблицу на новую:
```
ALTER TABLE Tickets DROP CONSTRAINT tickets_book_ref_fkey;
ALTER TABLE Bookings RENAME TO Bookings_old;
ALTER TABLE Bookings_partitioned RENAME TO Bookings;
```
> [!WARNING]
> А вот тут есть проблема. Мы включили партиционирование, но PK должен включать все колонки по которым идет разбиение. Ничто не мешает конечно сделать сложный PK, но тогда FK не создатся. Пришлось пока что его убрать.

Проверим план запроса:
```
EXPLAIN SELECT * FROM Bookings Where book_date > '2017-01-01' AND book_date  <  '2017-03-01';
QUERY PLAN                                                                     
---------------------------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on bookings_partitioned_2017q1 bookings  (cost=0.00..25.30 rows=5 width=52)
   Filter: ((book_date > '2017-01-01 00:00:00+00'::timestamp with time zone) AND (book_date < '2017-03-01 00:00:00+00'::timestamp with time zone))
(2 rows)
```
> [!NOTE]
> План запроса учитывает партиционирование.
