# Работа с индексами

## Цели

1. знать и уметь применять основные виды индексов PostgreSQL 
2. строить и анализировать план выполнения запроса 
3. уметь оптимизировать запросы для с использованием индексов

# Описание/Пошаговая инструкция выполнения домашнего задания

Создать индексы на БД, которые ускорят доступ к данным.
В данном задании тренируются навыки:

* определения узких мест
* написания запросов для создания индекса
* оптимизации

Необходимо:
1. Создать индекс к какой-либо из таблиц вашей БД
2. Прислать текстом результат команды explain, в которой используется данный индекс
3. Реализовать индекс для полнотекстового поиска
4. Реализовать индекс на часть таблицы или индекс на поле с функцией
5. Создать индекс на несколько полей
6. Написать комментарии к каждому из индексов
7. Описать что и как делали и с какими проблемами столкнулись

# Отчет по выполнению задания


*1. Создать индекс к какой-либо из таблиц вашей БД*\
*2. Прислать текстом результат команды explain, в которой используется данный индекс*

Работу выполним на демонстрационной базе данных «Авиаперевозки» https://edu.postgrespro.ru/demo_small.zip. 

Получим оценку плана выполнения запроса количество билетов на рейсы в 2016 году. Дя этого выполняем explain analyze.
С параметром analyse команда explain на самом деле выполняет запрос, а затем выводит фактическое число строк и время выполнения, 
накопленное в каждом узле плана, вместе с теми же оценками, что выдаёт обычная команда EXPLAIN.

```
explain analyze
SELECT f.departure_airport , f.departure_airport , count(1) as cnt_ticket 
from bookings.flights f
left join bookings.ticket_flights tf on tf.flight_id = f.flight_id 
where extract(year from f.scheduled_departure)=2016
group by f.departure_airport , f.departure_airport
```

Результат команда представлен ниже.

```
HashAggregate  (cost=22834.74..22835.57 rows=83 width=16) (actual time=258.899..258.907 rows=104 loops=1)
  Group Key: f.departure_airport
  Batches: 1  Memory Usage: 32kB
  ->  Hash Right Join  (cost=890.89..22808.54 rows=5241 width=4) (actual time=19.343..171.540 rows=1056621 loops=1)
        Hash Cond: (tf.flight_id = f.flight_id)
        ->  Seq Scan on ticket_flights tf  (cost=0.00..19172.26 rows=1045726 width=4) (actual time=0.007..43.370 rows=1045726 loops=1)
        ->  Hash  (cost=888.82..888.82 rows=166 width=8) (actual time=19.328..19.329 rows=33121 loops=1)
              Buckets: 65536 (originally 1024)  Batches: 1 (originally 1)  Memory Usage: 1806kB
              ->  Seq Scan on flights f  (cost=0.00..888.82 rows=166 width=8) (actual time=0.019..13.332 rows=33121 loops=1)
                    Filter: (EXTRACT(year FROM scheduled_departure) = '2016'::numeric)
Planning Time: 0.412 ms
Execution Time: 258.955 ms
```

Видим, что идет полное сканирование таблицы ticket_flights.

Создадим индекс на таблицу ticket_flights по полю flight_id:

```
CREATE INDEX ticket_flights_flight_id ON bookings.ticket_flights (flight_id)
```

Повторим запрос explain analyze.

Получили результат:

```
HashAggregate  (cost=1839.52..1840.35 rows=83 width=16) (actual time=218.236..218.244 rows=104 loops=1)
  Group Key: f.departure_airport
  Batches: 1  Memory Usage: 32kB
  ->  Nested Loop Left Join  (cost=0.42..1813.31 rows=5241 width=4) (actual time=0.020..131.503 rows=1056621 loops=1)
        ->  Seq Scan on flights f  (cost=0.00..888.82 rows=166 width=8) (actual time=0.012..7.094 rows=33121 loops=1)
              Filter: (EXTRACT(year FROM scheduled_departure) = '2016'::numeric)
        ->  Index Only Scan using ticket_flights_flight_id on ticket_flights tf  (cost=0.42..5.04 rows=53 width=4) (actual time=0.001..0.002 rows=32 loops=33121)
              Index Cond: (flight_id = f.flight_id)
              Heap Fetches: 0
Planning Time: 0.458 ms
Execution Time: 218.281 ms
```

Видим, что поиск в таблице ticket_flights теперь идет по индексу ticket_flights_flight_id. 

*3. Реализовать индекс для полнотекстового поиска*

Создадим таблицу Отзывов. С полями: id - идентификатор записи, content - содержимое отзыва  

```
create table reviews (
id                  integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
content                varchar             not null
);
```

И наполним:

```
insert into reviews(content) values
('хорошо сделано'),
('все мопеды хороши'),
('стильный мопед');
```

Создадим индекс для полнотектового поиска по полю content:

``
CREATE INDEX idx_reviews_content ON reviews USING GIN (to_tsvector('russian', content));
``

Выпоолним поиск отзывов в которорых упоминается слово "мопед":

``
SELECT * FROM reviews WHERE to_tsvector('russian', content) @@ to_tsquery('russian', 'мопед');
``

Получили все искомые отзывы:

```
id |      content      
----+-------------------
2 | все мопеды хороши
3 | стильный мопед
(2 rows)
```

*4. Реализовать индекс на часть таблицы или индекс на поле с функцией*

Работу выполним на демонстрационной базе данных «Авиаперевозки» https://edu.postgrespro.ru/demo_small.zip.
Создадим частичный индекс на таблице flights по полю status:
``
create index idx_flights_arrived on bookings.flights(status) where status = 'Arrived';
``
Выполним анализ запроса:
``
explain select count(1) from bookings.flights where status = 'Arrived';
``

Убедились, что индекс используется:

```
Aggregate  (cost=364.00..364.01 rows=1 width=8)
->  Index Only Scan using idx_flights_arrived on flights  (cost=0.29..322.23 rows=16707 width=0)
```

*5. Создать индекс на несколько полей*

Создадим составной индекс на таблицу tickets (билеты) по полю book_ref (номер бронирования) и passenger_id (идентификатор пассажира)

``
create index tickets_book_ref_passenger_id_idx on bookings.tickets(book_ref, passenger_id);
``
Выполним анализ
``
explain  select * from bookings.tickets where book_ref = 'BD402C' and passenger_id = '6243 166891' ;
``

Убеждаемся, что индекс работает:

```
Index Scan using tickets_book_ref_passenger_id_idx on tickets  (cost=0.42..8.44 rows=1 width=104)
Index Cond: ((book_ref = 'BD402C'::bpchar) AND ((passenger_id)::text = '6243 166891'::text))
```
