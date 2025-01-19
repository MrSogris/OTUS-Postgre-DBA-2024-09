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
CREATE TABLE scenarios (id SERIAL PRIMARY KEY, name VARCHAR(255) NOT NULL, description TEXT NULL, timestampOfCreation DATETIME NOT NULL, createdByUsername VARCHAR(255) NOT NULL, isDeleted BOOLEAN DEFAULT false);
```
Таблица для учета несценарных таблиц:
```
CREATE TABLE nonScenarioTables (tableName VARCHAR(255) PRIMARY KEY);
```
### Организация работы со служебными таблицами
Процедура создания слепка
```
CREATE PROCEDURE addScenario(name NVARCHAR(255), description TEXT, authorUsername NVARCHAR(255))
BEGIN
  INSERT INTO scenarios (name, description, timestampOfCreation, createdByUsername) VALUES (name, description, NOW(), authorUsername);
END
```
По удалению слепков было решено выполнить его асинхронным. Т.е. по факту при запросе на удаление будет лишь ставиться флаг удаления, а настоящее удаление запускать планировщиком во время простоев системы (по ночам). В связи с этим процедуры будет 2.
```
CREATE PROCEDURE deleteScenario(id INT)
BEGIN
  UPDATE scenarios s SET isDeleted = true WHERE s.id = id;
END
```
### Организация работы со справочными таблицами
Основная сложность данной задачи заключается в том, что нам надо не просто динамически получать список существующих таблиц, но и учитывать связи между ними:
1) Удаление данных должно начинаться с тех таблиц, которые не являются родителями в иерархии внешних ключей данных.
2) Вставка данных в новый слепок должна наоборот, начинаться с таблиц которые сами не содержат внешних ключей. При этом еще и надо понимать что вставляя данные с существующего слепка для реализации п.3 требований, нам нужно будет перезаписывать значения внешних ключей, т.к. в новом сценарии они будут другими.
Определимся с базовыми запросами, которые нам понадобятся для реализации процедур создания и удаления слепков.
Получение списка таблиц для копирования:
```
CREATE VIEW scenarioTables AS
SELECT t.relname FROM pg_class t WHERE t.reltype IN (SELECT oid FROM typname = 'rel') AND t.relname NOT IN (SELECT tableName FROM nonScenarioTables);
```
Получение данных по внешним ключам таблиц
```
CREATE VIEW scenarioTableForeignKeys AS
SELECT (select  r.relname from pg_class r where r.oid = c.confrelid) as pkTable,
       a.attname as pkColumn,
       (select r.relname from pg_class r where r.oid = c.conrelid) as fkTable,
       UNNEST((select array_agg(attname) from pg_attribute where attrelid = c.conrelid and array[attnum] <@ c.conkey)) as fkColumn
FROM pg_constraint c join pg_attribute a on c.confrelid=a.attrelid and a.attnum = ANY(confkey)
WHERE c.confrelid = (select oid from pg_class where relname IN (SELECT * FROM scenarioTables))   AND c.confrelid!=c.conrelid;
```
Теперь можем реализовать удаление данных:
```
CREATE PROCEDURE deleteScenarioData()
BEGIN
  BEGIN TRAN;  
  FOR scenarioTableRow in SELECT * FROM scenarios WHERE isDeleted = true LOOP
    CREATE TEMP TABLE clearedTables (tableName VARCHAR(255));
    WHILE EXISTS (SELECT * FROM scenarioTables WHERE tableName NOT IN (SELECT * FROM clearedTables))
      SELECT @deleteQuery = '' FROM scenarioTables WHERE tableName NOT IN (SELECT pkTable FROm scenarioTableForeignKeys WHERE pkTable NOT IN (SELECT tableName FROM clearedTables));
      EXECUTE @deleteQuery;
  END LOOP;
    DROP TABLE clearedTables;
  END LOOP;

  DELETE FROM scenarios Where isDeleted = true;
  COMMIT;
END
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
