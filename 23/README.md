# Задание 23
## Сбор и использование статистики 
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 15.8
### Тестовая БД
Для выполнения задания нам надо в идеале получить 3 таблицы, оформляем как пример, базу данных - "Рабочие проекты"
```
CREATE TABLE managers(id SERIAL PRIMARY KEY, name VARCHAR(255) NOT NULL);
INSERT INTO managers (name) VALUES ('John'), ('Paul'), ('George'), ('Richard');
CREATE TABLE projects(id SERIAL PRIMARY KEY, name VARCHAR(255) NOT NULL, managerId INTEGER REFERENCES managers(id));
INSERT INTO projects (name, managerId) VALUES ('lostProject', NULL), ('futureProject', NULL);
INSERT INTO projects (name, managerId)
SELECT CASE WHEN name = 'John' THEN 'Please pleas me' WHEN name = 'Paul' THEN 'Abbey Road' WHEN name = 'George' THEN 'Get back' END AS name, id AS managerId FROM managers WHERE name <> 'Richard';
CREATE TABLE workdays(workday DATE PRIMARY KEY);
INSERT INTO workdays(workday)
SELECT to_date('2025-01-01', 'YYYY-MM-DD') + s.a As workday FROM generate_series (0, 364) AS s(a) WHERE EXTRACT (dow FROM to_date('2025-01-01', 'YYYY-MM-DD') + s.a) NOT IN (0, 6);
```
### Прямое соединение
Вывод проектов у которых назначены менеджеры
```
postgres=# SELECT m.name AS managerName, p.name AS projectName FROM managers m JOIN projects p ON m.id = p.managerId;
 managername |   projectname   
-------------+-----------------
 John        | Please pleas me
 Paul        | Abbey Road
 George      | Get back
(3 rows)
```
### Правоороннее соединение
Вывод всех проектов независимо от наличия менеджера
```
postgres=# SELECT m.name AS managerName, p.name AS projectName FROM managers m RIGHT JOIN projects p ON m.id = p.managerId;
 managername |   projectname   
-------------+-----------------
             | lostProject
             | futureProject
 John        | Please pleas me
 Paul        | Abbey Road
 George      | Get back
(5 rows)
```
## Полное соединение
Сводная таблица проектов и менеджеров
```
postgres=# SELECT m.name AS managerName, p.name AS projectName FROM managers m FULL JOIN projects p ON m.id = p.managerId;
 managername |   projectname   
-------------+-----------------
             | lostProject
             | futureProject
 John        | Please pleas me
 Paul        | Abbey Road
 George      | Get back
 Richard     | 
(6 rows)
```
### Кросс-соединение
Посчитаем сколько человеко-часов у нас есть
```
postgres=# SELECT COUNT(*) * 8 AS employeeHours FROM workdays w CROSS JOIN managers m;
 employeehours 
---------------
          8352
(1 row)
```
### Разные типы в одном запросе
Выведем полный отчет по рабочим часам в проектах у которых есть менеджеры
```
postgres=# SELECT m.name, p.name, w.workday FROM managers m JOIN projects p ON m.id = p.managerId CROSS JOIN workdays w ORDER BY workday LIMIT 10;
  name  |      name       |  workday   
--------+-----------------+------------
 John   | Please pleas me | 2025-01-01
 Paul   | Abbey Road      | 2025-01-01
 George | Get back        | 2025-01-01
 John   | Please pleas me | 2025-01-02
 Paul   | Abbey Road      | 2025-01-02
 George | Get back        | 2025-01-02
 John   | Please pleas me | 2025-01-03
 Paul   | Abbey Road      | 2025-01-03
 George | Get back        | 2025-01-03
 John   | Please pleas me | 2025-01-06
(10 rows)
```
> [!NOTE]
> Ограничил тут вывод, чтобы не раздувать отчет.
