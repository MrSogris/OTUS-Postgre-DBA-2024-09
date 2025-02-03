# Организация создания, копирования и удаления слепков данных для моделирования по принципу "что-если".
## Описание проблемы, цели и задачи.
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
Естественно что пользователь при начале работы с системой выбирает в UI, с каким слепком он будет работать в данный момент. Когда редактируется выбранный слепок, другие никак не меняются.
Еще один важный здесь момент - это существование т.н. несценарных таблиц, т.е. таблиц которые являются общими для всех слепков. Как правило это сама таблица слепков и еще ряд таблиц которые по сути являются константами - например справочник регионов РФ, и т.п. Конечно, можно все таблицы сделать сценарными, но это не всегда удобно как с точки зрения производительности, так и актуализации несценарных данных.
На данный момент наши системы ориентированы на работу с СУБД MS SQL Server. Однако в связи со сложившейся ситуацией и невозможностью в адекватные сроки и затраты расширять парк инсталляций данной СУБД, возникает цель - реализовать хранилище слепков на СУБД Postgre SQL.
Для этого необходимо решить следующие задачи:
1) Организовать хранение данных о всех имеющихся слепках (названия и другая метаинформация необходимая для навигации по ним).
2) Реализовать алгоритм создания новых слепков, с возможность как создать пустой слепок, так и использовать существующий как шаблон, из которого будут скопированы все данные (копирование).
3) Реализовать алгоритм удаления слепков.
4) Обеспечить неизменность служебного кода независимо от структуры и количества сценарных справочников.

## Начальные соглашения и условности
1) Считаем, что поле со ссылкой на сценарий всегда называется **scenarioId**. Конечно, можно называть его как угодно и получать эту информацию из метаданных СУБД, но унификация названий - банально удобнее в разработке в т.ч. и таблиц. Не придется постоянно искать это поле в коде.
2) Таблицы, являющиеся источниками для внешних ключей, содержат суррогатный ключ (поле **id**). Классическое решение, хотя опять таки это все можно вычислять "на лету".

## Архитектура решения
В первую очередь, необходимо будет создать 2 схемы - под служебные объекты и под собственно данные. Схему с данными будем наполнять уже рабочими таблицами по запросу заказчика, служебная - остается неизменной (п.4 задач).
В результате анализа требований, стало ясно, что в служебной схеме потребуется создать следующие объекты:
1) Таблица слепков (для п.1 задач)
2) Процедура создания слепка (п.2 задач)
3) Процедура удаления слепка (п.3 задач)
Кроме того, хотя этого и нет в задачах, было решено реализовать схему с отложенным удалением слепков. Связано это с тем, что на практике, удаление даже 1 слепка физически из всех таблиц, может очень сильно нагружать систему, не говоря уже о случае удаления нескольких сразу. Таким образом, слепок будет лишь помечаться как "удаленный", а настоящую очистку данных можно будет проводить, например, по расписанию в безопасное для этой операции время.
В связи с этим, процеура из п.3 по факту распадается на 2 процедуры.

## Проблемы на этапе проектирования
1) Т.к. нельзя хардкодить структуру данных, все таблицы и колонки и т.д. нужно получать на лету.
2) Необходимо обеспечить корректность копирования данных с внешними ключами, т.к. у одной и той же по факту сущности в данной парадигме может быть множество копий с разными id, и при копировании этот id нужно будет "подменять".
3) Последствие п.2 - удалять и копировать данные "разом" невозможно. Удалять сценарии можно только начиная с ветвей дерева зависимости и до корня, а копировать - в обратную сторону.

## Детализация структуры служебных данных
Таблица, в которой будем хранить информацию о слепках:
```
CREATE TABLE scenarios (id SERIAL PRIMARY KEY, name VARCHAR(255) NOT NULL, description TEXT NULL, timestampOfCreation TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP, createdByUsername VARCHAR(255) NOT NULL, isDeleted BOOLEAN DEFAULT false);
```
Поле *isDeleted* необходимо для пометки об удалении сценария при очистке данных.
Далее, для упрощения кодирования алгоритмов, заведем списки сценарных и несценарных таблиц:
Таблица для учета несценарных таблиц:
```
CREATE TABLE nonScenarioTables (tableName VARCHAR(255) PRIMARY KEY);
CREATE VIEW scenarioTables AS SELECT table_name AS tableName FROM information_schema.tables WHERE table_schema = 'schema_with_data' AND table_type = 'BASE TABLE'  AND table_name NOT IN (SELECT tableName FROM nonScenarioTables);
```
Представление для получения данных по внешним ключам таблиц (отсеиваем несценарные таблицы, т.к. они не участвуют в копировании).
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
where tco.constraint_type = 'FOREIGN KEY' AND kcu.table_schema = 'schema_with_data' AND rel_tco.table_name IN (SELECT tableName FROM scenarioTables);
```
## Организация работы со слепками с помощью процедур
Процедура создания слепка, создает новую запись в таблице сценариев, а также копирует данные сценария с учетом замечаний выше.
```
CREATE PROCEDURE addScenario(name VARCHAR(255), description TEXT, createdByUsername VARCHAR(255), createdScenarioId OUT INT, sourceScenarioId INT DEFAULT NULL) AS $$
DECLARE insertedId INTEGER;
DECLARE currentTableName VARCHAR(500);
DECLARE tableFields VARCHAR(1000);
DECLARE insertQuery VARCHAR(2000);
DECLARE keyMapQuery VARCHAR(1000);
DECLARE fkUpdateQuery VARCHAR(2000);
DECLARE fkRow scenarioTableForeignKeys%rowtype;
BEGIN
  INSERT INTO scenarios (name, description, createdByUsername) VALUES (name, description, createdByUsername) RETURNING id INTO insertedId;

  IF sourceScenarioId IS NOT NULL THEN
    CREATE TEMP TABLE copiedTables (tableName VARCHAR(255));
    CREATE TEMP TABLE keysMapping (tableName VARCHAR(255), oldId INTEGER, newId INTEGER);
    WHILE EXISTS (SELECT 1 FROM scenarioTables WHERE tableName NOT IN (SELECT tableName FROM copiedTables)) LOOP
      SELECT tableName INTO currentTableName FROM scenarioTables WHERE NOT EXISTS (SELECT 1 FROM scenarioTableForeignKeys WHERE fkTable = tableName AND pkTable NOT IN (SELECT tableName FROM copiedTables)) AND tableName NOT IN (SELECT tableName FROM copiedTables);
      SELECT '"'||string_agg(column_name, '", "')||'"' INTO tableFields FROM information_schema.columns WHERE table_schema = 'data' AND table_name = currentTableName AND LOWER(column_name) NOT IN ('id', 'scenarioid');
      SELECT string_agg('t."'||column_name||'" = i."'||column_name||'"', ' AND ') INTO keyMapQuery FROM information_schema.columns WHERE table_schema = 'data' AND table_name = currentTableName AND LOWER(column_name) NOT IN ('id', 'scenarioid');
      
      IF EXISTS (SELECT 1 FROM scenarioTableForeignKeys WHERE pkTable = currentTableName) THEN        
        insertQuery = 'WITH insertedData AS (INSERT INTO data."'||currentTableName||'"(scenarioId, '||tableFields||') SELECT '||insertedId::varchar(255)||', '||tableFields||' FROM data."'||currentTableName||'" WHERE scenarioId = '||sourceScenarioId::varchar(255)||' RETURNING *) INSERT INTO keysMapping SELECT '''||currentTableName||''', t.id, i.id FROM insertedData i JOIN data."'||currentTableName||'" t ON t.scenarioId='||sourceScenarioId::varchar(255)||' AND '||keyMapQuery||';';
      ELSE
        insertQuery = 'INSERT INTO data."'||currentTableName||'"(scenarioId, '||tableFields||') SELECT '||insertedId::varchar(255)||', '||tableFields||' FROM data."'||currentTableName||'" WHERE scenarioId = '||sourceScenarioId::varchar(255)||';';
      END IF;

      EXECUTE insertQuery;
      RAISE NOTICE 'query = %', insertQuery;

      FOR fkRow IN SELECT * FROM scenarioTableForeignKeys WHERE fkTable = currentTableName LOOP
        fkUpdateQuery = 'UPDATE data."'||currentTableName||'" d SET "'||fkRow.fkColumn||'" = t.newId FROM keysMapping t WHERE t.oldId = d."'||fkRow.fkColumn||'" AND t.tableName = '''||fkRow.pkTable||''' AND d.scenarioId='||insertedId::varchar(255)||';';

        EXECUTE fkUpdateQuery;
        
        RAISE NOTICE 'query2 = %', fkUpdateQuery;
      END LOOP;

      INSERT INTO copiedTables(tableName) VALUES (currentTableName);
    END LOOP;
    DROP TABLE keysMapping;
    DROP TABLE copiedTables;
  END IF;

  createdScenarioId = insertedId;
END
$$
LANGUAGE PLPGSQL;
```
Обновление информации о слепке по сути касается только его имени и описания
```
CREATE PROCEDURE updateScenario(updatedScenarioId INTEGER, newName VARCHAR(255), newDescription TEXT) AS $$
BEGIN
       UPDATE scenarios SET name = newName, description = newDescription WHERE id = updatedScenarioId;
END
$$
LANGUAGE PLPGSQL;
```
По удалению слепков было решено выполнить его асинхронным. Т.е. по факту при запросе на удаление будет лишь ставиться флаг удаления, а настоящее удаление запускать планировщиком во время простоев системы (по ночам). В связи с этим процедуры будет 2.
```
CREATE PROCEDURE deleteScenario(deletedScenarioId INTEGER) AS $$
BEGIN
  UPDATE scenarios SET isDeleted = true WHERE id = deletedScenarioId;
END
$$
LANGUAGE PLPGSQL;
```
Собственно удаление сценариев. Также учитывается нужный порядок удаления.
```
CREATE PROCEDURE deleteScenarioData() AS $$
DECLARE currentTableName VARCHAR(500);
DECLARE deleteQuery VARCHAR(2000);
DECLARE scenarioTableRow scenarios%rowtype;
BEGIN  
  FOR scenarioTableRow in SELECT * FROM scenarios WHERE isDeleted = true LOOP
    CREATE TEMP TABLE clearedTables (tableName VARCHAR(255));
    WHILE EXISTS (SELECT 1 FROM scenarioTables WHERE tableName NOT IN (SELECT tableName FROM clearedTables)) LOOP
      SELECT tableName INTO currentTableName FROM scenarioTables WHERE NOT EXISTS (SELECT 1 FROM scenarioTableForeignKeys WHERE pkTable = tableName AND fkTable NOT IN (SELECT tableName FROM clearedTables)) AND tableName NOT IN (SELECT tableName FROM clearedTables);
      
      deleteQuery = 'DELETE FROM data."'||currentTableName||'" WHERE scenarioId='||scenarioTableRow.id::varchar(255)||';'; 
      
      EXECUTE deleteQuery;
      INSERT INTO clearedTables(tableName) VALUES(currentTableName);
    END LOOP;
    DROP TABLE clearedTables;    
    DELETE FROM scenarios Where id = scenarioTableRow.id;
  END LOOP;
END
$$
LANGUAGE PLPGSQL;
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
## Выводы и замечания
На тестовом наборе данных работа со слепками не вызывает никаких затруднений. В принципе, функционал ничем не уступает тому, что был в MS SQL.
Однако, хотелось бы отметить ряд моментов которые хотелось бы улучшить в будущем.
1) Сопоставление нового и старого ключа сущности в сценариях сделано путем прямого сравнения свех атрибутов, что не очень производительно. Обычно в схеме с суррогатным ключом всегда присутствует уникальный ключ, по которому можно было бы сопоставлять данные. Это вполне можно реализовать, получив его из метаданных СУБД.
2) Реализовать возможность отказаться от учета несценарных таблиц, а получать их из метаданных СУБД.
3) Добавить вакуумирование при очистке сценариев, т.к. из-за особенностей Postgre, если этого не делать, таблицы могут "распухнуть" от данных. Ну или как-то синхронизироваться с DBA по обслуживанию.
