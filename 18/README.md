# Задание 18
## Хранимые функции и процедуры часть 3
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен Docker desktop, а дальнейшие манипуляции производились над образом *postgres* 15.8
### Создание тестовой БД
Для начала создаем тестовую БД:
```
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

-- Продажи
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (goods_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);

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
Далее для поддержания в актуальном состоянии отчетной таблицы, создаем триггер
```
CREATE OR REPLACE FUNCTION onSalesChange() RETURNS TRIGGER AS $$
DECLARE deletedCount integer;
BEGIN
    WITH deleted AS (
        DELETE FROM good_sum_mart WHERE sum_sale = 0 AND TG_OP = 'DELETE' AND good_name IN (SELECT good_name FROM goods WHERE goods_id = OLD.good_id) RETURNING good_name
    )
    SELECT COUNT(*) INTO deletedCount FROM deleted;

    IF deletedCount = 0 THEN
        MERGE INTO good_sum_mart m
        USING (
            SELECT good_name AS goodName, 
                SUM(good_price * COALESCE(sales_qty, 0)) AS actualGoodSum 
            FROM goods 
                LEFT JOIN sales ON good_id = goods_id
            WHERE goods_id = CASE WHEN TG_OP = 'DELETE' THEN OLD.good_id ELSE NEW.good_id END
            GROUP BY good_name
        ) AS t
        ON m.good_name = t.goodName
        WHEN MATCHED THEN
            UPDATE SET sum_sale = actualGoodSum
        WHEN NOT MATCHED THEN
            INSERT (good_name, sum_sale)
            VALUES (goodName, actualGoodSum);
    END IF;
    
  RETURN NULL;
END $$
LANGUAGE PLPGSQL;

CREATE TRIGGER sales_OnChange AFTER INSERT OR UPDATE OR DELETE ON sales
FOR EACH ROW
EXECUTE PROCEDURE onSalesChange();
```
Начинаем заполнение данными, при этом проверяем витрину, чтобы убедиться что наши триггеры корректно работают:
```
postgres=# INSERT INTO goods (goods_id, good_name, good_price) VALUES (1, 'Спички хозайственные', .50), (2, 'Автомобиль Ferrari FXX K', 185000000.01);
INSERT 0 2
postgres=# INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);
INSERT 0 4
postgres=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale   
--------------------------+--------------
 Спички хозайственные     |        65.50
 Автомобиль Ferrari FXX K | 185000000.01
(2 rows)
```
> [!NOTE]
> Начальное заполнение прошло правильно, и отчет сформирован.

Продадим еще одну Ferrari
```
postgres=# INSERT INTO sales (good_id, sales_qty) VALUES(2, 1);
INSERT 0 1
postgres=# SELECT * FROM good_sum_mart;
        good_name         |   sum_sale   
--------------------------+--------------
 Спички хозайственные     |        65.50
 Автомобиль Ferrari FXX K | 370000000.02
(2 rows)
```
> [!NOTE]
> Сумма увеличилась.

Напоследок, все Ferrari вернули через суд
```
postgres=# DELETE FROM sales WHERE good_id = 2;
DELETE 2
postgres=# SELECT * FROM good_sum_mart;
      good_name       | sum_sale 
----------------------+----------
 Спички хозайственные |    65.50
(1 row)
```
> [!NOTE]
> Как видим, триггер корректно заполняет наш отчет, как при вставке данных, так и при удалении.

### Преимущества схемы с триггером
1) Ниже время доступа к отчету (его не нужно каждый раз собирать заново)
2) Разравнивается нагрузка на сервер (если отчет тяжелый, он будет вызывать пики нагрузки, а с триггером нагрузка будет распределена по всем моментам изменения данных)
3) Ничто не мешает теперь проиндексировать наш отчет, чтобы он работал еще быстрее
4) Если отчет собирать по требованию, то придется фиксировать историю цен, чтобы реагировать на ситуации, когда цена товара изменилась, и скажем до одной даты продажи надо считать по одной цене, а после - по другой.

> [!NOTE]
> Однако мое мнение что п.4 - не очень аргумент. Историю цен скорее всего все равно придется делать, как покаызвает практика. Это преимущество только для такого упрощенного случая, в реальности же настоящее преимущество тут как мне кажется это п.2.
