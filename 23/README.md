Для выполнения задания нам надо в идеале получить 3 таблицы, оформляем:
```
CREATE TABLE carClasses(classCode VARCHAR(2) NOT NULL PRIMARY KEY, description NVARCHAR(2000));
CREATE TABLE bmwSeries(serie INTEGER, carClassesClassCode VARCHAR(2) NOT NULL REFERENCES carClasses(classCode));
CREATE TABLE bmwEngineTypes(typeCode VARCHAR(2) NOT NULL PRIMARY KEY);
CREATE TABLE bmwEngines(id SERIAL NOT NULL PRIMARY KEY, displacement NUMERIC (2, 1), bmwEngineTypesTypeCode VARCHAR(2) REFERENCES bmwEngineTypes(typeCode));
```
### Прямое соединение
Выведем серии автомобилей и их принадлежность к классам
```
SELECT bs.serie, cc.classCode FROM carClasses cc JOIN bmwSeries bs ON cc.classCode = bs.carClassesClassCode;
```
### Левостороннее соединение
Определим, какие классы автомобилей концерн пока еще не закрыл
```
SELECT classCode FROM carClasses cc LEFT JOIN bmwSeries bs ON cc.classCode = bs.carClassesClassCode WHERE bs.serie IS NULL;
```
### Кросс-соединение
Определяем все возможные сейчас наименования моделей c ДВС
```
SELECT serie::varchar(1)|(displacement*10)::varchar(2)|bet.typeCode FROM bmwSeries bs CROSS JOIN bmwEngines be JOIN bmwEngineTypes bet ON bet.typeCode = be.bmwEngineTypesTypeCode WHERE be.bmwEngineTypesTypeCode IS NOT NULL;
```
## Полное соединение
Выведем сводную таблицу по ДВС и моделям
```
SELECT bs.serie, bet.typeCode, be.displacement FROM bmwSeries bs CROSS JOIN bmwEngines be FULL JOIN bmwEngineTypes bet ON bet.typeCode = be.bmwEngineTypesTypeCode;
```
