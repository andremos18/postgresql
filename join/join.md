# Работа с join'ами, статистикой

## Цели

1. знать и уметь применять различные виды join'ов
2. строить и анализировать план выполенения запроса
3. оптимизировать запрос
4. уметь собирать и анализировать статистику для таблицы

# Описание/Пошаговая инструкция выполнения домашнего задания

В результате выполнения ДЗ вы научитесь пользоваться
различными вариантами соединения таблиц.
В данном задании тренируются навыки:

написания запросов с различными типами соединений
Необходимо:
1. Реализовать прямое соединение двух или более таблиц
2. Реализовать левостороннее (или правостороннее)
   соединение двух или более таблиц
3. Реализовать кросс соединение двух или более таблиц
4. Реализовать полное соединение двух или более таблиц
5. Реализовать запрос, в котором будут использованы
   разные типы соединений
6. Сделать комментарии на каждый запрос
К работе приложить структуру таблиц, для которых
выполнялись соединения
Задание со звездочкой*
Придумайте 3 своих метрики на основе показанных представлений, отправьте их через ЛК, а так же поделитесь с коллегами в слаке

# Отчет по выполнению задания

Скачали скрипт создания демонстрационной база данных «Авиаперевозки» https://edu.postgrespro.ru/demo_small.zip

![структура таблиц https://postgrespro.ru/docs/postgrespro/9.6/apjs02](schema.png)

Выполнили скрипт:
```
psql \i  '/home/demo_small.sql'
```

**1. Реализовать прямое соединение двух или более таблиц**

Получим Информацию о рейсах - Модель самолета, аэропорт отправления, аэропорт прибытия и расписание:

```
select a.aircraft_code, a.model, dep.city departure, arr.city arrive, f.scheduled_departure, f.scheduled_arrival
from bookings.aircrafts a
join bookings.flights f on a.aircraft_code = f.aircraft_code
join bookings.airports dep on dep.airport_code = f.departure_airport
join bookings.airports arr on arr.airport_code = f.arrival_airport
```

**2. Реализовать левостороннее (или правостороннее) соединение двух или более таблиц**
**5. Реализовать запрос, в котором будут использованы разные типы соединений**

Найдем самолеты, которые не принимают участия в рейсах:

```
select a.*
from bookings.aircrafts a
left join bookings.flights f on a.aircraft_code = f.aircraft_code
where f.flight_id is null
```


Получим информацию о рейсах на которые не был приобретен ни один билет:

```
select f.flight_id, a.aircraft_code, a.model, dep.city departure, arr.city arrive, f.scheduled_departure, f.scheduled_arrival
from bookings.aircrafts a
join bookings.flights f on a.aircraft_code = f.aircraft_code
join bookings.airports dep on dep.airport_code = f.departure_airport
join bookings.airports arr on arr.airport_code = f.arrival_airport
left join bookings.ticket_flights tf on f.flight_id = tf.flight_id
where tf.flight_id is null
```

**3. Реализовать кросс соединение двух или более таблиц**

Найдем аэропорты между которыми нет прямых рейсов и расстояние между ними:

```
select dep.airport_name , arr.airport_name,
(point(dep.longitude,dep.latitude) <@> point(arr.longitude,arr.latitude)) as distance from bookings.airports dep
cross join bookings.airports arr
left join bookings.flights f on f.departure_airport=dep.airport_code and f.arrival_airport=arr.airport_code
where f.flight_id is null
order by distance desc
```

Для вычисления расстояния потребуется установить расширения:

```
create extension cube;
create extension earthdistance;
```



**4. Реализовать полное соединение двух или более таблиц**

Найдем расхождение между бронированием и билетами:

```
select *
from bookings.bookings b
full join bookings.tickets t on b.book_ref = t.book_ref
where b.book_ref is null or t.ticket_no is null
```


**Метрики**


Бизнес метрика: Список рейсов с заполненностью мест менее 50% в каком-то году, например в 2016:

```
select avg(stat_ticket.cnt_ticket*100.0 / stat_seats.cnt_seats) as occupancy, f.flight_no
from bookings.flights f

LEFT JOIN lateral
( SELECT count(1) as cnt_ticket FROM bookings.ticket_flights tf
WHERE tf.flight_id = f.flight_id
) AS stat_ticket ON true

LEFT JOIN lateral
( SELECT count(1) as cnt_seats FROM bookings.seats s
WHERE s.aircraft_code = f.aircraft_code
) AS stat_seats ON true
where extract(year from f.scheduled_departure)=2016 and status = 'Arrived' 
and (stat_ticket.cnt_ticket*100.0 / stat_seats.cnt_seats) < 50
group by f.flight_no
order by occupancy desc
```

Активные запросы длительностью более 5 сек:

```
select now() - query_start as runtime, usename, datname, state, wait_event_type, wait_event, query
from pg_stat_activity
where now() - query_start > '5 seconds'::interval and state='active'
order by runtime desc
```

Установим расширение и выполним рестарт:

```
CREATE EXTENSION pg_stat_statements;
alter system set shared_preload_libraries = 'pg_stat_statements';
pg_ctlcluster 15 main restart
```

Получим топ по времени выполнения:

```
SELECT substring(query, 1, 50) AS short_query, round(total_exec_time::numeric, 2) AS total_time,
calls, rows, round(total_exec_time::numeric / calls, 2) AS avg_time,
round((100 * total_exec_time / sum(total_exec_time::numeric) OVER ())::numeric, 2) AS percentage_cpu
FROM pg_stat_statements
ORDER BY total_time DESC LIMIT 20;
```