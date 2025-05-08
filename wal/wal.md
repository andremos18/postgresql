# Работа с журналами

## Цели
* уметь работать с журналами и контрольными точками
* уметь настраивать параметры журналов

# Описание/Пошаговая инструкция выполнения домашнего задания

1. Настройте выполнение контрольной точки раз в 30 секунд.
2. 10 минут c помощью утилиты pgbench подавайте нагрузку.
3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.
6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

# Отчет по выполнянию задания

*Настройте выполнение контрольной точки раз в 30 секунд.*

Делаем настройку выполнения контрольной точки каждые 30 сек и рестарт кластера:

```
alter system set checkpoint_timeout = '30';
sudo pg_ctlcluster 15 main3 stop
sudo pg_ctlcluster 15 main3 start

```

Запомнинаем текущий номер lsn:
```
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 0/EA995A40
(1 row)
```

Выполняем сброс статистики:

```
sudo -u postgres psql -c "select pg_stat_reset()"
sudo -u postgres psql -c "select pg_stat_reset_shared('bgwriter')"
```


Подаем нагрузку 10 минут c помощью утилиты pgbench

```
sudo -u postgres pgbench -p 5433 -c 50 -j 2 -P 10 -T 600 postgres

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 685220
number of failed transactions: 0 (0.000%)
latency average = 43.780 ms
latency stddev = 54.020 ms
initial connection time = 48.987 ms
tps = 1141.961095 (without initial connection time)
```



Смотрим текущий номер lsn:

```
postgres=# select pg_current_wal_insert_lsn();
 pg_current_wal_insert_lsn 
---------------------------
 1/11DC1C80
(1 row)
```

Вычисляем объем сгенерированных журнальных файлов:

```
postgres=# select round(('1/11DC1C80'::pg_lsn - '0/EA995A40'::pg_lsn)/(1024*1024),2) as MB;
   mb   
--------
 628.17
(1 row)
```

объем сгенерированных журнальных файлов примерно равен 628МБ.


*Оцените, какой объем приходится в среднем на одну контрольную точку.*

Выполним оценку среднего объема журнала на одну контрольную точку.

Посмотрим размер страницы

```
postgres=# select * from pg_settings block where block.name = 'block_size' \gx
-[ RECORD 1 ]---+--------------------------------
name            | block_size
setting         | 8192
unit            |
category        | Preset Options
short_desc      | Shows the size of a disk block.
extra_desc      |
context         | internal
vartype         | integer
source          | default
min_val         | 8192
max_val         | 8192
enumvals        |
boot_val        | 8192
reset_val       | 8192
sourcefile      |
sourceline      |
pending_restart | f
```

Размер страницы - 8192 байт.


Оценим объем в среднем на одну контрольную точку:

```
postgres=# select
round(buffers_checkpoint*8192/(checkpoints_timed + checkpoints_req)/(1024*1024),2)
from pg_stat_bgwriter bg;
 round 
-------
 18.00
(1 row)

```
Средний объем контрольной точки - 18МБ



*Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?*

Вычислим длительнсть статистики, среднее время выполнения контрольной точки

```
postgres=# select
(round(extract('epoch' from now() - stats_reset)/60)::numeric) "Statistics duration in minutes",
round((round(extract('epoch' from now() - stats_reset))::numeric)/(checkpoints_timed + checkpoints_req),2) "Seconds between
checkpoints",
round(checkpoint_write_time::numeric/((checkpoints_timed + checkpoints_req )*1000),2) "Average
write time per checkpoint (s)",
round(checkpoint_sync_time::numeric/((checkpoints_timed + checkpoints_req)*1000),2) "Average
sync time per checkpoint (s)"
from pg_stat_bgwriter bg \gx
-[ RECORD 1 ]------------------+------
Statistics duration in minutes | 10
Seconds between               +| 29.90
checkpoints                    |
Average                       +| 25.67
write time per checkpoint (s)  |
Average                       +| 0.02
sync time per checkpoint (s)   |
```



Судя по статистике среднее время выполнения контролькой точки 25 секунд. 
Время выполнения КТ укладывается в заданное время между выполнениями контрольных точек checkpoint_timeout 30 сек.
Все контрольные точки выполнены.

Посмотрим стастистику pg_stat_bgwriter 

```
postgres=# select * from pg_stat_bgwriter bg \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 24
checkpoints_req       | 0
checkpoint_write_time | 566017
checkpoint_sync_time  | 383
buffers_checkpoint    | 51160
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 4417
buffers_backend_fsync | 0
buffers_alloc         | 4408
stats_reset           | 2025-05-08 21:34:11.180733+04
```

ВЫполнено 24 контрольные точки, что подтверждает что все контрольные точки выполнены.


*Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.*

Включаем асинхронный коммит:
```
alter system set synchronous_commit = off;
```

делаем рестарт:
```
sudo pg_ctlcluster 15 main stop
sudo pg_ctlcluster 15 main start
```

Запускаем нагрузочное тестирование pgbench:

```
sudo -u postgres pgbench -p 5433 -c 50 -j 2 -P 10 -T 600 postgres
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2192759
number of failed transactions: 0 (0.000%)
latency average = 13.680 ms
latency stddev = 17.405 ms
initial connection time = 48.484 ms
tps = 3654.503322 (without initial connection time)
```

Разница показателях производительности (latency, tps) почти в 3 раза.
Причина в более эффективной записи на диск. Вместо того чтобы моментально записать на диск каждую операцию, 
записывается отдельным процессом по расписанию пачками.


*Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. 
Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. 
Что и почему произошло? как проигнорировать ошибку и продолжить работу?*

Создаем кластер с включенной контрольной суммой страниц:

```
sudo pg_createcluster 15 main1 -- --data-checksums
sudo pg_ctlcluster 15 main1 start;
```

Проверяем настройку:

```
postgres=# show data_checksums;
data_checksums
----------------
on
(1 row)
```

Создаем таблицу students и вставляем строки:

```
create table students (name text);
insert into students (name) values ('Ivanov'), ('Petrov'), ('Sidorov');
```

Получаем путь к таблице students на диске:
```
postgres=# select pg_relation_filepath('students');
 pg_relation_filepath 
----------------------
 base/5/16388
(1 row)

```

Останавливаем кластер:
```
sudo pg_ctlcluster 15 main1 stop;
```

Поменяли в hex-редакторе пару байт в конце файла /var/lib/postgresql/15/main1/base/5/16388

Запускаем кластер:
```
sudo pg_ctlcluster 15 main1 start;
```

Сделаем выборку из таблицы students:

```
postgres=# select * from students;
WARNING:  page verification failed, calculated checksum 50185 but expected 20008
ERROR:  invalid page in block 0 of relation base/5/16388
```

Ошибка в связи с нарушением целостности таблицы на уровне байтов, так как включена проверка целостности - data checksums.

*Как проигнорировать ошибку и продолжить работу?*

Есть несколько вариантов:

Устанавливаем параметр игнорирования ошибки контрольной суммы и повтороим запрос:
```
set ignore_checksum_failure = 'on';
select * from students;
```

Получим результат с предупреждением:

```
postgres=# select * from students;
WARNING:  page verification failed, calculated checksum 50185 but expected 20008
  name   
---------
 Ivantu
 Petrov
 Sidorov
(3 rows)
```

Можно занулить поврежденную страницу и выполнить полный вакуум. Такое решение уничтожит поврежденные данные:

```
SET zero_damaged_pages = on;
vacuum full students;
select * from students;
```