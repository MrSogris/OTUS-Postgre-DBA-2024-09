# Задание 13
## Виды индексов. Работа с индексами и оптимизация запросов 
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 15.8
### Создание тестовой БД и заполнение ее данными
```
CREATE TABLE Cities (Id serial PRIMARY KEY, Name varchar(255) NOT NULL UNIQUE, PostalIndex int NOT NULL UNIQUE);
CREATE TABLE Clients (Id serial PRIMARY KEY, Name varchar(255) NOT NULL, "Login" varchar(20) NOT NULL UNIQUE, CityId int NOT NULL references Cities(Id), Address tsvector NOT NULL);
CREATE TABLE Orders (Id serial PRIMARY KEY, OrderNumber varchar(255) NOT NULL UNIQUE, ClientId int NOT NULL references Clients(Id), TotalSum money NOT NULL);
INSERT INTO Cities (Name, PostalIndex) SELECT md5(random()::text) "Name", 100000 + floor(random() * 899999) "PostalIndex" FROM generate_series (1, 1000);
INSERT INTO Clients (Name, "Login", CityId, Address) SELECT md5(random()::text) "Name", Left(md5(random()::text), 20) "Login", 1 + floor(random() * 1000) CityId, md5(random()::text) "Address" FROM generate_series (1, 2000);
INSERT INTO Orders (OrderNumber, ClientId, TotalSum) SELECT md5(random()::text) "OrderNumber", 1 + floor(random() * 2000) ClientId, 0.01 + random() * 10000 FROM generate_series (1, 2000);
```
### Уникальный Btree индекс
```
postgres=# CREATE UNIQUE INDEX IX_Cities ON Cities (Name);
CREATE INDEX
```
План запроса с этим индексом:
```
postgres=# EXPLAIN SELECT * FROM Cities WHERE Name = 'e26f65c7e3a7c3b6e6f02217fa71956f';
                                QUERY PLAN
--------------------------------------------------------------------------
 Index Scan using ix_cities2 on cities  (cost=0.28..8.29 rows=1 width=41)
   Index Cond: ((name)::text = 'e26f65c7e3a7c3b6e6f02217fa71956f'::text)
(2 rows)
```
### Полнотекстовый индекс
```
postgres=# CREATE INDEX IX_Clients_text ON Clients USING GIN(Address);
CREATE INDEX
```
Пробуем его в действии:
```
postgres=# EXPLAIN SELECT * FROM Clients WHERE Address @@ plainto_tsquery('test');
                                  QUERY PLAN
-------------------------------------------------------------------------------
 Bitmap Heap Scan on clients  (cost=12.25..16.52 rows=1 width=103)
   Recheck Cond: (address @@ plainto_tsquery('test'::text))
   ->  Bitmap Index Scan on ix_clients_text  (cost=0.00..12.25 rows=1 width=0)
         Index Cond: (address @@ plainto_tsquery('test'::text))
(4 rows)
```
> [!NOTE]
> Прекрасно работает.
### Индекс на часть таблицы
```
CREATE INDEX IX_Cities_Partial ON Cities(PostalIndex) WHERE PostalIndex > 200000;
CREATE INDEX
```
> [!WARNING]
> А этот индекс я запустить не смог, вставлял в условие такой же текст как в индексе, но в итоге всегда имеем Sequnce Scan в плане, так и не смог понять почему. Видимо сервер решил что так выгоднее будет... Может есть смысл в этот индекс колонок еще с данными добавить.
### Индекс на несколько полей
```
CREATE UNIQUE INDEX IX_Orders ON Orders (OrderNumber, ClientId);
CREATE INDEX
```
Пробуем его в действии:
```
postgres=# EXPLAIN SELECT * FROM Orders WHERE ClientId = 5 AND OrderNumber = 'SomeOrder';
                                  QUERY PLAN
------------------------------------------------------------------------------
 Index Scan using ix_orders on orders  (cost=0.28..8.30 rows=1 width=49)
   Index Cond: (((ordernumber)::text = 'SomeOrder'::text) AND (clientid = 5))
(2 rows)
```
> [!NOTE]
> Прекрасно работает.
