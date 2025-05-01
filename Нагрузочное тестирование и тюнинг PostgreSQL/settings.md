# Нагрузочное тестирование и тюнинг PostgreSQL

Запустили виртуальную машину с 4GB RAM, 2 ядра CPU, 10GB SSD.

Cоздали новый кластер

```
sudo pg_createcluster 15 main3
```

Выполнили иициализацию pgbench стандартной конфигурации  (https://postgrespro.ru/docs/postgrespro/14/pgbench)

```
sudo -u postgres pgbench -i postgres -p 5434

dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
vacuuming...                                                                              
creating primary keys...
done in 0.15 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.07 s, vacuum 0.03 s, primary keys 0.03 s).
```

Убедились что таблицы созданы

```
sudo -u postgres psql -p 5434
postgres=# \dt
List of relations
Schema |       Name       | Type  |  Owner   
--------+------------------+-------+----------
public | pgbench_accounts | table | postgres
public | pgbench_branches | table | postgres
public | pgbench_history  | table | postgres
public | pgbench_tellers  | table | postgres
(4 rows)
```

Звпустили нагрузочное тестирование 

```
sudo -u postgres pgbench -p 5434 -c 50 -j 2 -P 10 -T 600 postgres
```

Получили результаты

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 659405
number of failed transactions: 0 (0.000%)
latency average = 45.493 ms
latency stddev = 95.652 ms
initial connection time = 50.910 ms
tps = 1098.959768 (without initial connection time)
```

Меняем параметры в postgresql.conf чтоы улучшить производительность, учитывая рекомендации 
https://pgconfigurator.cybertec.at/

```
shared_buffers = '1024MB'   (25% от RAM)
effective_cache_size = '3GB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
checkpoint_timeout = '30min'
synchronous_commit = 'off'
max_wal_size = '1024 MB'
min_wal_size = '512 MB'
```

Проверяем применились ли параметры (поле applied)

```
postgres=# select * from pg_file_settings where name='shared_buffers';

sourcefile                | sourceline | seqno |      name      | setting | applied |            error             
------------------------------------------+------------+-------+----------------+---------+---------+------------------------------
/etc/postgresql/15/main3/postgresql.conf |        129 |    11 | shared_buffers | 1024MB  | f       | setting could not be applied
(1 row)
```

Выполняем перезапуск кластера
```
sudo pg_ctlcluster 15 main3 stop
sudo pg_ctlcluster 15 main3 start
```

Проверяем применились ли параметры

```
postgres=# select * from pg_file_settings where name='shared_buffers';
                sourcefile                | sourceline | seqno |      name      | setting | applied | error 
------------------------------------------+------------+-------+----------------+---------+---------+-------
 /etc/postgresql/15/main3/postgresql.conf |        129 |    11 | shared_buffers | 1024MB  | t       | 
(1 row)

```

Выполняем тест повторно

```
sudo -u postgres pgbench -p 5434 -c 50 -j 2 -P 10 -T 600 postgres
```

Получаем результат с значительно лучшим tps

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2366036
number of failed transactions: 0 (0.000%)
latency average = 12.678 ms
latency stddev = 15.905 ms
initial connection time = 52.789 ms
tps = 3943.340368 (without initial connection time)
```

# Тест производительности PostgreSQL с помощью Sysbench
Устанавливаем sysbench 

https://github.com/akopytov/sysbench

и скачиваем скрипт https://github.com/Percona-Lab/sysbench-tpcc

Выполняем подготовку данных
```
./tpcc.lua --pgsql-host=localhost --pgsql-port=5434 --pgsql-user=postgres --pgsql-password=123 --pgsql-db=postgres --time=120 --threads=10 --report-interval=1 --tables=2 --scale=10 --use_fk=0  --trx_level=RC --db-driver=pgsql prepare
```

Запускаем тест
```
./tpcc.lua --pgsql-host=localhost --pgsql-port=5434 --pgsql-user=postgres --pgsql-password=123 --pgsql-db=postgres --time=120 --threads=10 --report-interval=1 --tables=2 --scale=10 --use_fk=0  --trx_level=RC --db-driver=pgsql run


SQL statistics:
    queries performed:
        read:                            1047066
        write:                           1086587
        other:                           161488
        total:                           2295141
    transactions:                        80734  (672.23 per sec.)
    queries:                             2295141 (19110.36 per sec.)
    ignored errors:                      361    (3.01 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          120.0983s
    total number of events:              80734

Latency (ms):
         min:                                    0.29
         avg:                                   14.87
         max:                                  157.63
         95th percentile:                       40.37
         sum:                              1200151.86

Threads fairness:
    events (avg/stddev):           8073.4000/120.17
    execution time (avg/stddev):   120.0152/0.03

```

https://www.percona.com/blog/tuning-postgresql-for-sysbench-tpcc/
https://severalnines.com/blog/how-benchmark-postgresql-performance-using-sysbench/