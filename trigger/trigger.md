# Триггеры, поддержка заполнения витрин

## Цель 
Создать триггер для поддержки витрины в актуальном состоянии.

# Отчет по выполнению задания

Создали новую базу и наполнили ее командой psql:
```
\i /home/user/books/student/otus/hw_triggers.sql
```

Сделали первоначальное наполнение таблицы good_sum_mart:

```
INSERT INTO pract_functions.good_sum_mart(good_name, sum_sale)
SELECT G.good_name, sum(G.good_price * S.sales_qty)
FROM pract_functions.goods G
INNER JOIN pract_functions.sales S ON S.good_id = G.goods_id
GROUP BY G.good_name;
```

Триггерная функция добавления в good_sum_mart после вставки вставки:

```
CREATE OR REPLACE FUNCTION pract_functions.insert_sale()
RETURNS trigger
AS
$$
DECLARE
   	new_sum_record record;
begin

SELECT G.good_name as good_name, sum(G.good_price * S.sales_qty) as sum_sale into new_sum_record
	FROM pract_functions.sales S
	left JOIN pract_functions.goods G ON G.goods_id = S.good_id
	where S.sales_id=NEW.sales_id
    group by G.good_name;

if exists(select 1 from pract_functions.good_sum_mart where good_name = new_sum_record.good_name) then
 update pract_functions.good_sum_mart
 set sum_sale = sum_sale + new_sum_record.sum_sale
 where good_name = new_sum_record.good_name;
else
 insert into pract_functions.good_sum_mart (good_name, sum_sale) values (new_sum_record.good_name, new_sum_record.sum_sale);
end if;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Триггер AFTER INSERT:
```
CREATE TRIGGER trg_after_insert_sale
AFTER INSERT
ON pract_functions.sales
FOR EACH ROW
EXECUTE FUNCTION pract_functions.insert_sale();
```

Триггерная функция обновления good_sum_mart:

```
CREATE OR REPLACE FUNCTION pract_functions.update_sale()
RETURNS trigger
AS
$$
DECLARE
    old_sum_record record;
    new_sum_record record;
begin

SELECT G.good_name as good_name, sum(G.good_price * OLD.sales_qty) as sum_sale into old_sum_record
		FROM pract_functions.goods G
		where G.goods_id = OLD.good_id
        group by G.good_name;

update pract_functions.good_sum_mart
 set sum_sale = sum_sale - old_sum_record.sum_sale
 where good_name = old_sum_record.good_name;

SELECT G.good_name as good_name, sum(G.good_price * NEW.sales_qty) as sum_sale into new_sum_record
		FROM pract_functions.goods G
		where G.goods_id = NEW.good_id
        group by G.good_name;

update pract_functions.good_sum_mart
 set sum_sale = sum_sale + new_sum_record.sum_sale
 where good_name = new_sum_record.good_name;

  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

Триггер AFTER UPDATE:

```
CREATE TRIGGER trg_after_update_sale
AFTER UPDATE
ON pract_functions.sales
FOR EACH ROW
EXECUTE FUNCTION pract_functions.update_sale();
```


Триггерная функция удаления записей из good_sum_mart:

```
CREATE OR REPLACE FUNCTION pract_functions.delete_sale()
RETURNS trigger
AS
$$
DECLARE
    old_sum_record record;
begin

SELECT G.good_name as good_name, sum(G.good_price * OLD.sales_qty) as sum_sale into old_sum_record
		FROM pract_functions.goods G
		where G.goods_id = OLD.good_id
        group by G.good_name;

update pract_functions.good_sum_mart
 set sum_sale = sum_sale - old_sum_record.sum_sale
 where good_name = old_sum_record.good_name;


  RETURN OLD;
END;
$$ LANGUAGE plpgsql;
```

Триггер AFTER DELETE:
```
CREATE TRIGGER trg_after_delete_sale
AFTER DELETE
ON pract_functions.sales
FOR EACH ROW
EXECUTE FUNCTION pract_functions.delete_sale();
```

Добавим записи в goods и sales:

```
insert into pract_functions.goods(goods_id, good_name, good_price)
values(3, 'мыло', 1);

INSERT INTO pract_functions.sales(good_id,sales_time,sales_qty) VALUES
	 (3,'2025-03-09 17:51:21.985398+03',1);
```

Посмотрим сатистику продаж good_sum_mart:

```
select *
from pract_functions.good_sum_mart;
```

```
good_name               |sum_sale    |
------------------------+------------+
Автомобиль Ferrari FXX K|185000000.01|
Спички хозайственные    |       65.50|
мыло                    |        1.00|
```

Триггер trg_after_insert_sale отработал корректно

Обновили sales:
```
update pract_functions.sales
set sales_qty = 2
where good_id = 3
```

Содержимое траблицы good_sum_mart:

```
good_name               |sum_sale    |
------------------------+------------+
Автомобиль Ferrari FXX K|185000000.01|
Спички хозайственные    |       65.50|
мыло                    |        2.00|
```

Триггер trg_after_update_sale отработал корректно

Удалили строку из sales:
```
delete from pract_functions.sales
where good_id=3
```

```
good_name               |sum_sale    |
------------------------+------------+
Автомобиль Ferrari FXX K|185000000.01|
Спички хозайственные    |       65.50|
мыло                    |        0.00|
```

Триггер trg_after_delete_sale отработал корректно

`Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
Подсказка: В реальной жизни возможны изменения цен.`

Цена товара может измениться, тогда изменится значение поля good_price таблицы goods. 
В этом случае отчет "по требованию" будет некорректный, так как пересчет будет по новым ценам.

