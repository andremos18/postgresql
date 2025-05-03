# Настройка autovacuum с учетом особеностей производительности

Создан инстанс virtualbox ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB.
Установлен на него PostgreSQL 15 с дефолтными настройками.

Выполнили инциализацию pgbench
```
sudo -u postgres pgbench -i postgres -p 5435

user@vubuntu-22:~/sysbench-tpcc$ sudo -u postgres pgbench -i postgres -p 5435
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.05 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.16 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.07 s, vacuum 0.03 s, primary keys 0.05 s).
```


Справочно. Удалить данные, созданные инициализацией, можно командой:
```
sudo -u postgres pgbench -i -I d -U postgres postgres -p 5435
```

Выполнили нагрузочное тестирование

```
sudo -u postgres pgbench -c8 -P 6 -T 60 -U postgres postgres -p 5435

user@vubuntu-22:~/sysbench-tpcc$ sudo -u postgres pgbench -c8 -P 6 -T 60 -U postgres postgres -p 5435
pgbench (15.12 (Ubuntu 15.12-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 1279.7 tps, lat 6.227 ms stddev 8.430, 0 failed
progress: 12.0 s, 1323.2 tps, lat 6.043 ms stddev 5.854, 0 failed
progress: 18.0 s, 1338.2 tps, lat 5.976 ms stddev 4.119, 0 failed
progress: 24.0 s, 1356.4 tps, lat 5.894 ms stddev 4.175, 0 failed
progress: 30.0 s, 1333.4 tps, lat 5.998 ms stddev 4.188, 0 failed
progress: 36.0 s, 1330.8 tps, lat 6.008 ms stddev 4.413, 0 failed
progress: 42.0 s, 1359.7 tps, lat 5.883 ms stddev 4.172, 0 failed
progress: 48.0 s, 1294.3 tps, lat 6.177 ms stddev 7.652, 0 failed
progress: 54.0 s, 1377.8 tps, lat 5.804 ms stddev 4.080, 0 failed
progress: 60.0 s, 1361.9 tps, lat 5.872 ms stddev 3.928, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 80140
number of failed transactions: 0 (0.000%)
latency average = 5.986 ms
latency stddev = 5.304 ms
initial connection time = 17.980 ms
tps = 1335.644206 (without initial connection time)
```

Применили значения параметров postgresql.conf на указанные к ДЗ, выполнили рестарт и повторили тест

```
sudo pg_ctlcluster 15 main stop
sudo pg_ctlcluster 15 main start

sudo -u postgres pgbench -c8 -P 6 -T 60 -U postgres postgres -p 5435

pgbench (15.12 (Ubuntu 15.12-1.pgdg24.04+1))
starting vacuum...end.
progress: 6.0 s, 1375.8 tps, lat 5.789 ms stddev 4.055, 0 failed
progress: 12.0 s, 1393.0 tps, lat 5.741 ms stddev 3.977, 0 failed
progress: 18.0 s, 1389.7 tps, lat 5.752 ms stddev 3.914, 0 failed
progress: 24.0 s, 1394.5 tps, lat 5.736 ms stddev 3.948, 0 failed
progress: 30.0 s, 1377.2 tps, lat 5.808 ms stddev 3.955, 0 failed
progress: 36.0 s, 1389.3 tps, lat 5.755 ms stddev 3.953, 0 failed
progress: 42.0 s, 1368.1 tps, lat 5.846 ms stddev 3.861, 0 failed
progress: 48.0 s, 1383.5 tps, lat 5.779 ms stddev 3.988, 0 failed
progress: 54.0 s, 1361.0 tps, lat 5.873 ms stddev 4.184, 0 failed
progress: 60.0 s, 1398.7 tps, lat 5.718 ms stddev 3.933, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 82993
number of failed transactions: 0 (0.000%)
latency average = 5.780 ms
latency stddev = 3.979 ms
initial connection time = 19.227 ms
tps = 1383.216897 (without initial connection time)
```

Производительность БД увеличилась (выше tps, меньше latency), так как значения по умолчанию далеки от оптимальных. 
Сделано специально чтобы для запуска postgresql требовалось минимум системных ресурсов.
Параметры из ДЗ лучше по производительности. В частности, shared_buffers (1GB) выбран в соответствии с рекомендациями 
в 25% от общего RAM (4GB). Но есть проблема в параметре max_wal_size = 16GB, хотя размер накопителя SSD всего 10GB.

Создана таблица student с текстовым полем и заполнена данными в размере 1млн строк

```
CREATE TABLE student ( fio varchar);
INSERT INTO student(fio) SELECT 'noname' FROM generate_series(1,1000000);
```

Смотрим размер таблицы

```
SELECT pg_size_pretty(pg_total_relation_size('student'));

postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
pg_size_pretty
----------------
35 MB
(1 row)
```

Выполняем 5 раз обновление тектового поля fio таблицы student

```
postgres=# update student set fio = 'noname$';
UPDATE 1000000
postgres=# update student set fio = 'noname$';
UPDATE 1000000
postgres=# update student set fio = 'noname$';
UPDATE 1000000
postgres=# update student set fio = 'noname$';
UPDATE 1000000
postgres=# update student set fio = 'noname$';
UPDATE 1000000
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_TABLEs WHERE relname = 'student';
relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
student |    1000000 |    4998322 |    499 | 2025-05-02 12:58:31.859839+04
(1 row)
```

Размер таблицы увеличился
```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
pg_size_pretty
----------------
207 MB
(1 row)
```



Проверяем сработал ли автовакуум

```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
FROM pg_stat_user_TABLES WHERE relname = 'student';
relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
student |    1000000 |          0 |      0 | 2025-05-02 12:59:32.070894+04
(1 row)
```

Автовауум звпускался в 2025-05-02 12:59, количество мертвых строчек n_dead_tup = 0
Размер таблицы не изменился. Освобождённое место не возвращается операционной системе.

```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
pg_size_pretty
----------------
207 MB
(1 row)
```

Выполняем еще 5 раз обновление текстового поля fio таблицы student

```
postgres=# update student set fio = 'noname$';
UPDATE 1000000
postgres=# update student set fio = 'noname$';
UPDATE 1000000
postgres=# update student set fio = 'noname$';
UPDATE 1000000
postgres=# update student set fio = 'noname$';
UPDATE 1000000
postgres=# update student set fio = 'noname$';
UPDATE 1000000
postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
pg_size_pretty
----------------
207 MB
(1 row)
```

Размер таблицы не изменился.


Отключаем автовакуум на таблице student
```
ALTER TABLE student SET (autovacuum_enabled = off);
```

Выполняем 10 раз обновление поля fio таблицы student
```
update student set fio = 'noname$1'
```

Смотрим размер таблицы
```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
pg_size_pretty
----------------
457 MB
(1 row)
```
Размер таблицы значительно увеличился.
Автовакуум высвобождает пространство и делает его доступным для повторного использования.
При отключенном автоваууме не происходит освобождение пространства, занятого "мертвыми" строками. 
Этим объясняется значительный рост размера таблицы.

Включили автовакуум на таблице student
```
ALTER TABLE student SET (autovacuum_enabled = on);
```
Подождали выполнение автовакуума, выполнили полную очистку

```
postgres=# VACUUM full verbose student;
INFO:  vacuuming "public.student"
INFO:  "public.student": found 0 removable, 1000000 nonremovable row versions in 58448 pages
DETAIL:  0 dead row versions cannot be removed yet.
CPU: user: 0.13 s, system: 0.04 s, elapsed: 0.25 s.
VACUUM
```

Размер таблицы уменьшился

```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('student'));
pg_size_pretty
----------------
42 MB
(1 row)
```

VACUUM FULL переписывает всё содержимое таблицы в новый файл на диске, не содержащий ничего лишнего, что позволяет 
возвратить неиспользованное пространство операционной системе. Эта форма работает намного медленнее и запрашивает 
исключительную блокировку для каждой обрабатываемой таблицы.

# Анонимная процедура, которая в цикле 10 раз обновляет все строчки в таблице student

```
do $$
begin
   for i in 1..10 loop
       raise notice 'Номер итерации %', i;
       update student
       set fio = 'test';
       
   end loop;
end $$;
```