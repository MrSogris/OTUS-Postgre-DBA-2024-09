# Задание 7
## Механизм блокировок
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 15.8
### Логирование длительных по времени блокировок
Дописываем в файл *postgresql.conf* такие строки:
```
log_lock_waits = on
deadlock_timeout = 200
```
и рестартуем. Чтобы создать блокировку, сделаем тестовую таблицу:
```
postgres=# CREATE TABLE testTable(code varchar(10), name varchar(255));
CREATE TABLE
```
Далее, убираем автокоммит, откроем транзакцию и скажем в ней вставим данные в таблицу:
```
postgres=# BEGIN;
BEGIN
postgres=*# INSERT INTO testTable(code, name) VALUES('code', 'name');
INSERT 0 1
postgres=*# INSERT INTO testTable(code, name) VALUES('code1', 'name1');
INSERT 0 1
postgres=*# SELECT * FROM testTable;
 code  | name  
-------+-------
 code  | name
 code1 | name1
(2 rows)
```
Теперь открываем в другом контейнере клиент и попытаемся сделать *VACUUM FULL*
```
postgres=# VACUUM FULL testTable;
```
> [!IMPORTANT]
> Команда успешно "зависла"

Но стоит лишь сделать в первом сеансе 
```
postgres=*# COMMIT;
COMMIT
```
как "зависание" исчезло и команда отработала. Проверим лог-файл:
```
2024-11-04 15:01:50.291 UTC [61] LOG:  process 61 still waiting for AccessExclusiveLock on relation 16388 of database 5 after 200.182 ms
2024-11-04 15:01:50.291 UTC [61] DETAIL:  Process holding the lock: 66. Wait queue: 61.
2024-11-04 15:01:50.291 UTC [61] STATEMENT:  VACUUM FULL testTable;
2024-11-04 15:02:32.587 UTC [61] LOG:  process 61 acquired AccessExclusiveLock on relation 16388 of database 5 after 42495.393 ms
2024-11-04 15:02:32.587 UTC [61] STATEMENT:  VACUUM FULL testTable;
```
> [!IMPORTANT]
> Видим что наша блокировка "задержалась" выше положенного времени, а дальше отработал VACUUM, как только блок снялся. Команда попала в ожидение т.к. *VACUUM FULL* требует эксклюзивный доступ к таблице и не сочетается ни с одной блокировкой, которую могла бы поставить команда *INSERT* в транзакции.

### Блокировки UPDATE
Открываем 3 сеанса с клиентом и отдаем в них команды внутри открытой транзакции:

**#1**
```
postgres=# BEGIN;
BEGIN
postgres=# UPDATE testTable SET name = name || '1';
UPDATE 3
```
**#2**
```
postgres=# UPDATE testTable SET name = 'name666' WHERE code = 'code1';
```
**#3**
```
postgres=# UPDATE testTable SET code = code || 'x';
```
> [!IMPORTANT]
> В сеансах **#2** и **#3** команды "зависли".

Смотрим блокировки:
```
   locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction | pid |       mode       | granted | fastpath |           waitstart           
---------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+-----+------------------+---------+----------+-------------------------------
 relation      |        5 |    12073 |      |       |            |               |         |       |          | 6/13               |  94 | AccessShareLock  | t       | t        | 
 virtualxid    |          |          |      |       | 6/13       |               |         |       |          | 6/13               |  94 | ExclusiveLock    | t       | t        | 
 relation      |        5 |    16388 |      |       |            |               |         |       |          | 5/12               |  87 | RowExclusiveLock | t       | t        | 
 virtualxid    |          |          |      |       | 5/12       |               |         |       |          | 5/12               |  87 | ExclusiveLock    | t       | t        | 
 relation      |        5 |    16388 |      |       |            |               |         |       |          | 4/30               |  86 | RowExclusiveLock | t       | t        | 
 virtualxid    |          |          |      |       | 4/30       |               |         |       |          | 4/30               |  86 | ExclusiveLock    | t       | t        | 
 relation      |        5 |    16388 |      |       |            |               |         |       |          | 3/46               |  85 | RowExclusiveLock | t       | t        | 
 virtualxid    |          |          |      |       | 3/46       |               |         |       |          | 3/46               |  85 | ExclusiveLock    | t       | t        | 
 tuple         |        5 |    16388 |    0 |     1 |            |               |         |       |          | 5/12               |  87 | ExclusiveLock    | t       | f        | 
 tuple         |        5 |    16388 |    0 |     2 |            |               |         |       |          | 4/30               |  86 | ExclusiveLock    | t       | f        | 
 transactionid |          |          |      |       |            |           742 |         |       |          | 5/12               |  87 | ShareLock        | f       | f        | 2024-11-04 15:10:45.049683+00
 transactionid |          |          |      |       |            |           742 |         |       |          | 4/30               |  86 | ShareLock        | f       | f        | 2024-11-04 15:10:19.277177+00
 transactionid |          |          |      |       |            |           744 |         |       |          | 5/12               |  87 | ExclusiveLock    | t       | f        | 
 transactionid |          |          |      |       |            |           742 |         |       |          | 3/46               |  85 | ExclusiveLock    | t       | f        | 
 transactionid |          |          |      |       |            |           743 |         |       |          | 4/30               |  86 | ExclusiveLock    | t       | f        | 
(15 rows)
```
1. Удобнее всего начать с последних 3-х блокировок. Т.к. типа объекта - *transactionid*, и блокировка - *ExclusiveLock*, очевидно - сервер занял номера транзакций под каждую из трех открытых. Блок успешный, т.к. *granted = t*
2. Далее, строки 11 и 12 мы видим что 2 транзакции блокируются третьей, т.к. *granted = f*. По колонке *virtualtransaction* можно понять что транзакция **3/46** заблокировала выполнение для транзакций **5/12** и **4/30**.
3. Строки 9 и 10 содержат блокировку строки в таблице (*tuple*). По *virtualtransaction* видно что это 2 "опоздавшие" транзакции. Они удерживают блокировки на строках, которые будут обновлять.
4. Строки с  *locktype = virtualxid* это блокировка номера виртуальной транзакции, их 4, 4-я - сеанс в котором смотрим блокировки.
5. Остались строки с *locktype = relation*. Транзакцию с *6/13* можно не смотреть, мы в ней глядим блокировки.Что касается остальных, то видно что каждая транзакция положила на таблицу *RowExclusiveLock* для выполнения UPDATE.
### Взаимоблокировка (deadlock) в трех транзакциях
Открываем 3 сеанса с клиентом и отдаем в них команды внутри открытой транзакции:

**#1**
```
postgres=# BEGIN;
BEGIN
postgres=# UPDATE testTable set name = 'testname' WHERE code = 'code2';
```
**#2**
```
postgres=# BEGIN;
BEGIN
postgres=# UPDATE testTable SET name = 'test2' WHERE code = 'code2';
```
**#3**
```
postgres=# BEGIN;
BEGIN
postgres=# UPDATE testTable SET name = name||'x';
```
> [!IMPORTANT]
> В сеансах **#2** и **#3** команды "зависли".

И теперь чтобы получить состояние *deadlock* в **#1** командуем:
```
postgres=# UPDATE testTable set name = 'testname' WHERE code = 'code1';
ERROR:  deadlock detected
DETAIL:  Process 160 waits for ShareLock on transaction 762; blocked by process 162.
Process 162 waits for ExclusiveLock on tuple (0,17) of relation 16388 of database 5; blocked by process 161.
Process 161 waits for ShareLock on transaction 760; blocked by process 160.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,13) in relation "testtable"
```
> [!IMPORTANT]
> Тут "повезло" и первый же сеанс стал "жертвой". Почему deadlock произошел? Сеанс **#1** заблокировал строку в таблице, сеанс **#2** стал ожидать снятия блокировки по той же строке. Сеанс **#3** вообще ожидает блокировки на всю таблицу.
> Теперь когда сеанс **#1** пытается сам обновить другую строку, происходит авария - чтобы ее обновить нужно дождаться снятия блокировки со всей таблицы сеансом **#3**, который в свою очередь должен дождаться снятия блокировки сеансом **#2** ожидающим **#1**. Выхода нет, СУБД "убивает" транзакцию, чтобы разрешить проблему.

Смотрим лог:
```
2024-11-04 16:11:32.441 UTC [160] ERROR:  deadlock detected
2024-11-04 16:11:32.441 UTC [160] DETAIL:  Process 160 waits for ShareLock on transaction 762; blocked by process 162.
	Process 162 waits for ExclusiveLock on tuple (0,17) of relation 16388 of database 5; blocked by process 161.
	Process 161 waits for ShareLock on transaction 760; blocked by process 160.
	Process 160: UPDATE testTable set name = 'testname' WHERE code = 'code1';
	Process 162: UPDATE testTable SET name = name||'x';
	Process 161: UPDATE testTable SET name = 'test2' WHERE code = 'code2';
```
> [!IMPORTANT]
> Пытаемся разобраться. Возможно конечно у меня так настроен "удачно" журнал, но по записям видно что процесс 162 хочет обновить всю таблицу, которую уже блокирует 161, но при этом сам 161 отдает последнюю по времени команду которая противоречит блокировке из 162. Как мне кажется информации достаточно.

 ### Блокировка на 2 транзакции
 
**Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?**
Насколько я понял - нет. Если открыть транзакцию и в ней попытаться обновить все записи в таблице, на эту таблицу тут же навешивается блокировка, которая запрещает другим транзакциям проводить любое обновление данных.
