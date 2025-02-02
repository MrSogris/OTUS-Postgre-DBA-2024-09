# Организация создания, копирования и удаления слепков данных для моделирования по принципу "что-если".
## Описание задачи и цели.
Достаточно часто в рамках своей работы приходится сталкиваться с аналитическими системами, которые предполагают проведение тех или иных расчетов на базе готового набора данных. 
Как правило, это сложные расчеты, занимающие длительное время. Если результат не устраивает, приходится редактировать исходный набор данных, снова и снова, при этом без уверенности что внесенная правка гарантирует нужный результат.
Конечно же, можно просто отменить все правки, но иногда их может быть очень много, да и сам по себе процесс отката может быть сложным если делать его логически (а не физически, скажем из бэкапа).
В связи с этим в наших локальных системах используется подход "слепков" данных. По сути это такие же бэкапы, которые однако хранятся в таблицах, как живые данные и к которым можно вернуться в любой момент времени.
Например, пусть у нас есть некий базовый справочник:
```
CREATE TABLE someTable(id SERIAL PRIMARY KEY, someName VARCHAR(255), someParameter NUMERIC(10, 3));
```
Для того чтобы организовать хранение слепков, мы добавляем в него поле с id слепка, указывающим на справочник слепков данных:
```
CREATE TABLE someTable(id SERIAL PRIMARY KEY, scenarioId int REFERENCES scenarios, someName VARCHAR(255), someParameter NUMERIC(10, 3));
```
где *scenarios* - это специальная таблица, содержащая информацию о всех существующих в данный момент слепках данных.
Естественно что пользователь при начале работы с системой выбирает, с каким слепком он будет работать в данный момент. Когда редактируется выбранный слепок, другие никак не меняются.
Еще один важный здесь момент - это существование т.н. несценарных таблиц, т.е. таблиц которые являются общими для всех слепков. Как правило это сама таблица слепков и еще ряд таблиц которые по сути являются константами - например справочник регионов РФ, и т.п.
На данный момент наши системы ориентированы на работу с СУБД MS SQL Server. Однако в связи со сложившейся ситуацией и невозможностью в адекватные сроки и затраты расширять парк инсталляций данной СУБД, возникает потребность реализовать хранилище слепков на СУБД Postgre SQL.
Хранилище должно реализовывать следующие функциональные требования:
1) Предоставлять данные о всех имеющихся слепках (названия и другая метаинформация необходимая для навигации по ним).
2) Создание новых слепков для заполнения (т.е. изначально пустых).
3) Использование существующего слепка для создания нового (т.е. новый слепок будет содержать данные, скопированные из выбранного другого).
4) Удаление уже ненужных слепков (т.к. они могут занимать много места).
5) Создание списков сценарных таблиц должно быть автоматическим, т.к. иначе могли бы потребоваться программные модификации при изменении списка справочников, что не совсем удобно, т.к. справочники могут быть настраиваемыми.
6) Исключение несценарных таблиц из списка рабочих.

## Архитектура решения
В результате анализа требований, стало ясно, что в будущей БД потребуется создать следующие служебные объекты:
1) Таблица слепков (для п.1 требований)
2) Процедура создания слепка (п.2, п.3, п.5 требований)
3) Процедура удаления слепка (п.4 требований)
4) Таблица со списком несценарных таблиц (п.6 требований)
Остальные объекты будем создавать уже как тестовое наполнение системы.
Сразу отмечу, что не рассматриваем случай превращения таблицы из несценарной в сценарную и обратно, т.к. это очень серьезная модификация которая скорее всего потребует перенастройку UI и т.д. - 100% вмешательство разработчиков.
## Реализация
Таблица, в которой будем хранить информацию о слепках:
```
CREATE TABLE scenarios (id SERIAL PRIMARY KEY, name VARCHAR(255) NOT NULL, description TEXT NULL, timestampOfCreation TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP, createdByUsername VARCHAR(255) NOT NULL, isDeleted BOOLEAN DEFAULT false);
```
Таблица для учета несценарных таблиц:
```
CREATE TABLE nonScenarioTables (tableName VARCHAR(255) PRIMARY KEY);
```
Для того, чтобы не переписывать каждый раз при добавлении таблиц все скрипты, будем автоматически собирать о них информацию.
Получение списка таблиц для копирования:
```
CREATE VIEW scenarioTables AS
SELECT table_name AS tableName FROM information_schema.tables WHERE table_schema = 'some_schema' AND table_type = 'BASE TABLE'  AND table_name NOT IN (SELECT tableName FROM nonScenarioTables);
```
Получение данных по внешним ключам таблиц
```
CREATE VIEW scenarioTableForeignKeys AS
select kcu.table_name as fkTable,
       rel_tco.table_name as pkTable,
       kcupk.column_name as pkColumn,
       kcu.column_name as fkColumn
from information_schema.table_constraints tco
join information_schema.key_column_usage kcu
          on tco.constraint_schema = kcu.constraint_schema
          and tco.constraint_name = kcu.constraint_name
join information_schema.referential_constraints rco
          on tco.constraint_schema = rco.constraint_schema
          and tco.constraint_name = rco.constraint_name
join information_schema.table_constraints rel_tco
          on rco.unique_constraint_schema = rel_tco.constraint_schema
          and rco.unique_constraint_name = rel_tco.constraint_name
join information_schema.key_column_usage kcupk
          on rel_tco.constraint_schema = kcupk.constraint_schema
          and rel_tco.constraint_name = kcupk.constraint_name
where tco.constraint_type = 'FOREIGN KEY' AND kcu.table_schema = 'some_schema';
```
### Организация работы со служебными таблицами
Процедура создания слепка
```
CREATE PROCEDURE addScenario(name NVARCHAR(255), description TEXT, createdByUsername NVARCHAR(255), createdScenarioId OUT INT, sourceScenarioId INT DEFAULT NULL) AS $$
DECLARE inserted_id INTEGER;
BEGIN
  INSERT INTO scenarios (name, description, createdByUsername) VALUES (name, description, createdByUsername) RETURNING id INTO inserted_id;

  IF sourceScenarioId IS NOT NULL THEN
    
  END IF

  createdScenarioId = insertedId;
END
$$
LANGUAGE PLPGSQL;
```
Обновление информации о слепке по сути касается только его имени и описания
```
CREATE PROCEDURE utils.updateScenario(updatedScenarioId INTEGER, newName VARCHAR(255), newDescription TEXT) AS $$
BEGIN
       UPDATE utils.scenarios SET name = newName, description = newDescription WHERE id = updatedScenarioId;
END
$$
LANGUAGE PLPGSQL;
```
По удалению слепков было решено выполнить его асинхронным. Т.е. по факту при запросе на удаление будет лишь ставиться флаг удаления, а настоящее удаление запускать планировщиком во время простоев системы (по ночам). В связи с этим процедуры будет 2.
```
CREATE PROCEDURE utils.deleteScenario(deletedScenarioId INTEGER) AS $$
BEGIN
  UPDATE utils.scenarios SET isDeleted = true WHERE id = deletedScenarioId;
END
$$
LANGUAGE PLPGSQL;
```
### Организация работы со справочными таблицами
Основная сложность данной задачи заключается в том, что нам надо не просто динамически получать список существующих таблиц, но и учитывать связи между ними:
1) Удаление данных должно начинаться с тех таблиц, которые не являются родителями в иерархии внешних ключей данных.
2) Вставка данных в новый слепок должна наоборот, начинаться с таблиц которые сами не содержат внешних ключей. При этом еще и надо понимать что вставляя данные с существующего слепка для реализации п.3 требований, нам нужно будет перезаписывать значения внешних ключей, т.к. в новом сценарии они будут другими.
```
CREATE PROCEDURE utils.deleteScenarioData() AS $$
DECLARE currentTableName VARCHAR(500);
DECLARE deleteQuery VARCHAR(2000);
DECLARE scenarioTableRow utils.scenarios%rowtype;
BEGIN  
  FOR scenarioTableRow in SELECT * FROM utils.scenarios WHERE isDeleted = true LOOP
    CREATE TEMP TABLE clearedTables (tableName VARCHAR(255));
    WHILE EXISTS (SELECT 1 FROM utils.scenarioTables WHERE tableName NOT IN (SELECT tableName FROM clearedTables)) LOOP
      SELECT tableName INTO currentTableName FROM utils.scenarioTables WHERE NOT EXISTS (SELECT 1 FROM utils.scenarioTableForeignKeys WHERE pkTable = tableName AND fkTable NOT IN (SELECT tableName FROM clearedTables)) AND tableName NOT IN (SELECT tableName FROM clearedTables);
      
      deleteQuery = 'DELETE FROM data."'||currentTableName||'" WHERE scenarioId='||scenarioTableRow.id::varchar(255)||';'; 
      
      EXECUTE deleteQuery;
      INSERT INTO clearedTables(tableName) VALUES(currentTableName);
    END LOOP;
    DROP TABLE clearedTables;    
    DELETE FROM utils.scenarios Where id = scenarioTableRow.id;
  END LOOP;
END
$$
LANGUAGE PLPGSQL;
```
Копирование данных слепка
```
CREATE PROCEDURE copyScenarioData(sourceScenarioId int, targetScenarioId int)
BEGIN
  CREATE TEMP TABLE clearedTables (tableName VARCHAR(255));
    WHILE EXISTS (SELECT * FROM scenarioTables WHERE tableName NOT IN (SELECT * FROM clearedTables))
      @deleteQuery = '';
      EXECUTE @deleteQuery;
  END LOOP;
    DROP TABLE clearedTables;
END
```
### Тестовый набор справочников
Для теста создадим достаточно типичную легенду.
> [!NOTE]
> Федеральная группа компаний ООО "Вектор" занимается производством неких сыпучих или жидких продуктов в промышленных масштабах и продает их оптовым клиентам. Заводы раскиданы по всей стране, как и покупатели. Необходимо обеспечить возможность хранить и анализировать различные варианты объемов производства и цен на продукты в зависимости от предполагаемых затрат на транспортировку по плечу *завод - покупатель* и предполагаемых бюджетов покупателя на выкуп.

Основные сущности будут:
1) Города
2) Заводы
3) Клиенты
4) Продукты
5) Объемы производства
6) Логистика между городами и затраты на нее

Для размещения рабочих данных создадим схему *data*.
Скрипт для быстрого создания тестовой структуры и данных в ней:
```
CREATE DATABASE work;
\c work
CREATE SCHEMA data;
CREATE TABLE data.cities(id SERIAL PRIMARY KEY, name VARCHAR(255) NOT NULL, UNIQUE(name));
CREATE TABLE data.plants (id SERIAL PRIMARY KEY, name VARCHAR(255) NOT NULL, cityId INT NOT NULL REFERENCES data.cities(id));
CREATE TABLE data.products (id SERIAL PRIMARY KEY, scenarioId INTEGER NOT NULL, name VARCHAR(255) NOT NULL, price NUMERIC(6, 2) NOT NULL, UNIQUE(scenarioId, name));
CREATE TABLE data.production (scenarioId INTEGER NOT NULL, plantId INTEGER NOT NULL REFERENCES data.plants(id), productId INTEGER NOT NULL REFERENCES data.products(id), volume NUMERIC(6, 2) NOT NULL, PRIMARY KEY (scenarioId, plantId, productId));
CREATE TABLE data.buyers (id SERIAL PRIMARY KEY, scenarioId INTEGER NOT NULL, name VARCHAR(255), cityId INTEGER NOT NULL REFERENCES data.cities(id), budget NUMERIC (10, 2) NOT NULL, UNIQUE(scenarioId, name));
CREATE TABLE data.logistics (scenarioId INTEGER NOT NULL, cityFromId INTEGER NOT NULL REFERENCES data.cities(id), cityToId INTEGER NOT NULL REFERENCES data.cities(id), costPerUnit NUMERIC(6, 2) NOT NULL, PRIMARY KEY (scenarioId, cityFromId, cityToId));
```
Управляющие объекты разместим в схеме *utils*
```
\c work
CREATE SCHEMA utils;
--Далее используем определения объектов сделанные ранее, но с учетом схемы.
-- Определяем несценарные таблицы
INSERT INTO utils.nonScenarioTables (tableName) VALUES ('cities'), ('plants');
--Создаем начальный сценарий и заполним его данными
CALL utils.addScenario('initial', 'initial scenario', 'Vasya', 0, NULL);
INSERT INTO data.cities(name) VALUES ('Moscow'), ('Sankt-Peterburg'), ('Kaliningrad'), ('Samara'), ('Ekaterinburg'), ('Omsk'), ('Tomsk'), ('Kemerovo'), ('Novosibirsk'), ('Krasnoyarsk'), ('Irkutsk'), ('Khabarovsk'), ('Vladivostok');
INSERT INTO data.plants(name, cityId) SELECT md5(random()::text) as name, c.id FROM generate_series(1, 2) t1 CROSS JOIN data.cities c;
INSERT INTO data.products(scenarioId, name, price) SELECT id, md5(random()::text) as name, random() * 100 as price FROM generate_series(1, 10) CROSS JOIN utils.scenarios;
INSERT INTO data.production(scenarioId, plantId, productId, volume) SELECT s.id, p.id, pr.id, random() *  1000 as volume FROM utils.scenarios s CROSS JOIN data.plants p CROSS JOIN data.products pr;
INSERT INTO data.buyers(scenarioId, name, cityId, budget) SELECT s.id, md5(random()::text) as name, c.id, random() * 10000 FROM generate_series(1, 5) t1 CROSS JOIN data.cities c CROSS JOIN utils.scenarios s;
INSERT INTO data.logistics(scenarioId, cityFromId, cityToId, costPerUnit) SELECT s.id, c1.id, c2.id, random() * 10 FROM utils.scenarios s CROSS JOIN data.cities c1 JOIN data.cities c2 ON c1.id <> c2.id;
```
### Тестовые операции
Создаем новый пустой сценарий **Test 2**
```
work=# CALL utils.addScenario('Test2', NULL, 'Ivanov Ivan', 0, NULL);
 createdscenarioid 
-------------------
                 4
(1 row)

work=# SELECT * FROM utils.scenarios;
 id |  name   |   description    |    timestampofcreation     | createdbyusername | isdeleted 
----+---------+------------------+----------------------------+-------------------+-----------
  3 | initial | initial scenario | 2025-02-02 15:16:53.39401  | Vasya             | f
  4 | Test2   |                  | 2025-02-02 15:43:32.703312 | Ivanov Ivan       | f
(2 rows)
```
А теперь попробуем создать новый сценарий, но на базе уже нашего сгенерированного и полного данных:
```
CALL utils.addScenario('initial copy', 'copied from initial', 'Vasya', 0, 3);
```
