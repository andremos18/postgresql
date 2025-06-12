# Секционирование таблицы
## Цель
научиться выполнять секционирование таблиц в PostgreSQL;
повысить производительность запросов и упростив управление данными;

# Описание/Пошаговая инструкция выполнения домашнего задания:
На основе готовой базы данных примените один из методов секционирования в зависимости от структуры данных.
https://postgrespro.ru/education/demodb

# Отчет по выполнению задания

В демонстрационной БД «Авиаперевозки» для секционирования выбрана таблица бронирований bookings по диапазонам значений дат 
бронирования (book_date). Выполним секционирование по месяцам. Выбор сделан исходя из следующих предположений:
* Выборка из таблицы будет производится только с указанием диапазона дат. 
* В большинстве случаев будут интересовать записи бронирования только в рамках одного месяца.  

Создадим новую партицированную таблицу bookings_main:

```
CREATE TABLE bookings.bookings_main (
book_ref character(6) NOT NULL,
book_date timestamp with time zone NOT NULL,
total_amount numeric(10,2) NOT null,
    PRIMARY KEY (book_ref, book_date)
) PARTITION BY RANGE (book_date);
```

Создадим партиции скриптом:

```
DO $$
declare
 dates record;
 minYear integer;
 minMonth integer;
 maxYear integer;
 maxMonth integer;
 currentMonth integer;
 dateFrom varchar;
 dateTo varchar;
BEGIN
     select into dates min(book_date) minDate, max(book_date) maxDate from bookings.bookings b;

     minYear = extract(year from dates.minDate);
     maxYear = extract(year from dates.maxDate);
     RAISE INFO 'min max: % % % %', dates.minDate, dates.maxDate, minYear, maxYear;

     currentMonth = extract(month from dates.minDate);
     maxMonth = extract(month from dates.maxDate);

     FOR y IN minYear..maxYear LOOP
        FOR m IN currentMonth..11 LOOP
          IF m < 9 THEN
           dateFrom = format('%s-0%s-01', y, m);
           dateTo = format('%s-0%s-01', y, m + 1);
          ELSEIF m = 9 THEN
           dateFrom = format('%s-0%s-01', y, m);
           dateTo = format('%s-%s-01', y, m + 1);
          ELSE
           dateFrom = format('%s-%s-01', y, m);
           dateTo = format('%s-%s-01', y, m + 1);
          END IF;

RAISE INFO 'create table bookings.bookings_%_% partition of bookings.bookings_main for values from (%) TO (%);', y, m, quote_literal(dateFrom), quote_literal(dateTo);
execute format('create table bookings.bookings_%s_%s partition of bookings.bookings_main for values from (%s) TO (%s);', y, m, quote_literal(dateFrom), quote_literal(dateTo));

        END LOOP;
        currentMonth = 1;

     END LOOP;
     
END$$ language plpgsql;
```

Выполним перенос данных в новую таблицу:
```
INSERT INTO bookings.bookings_main
SELECT * FROM bookings.bookings;
```

Срвнивая количетво строк в исходной таблица и партициронанной убедились, что данные перенесены корректно:

Например, сравнивая данные в диапазоне дат ['2016-08-01'; '2016-09-01'):
```
select *
from bookings.bookings
where book_date between '2016-08-01' and '2016-09-01';

select *
from bookings.bookings_main
where book_date between '2016-08-01' and '2016-09-01';
```

Сравним планы запросов.

Несекционированная таблица:
```
explain analyze
select count(*)
from bookings.bookings
where book_date between '2016-08-01' and '2016-09-01'
```

```
Finalize Aggregate  (cost=5022.07..5022.08 rows=1 width=8) (actual time=12.980..13.930 rows=1 loops=1)
  ->  Gather  (cost=5021.96..5022.07 rows=1 width=8) (actual time=12.316..13.923 rows=2 loops=1)
        Workers Planned: 1
        Workers Launched: 1
        ->  Partial Aggregate  (cost=4021.96..4021.97 rows=1 width=8) (actual time=9.131..9.132 rows=1 loops=2)
              ->  Parallel Seq Scan on bookings  (cost=0.00..3992.72 rows=11697 width=0) (actual time=0.061..8.613 rows=9923 loops=2)
                    Filter: ((book_date >= '2016-08-01 00:00:00+04'::timestamp with time zone) AND (book_date <= '2016-09-01 00:00:00+04'::timestamp with time zone))
                    Rows Removed by Filter: 121471
Planning Time: 0.230 ms
Execution Time: 13.965 ms
```


Секционированная таблица:

```
explain analyze
select count(*)
from bookings.bookings_main
where book_date between '2016-08-01' and '2016-09-01'
```

```
Finalize Aggregate  (cost=3880.43..3880.44 rows=1 width=8) (actual time=14.974..15.963 rows=1 loops=1)
  ->  Gather  (cost=3880.21..3880.42 rows=2 width=8) (actual time=13.436..15.958 rows=3 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Partial Aggregate  (cost=2880.21..2880.22 rows=1 width=8) (actual time=6.576..6.578 rows=1 loops=3)
              ->  Parallel Append  (cost=0.00..2859.54 rows=8268 width=0) (actual time=1.279..6.046 rows=6615 loops=3)
                    ->  Parallel Seq Scan on bookings_2016_9 bookings_main_2  (cost=0.00..2516.13 rows=1 width=0) (actual time=1.298..4.134 rows=2 loops=3)
                          Filter: ((book_date >= '2016-08-01 00:00:00+04'::timestamp with time zone) AND (book_date <= '2016-09-01 00:00:00+04'::timestamp with time zone))
                          Rows Removed by Filter: 55197
                    ->  Parallel Seq Scan on bookings_2016_8 bookings_main_1  (cost=0.00..302.07 rows=11671 width=0) (actual time=0.010..3.519 rows=19841 loops=1)
                          Filter: ((book_date >= '2016-08-01 00:00:00+04'::timestamp with time zone) AND (book_date <= '2016-09-01 00:00:00+04'::timestamp with time zone))
Planning Time: 0.279 ms
Execution Time: 15.994 ms
```

По плану запроса видно, что планировщик выбрал правильную секцию, что уменьшает объем прочитанных данных. 

