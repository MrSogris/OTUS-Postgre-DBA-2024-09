# Задание 18
## Хранимые функции и процедуры часть 3
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 15.8
### Создание тестовой БД
Для начала создаем тестовую БД:
```
-- ДЗ тема: триггеры, поддержка заполнения витрин

DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;

SET search_path = pract_functions, publ

-- товары:
CREATE TABLE goods
(
    goods_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(1, 'Спички хозайственные', .50),
		(2, 'Автомобиль Ferrari FXX K', 185000000.01);

-- Продажи
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
-- с увеличением объёма данных отчет стал создаваться медленно
-- Принято решение денормализовать БД, создать таблицу
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2)NOT NULL
);
```
Заполнение таблицы **good_sum_mart**
```
-- отчет:
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
```
### Создать триггер для поддержки витрины в актуальном состоянии
Анализируя скрипт который ранее составлял отчет, видим, что он собирает данные с 2-х таблиц, из чего следует что на обе таблицы придется добавлять триггеры.
Чтобы не копировать код расчета, составим функцию:
```
CREATE FUNCTION updateReportSum(goodId integer) RETURNS void AS $$
DECLARE goodSum numeric(16, 2);
DECLARE goodName varchar(63);
BEGIN
SELECT sum(G.good_price * S.sales_qty) INTO goodSum
FROM goods G
INNER JOIN sales S ON S.good_id = G.goods_id AND G.goods_id = goodId
GROUP BY G.good_name;
SELECT good_name INTO goodName FROM goods WHERE goods_id = goodId;
IF goodSum IS NULL THEN
  DELETE FROM good_sum_mart WHERE good_name = good_name;
ELSEIF EXISTS (SELECT * FROM good_sum_mart WHERE good_name IN (SELECT good_name FROM goods WHERE goods_id = goodId)) THEN
  UPDATE good_sum_mart SET sum_sale = goodSum WHERE good_name = goodName;
ELSE
  INSERT INTO good_sum_mart VALUES (goodName, goodSum);
END IF;
END $$
LANGUAGE PLPGSQL;
```
Далее для поддержания в актуальном состоянии отчетной таблицы, создаем 2 триггера
```
CREATE FUNCTION onSalesChange() RETURNS trigger AS $sales$
BEGIN
  IF NEW IS NULL THEN
    SELECT updateReportSum(NEW.good_id);
  ELSE
    SELECT updateReportSum(OLD.good_id);
  END IF;
END;
$sales$ LANGUAGE PLPGSQL;
CREATE TRIGGER sales_OnChange AFTER INSERT OR UPDATE OR DELETE ON sales
FOR EACH ROW
EXECUTE FUNCTION onSalesChange();

CREATE TRIGGER goods_OnChange AFTER INSERT OR UPDATE OR DELETE ON goods
FOR EACH ROW
BEGIN
  IF :new IS NULL THEN
    SELECT updateReportSum(:old.goods_id);
  ELSE
    SELECT updateReportSum(:new.goods_id);
  END IF;
END;
```
### Преимущества схемы с триггером
1) Ниже время доступа к отчету (его не нужно каждый раз собирать заново)
2) Разравнивается нагрузка на сервер (если отчет тяжелый, он будет вызывать пики нагрузки, а с триггером нагрузка будет распределена по всем моментам изменения данных)
3) Ничто не мешает теперь проиндексировать наш отчет, чтобы он работал еще быстрее
