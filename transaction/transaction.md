# Работа с уровнями изоляции транзакции в PostgreSQL

# Описание/Пошаговая инструкция выполнения домашнего задания

* создать новый проект в Яндекс облако или на любых ВМ, докере
* далее создать инстанс виртуальной машины с дефолтными параметрами
* добавить свой ssh ключ в metadata ВМ
* зайти удаленным ssh (первая сессия), не забывайте про ssh-add
* поставить PostgreSQL зайти вторым ssh (вторая сессия) запустить везде psql из под пользователя postgres 
  выключить auto commit 
* сделать в первой сессии новую таблицу и наполнить ее данными 
  create table persons(id serial, first_name text, second_name text); 
  insert into persons(first_name, second_name) values('ivan', 'ivanov');
  insert into persons(first_name, second_name) values('petr', 'petrov'); 
  commit;
* посмотреть текущий уровень изоляции: show transaction isolation level
* начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции
* в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');
* сделать select from persons во второй сессии
* видите ли вы новую запись и если да то почему?
* завершить первую транзакцию - commit;
* сделать select from persons во второй сессии
* видите ли вы новую запись и если да то почему?
* завершите транзакцию во второй сессии
* начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;
* в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');
* сделать select* from persons во второй сессии*
* видите ли вы новую запись и если да то почему?
* завершить первую транзакцию - commit;
* сделать select from persons во второй сессии
* видите ли вы новую запись и если да то почему?
* завершить вторую транзакцию
* сделать select * from persons во второй сессии
* видите ли вы новую запись и если да то почему?

# Отчет по выполнянию задания

Создана виртуальная машина virtualbox, устновлен PostgreSQL 15 версии.

Открыли две сессии подключения к PostgreSQL. В обоих сессии отключили автоматический commit 
в psql командой ```\set AUTOCOMMIT off```.

Создана таблица persons и наполенена данными:
```
postgres=# create table persons(id serial, first_name text, second_name text); insert into persons(first_name, second_name) values('ivan', 'ivanov'); insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
```


*посмотреть текущий уровень изоляции: show transaction isolation level*

```
postgres=# show transaction isolation level;
transaction_isolation
-----------------------
read committed
(1 row)
```

*начать новую транзакцию в обоих сессиях с дефолтным (не меняя) уровнем изоляции*
*в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sergey', 'sergeev');*

В обоих сессиях выполняем команду начала транзакции: ```begin```.
В первой сессии добавляем запись в таблицу persons:

```
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```

*сделать select from persons во второй сессии*
*видите ли вы новую запись и если да то почему?*

```
postgres=*# select * from persons;
id | first_name | second_name
----+------------+-------------
1 | ivan       | ivanov
2 | petr       | petrov
(2 rows)
```
Новая запись не видна во второй сессии, так уровень изоляции не допускает "грязного чтения", 
т.е. не видит незафиксированные данные.

*завершить первую транзакцию - commit;*
*сделать select from persons во второй сессии*
*видите ли вы новую запись и если да то почему?*

Завершаем транзакцию в первой сессии командой ```commit```.
Повторим select во втрой сессии:

```
postgres=*# select * from persons;
id | first_name | second_name
----+------------+-------------
1 | ivan       | ivanov
2 | petr       | petrov
3 | sergey     | sergeev
(3 rows)
```

Новая запись видна, так как при уровне изоляции read committed транзакция видит все зафиксированные строки таблицы 
в других транзакциях. Но при этом уровне изоляции возможна аномалия неповторяющегося чтения.

*завершите транзакцию во второй сессии*
*начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;*
*в первой сессии добавить новую запись insert into persons(first_name, second_name) values('sveta', 'svetova');*
*сделать select * from persons во второй сессии*
*видите ли вы новую запись и если да то почему?*

Начинаем новые транзакции в двух сессиях с уровнем изоляции repeatable read.
Добавлем новую запись в таблицу persons в первом сеансе:

```
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```
Во втром сеансе сделаем выборку из таблицы persons:

```
postgres=*# select * from persons;
id | first_name | second_name
----+------------+-------------
1 | ivan       | ivanov
2 | petr       | petrov
3 | sergey     | sergeev
(3 rows)
```

Новая запись не видна. PostgreSQL не допускает "грязного чтения" при любом уровне изоляции.

*завершить первую транзакцию - commit;*
*сделать select from persons во второй сессии*
*видите ли вы новую запись и если да то почему?*

В первой сессии завершаем транзакцию. В второй сессии сделали выборку из таблицы persons:  

```
postgres=*# select * from persons;
id | first_name | second_name
----+------------+-------------
1 | ivan       | ivanov
2 | petr       | petrov
3 | sergey     | sergeev
(3 rows)
```

Новая запись не видна, так как уровень транзакции repeatable read не допускает неповторяющегося чтения.

*завершить вторую транзакцию*
*сделать select * from persons во второй сессии*
*видите ли вы новую запись и если да то почему?*

Завершили транзакцию во второй сессии. Выполнили select * from persons во второй сессии

```
postgres=*# select * from persons;
id | first_name | second_name
----+------------+-------------
1 | ivan       | ivanov
2 | petr       | petrov
3 | sergey     | sergeev
4 | sveta      | svetova
(4 rows)
```

Новая запись видна, так как на начало транзакции во второй сессии транзакция в первой сессии была завешена.

