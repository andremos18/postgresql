# Механизм блокировок

## Цели

Понимать как работает механизм блокировок объектов и строк.

# Описание/Пошаговая инструкция выполнения домашнего задания

1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, 
   удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. 
   Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. 
   Пришлите список блокировок и объясните, что значит каждая.
3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?
4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), 
   заблокировать друг друга?

# Отчет по выполнению задания

*Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках,
удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.*

Выполняем настройку deadlock_timeout в 200 мс

```
alter system set log_lock_waits to on;
alter system set deadlock_timeout to 200;
select pg_reload_conf();
```

Создадим таблицу students и наполним данными:

```
 create table students(id int primary key,name varchar);
 insert into students values (1, 'Ivanov'), (2, 'Petrov'), (3, 'Sidorov');
```

session 1
```
begin;
update students set name='Petrova' where id=2;
SELECT pg_sleep(1);
```

session 2:
```
update students set name='Petrov P';
```

В первом сеансе выпоняем ```commit;```. Команда update во втром сеансе выполнилась успешно.

Убеждаемся, что информация о блокировке попала в журнал

```
tail -n 10 /var/log/postgresql/postgresql-15-main.log

2025-05-11 18:10:27.399 +04 [1199] LOG:  checkpoint complete: wrote 1 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.104 s, sync=0.083 s, total=0.234 s; sync files=1, longest=0.083 s, average=0.083 s; distance=0 kB, estimate=118 kB
2025-05-11 18:12:15.822 +04 [4681] postgres@postgres LOG:  process 4681 still waiting for ShareLock on transaction 5812938 after 200.262 ms
2025-05-11 18:12:15.822 +04 [4681] postgres@postgres DETAIL:  Process holding the lock: 6665. Wait queue: 4681.
2025-05-11 18:12:15.822 +04 [4681] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "students"
2025-05-11 18:12:15.822 +04 [4681] postgres@postgres STATEMENT:  update students set name='Petrov P' where id=2;
2025-05-11 18:12:27.055 +04 [4681] postgres@postgres LOG:  process 4681 acquired ShareLock on transaction 5812938 after 11434.007 ms
2025-05-11 18:12:27.055 +04 [4681] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "students"
2025-05-11 18:12:27.055 +04 [4681] postgres@postgres STATEMENT:  update students set name='Petrov P' where id=2;
2025-05-11 18:12:27.509 +04 [1199] LOG:  checkpoint starting: time
2025-05-11 18:12:27.666 +04 [1199] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.106 s, sync=0.004 s, total=0.157 s; sync files=2, longest=0.003 s, average=0.002 s; distance=0 kB, estimate=106 kB
```

*Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах.
Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны.
Пришлите список блокировок и объясните, что значит каждая.*

В каждом сеансе выполним обновление поля name строки таблицы student с id=1 значениями: Ivanov I, Ivanov II и
Ivanov III. Выведем иденификатор процесса и идентификатор транзакции:

```
BEGIN;
SELECT pg_backend_pid() as pid, txid_current() as tid;
UPDATE students SET name = 'Ivanov I' WHERE id = 1;
```

| шаг | сеанс 1                                                | сеанс 2                                                | сеанс 3                                                |
|-----|--------------------------------------------------------|--------------------------------------------------------|--------------------------------------------------------|
| 1   | begin;                                                 | begin;                                                 | begin;                                                 |
| 2   | SELECT pg_backend_pid() as pid, txid_current() as tid; | SELECT pg_backend_pid() as pid, txid_current() as tid; | SELECT pg_backend_pid() as pid, txid_current() as tid; |
| 3   | UPDATE students SET name = 'Ivanov I' WHERE id = 1;    | UPDATE students SET name = 'Ivanov II' WHERE id = 1;   | UPDATE students SET name = 'Ivanov III' WHERE id = 1;  |

Сведем полученные данные в таблицу:

| сеанс | pid   | tid     |
|-------|-------|---------|
| 1     | 6665  | 5812942 |
| 2     | 4681  | 5812943 |
| 3     | 7545  | 5812944 |



Получим информацию о блокировках:

```
SELECT distinct blocked_locks.pid     AS blocked_pid,
blocked_activity.usename  AS blocked_user,
pg_blocking_pids(blocked_locks.pid) AS blocking_pid,
blocking_activity.usename AS blocking_user,
blocked_activity.query    AS blocked_statement,
blocking_activity.query   AS current_statement_in_blocking_process,
blocked_activity.application_name AS blocked_application,
blocking_activity.application_name AS blocking_application
FROM  pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity  ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = any(pg_blocking_pids(blocked_locks.pid));
WHERE NOT blocked_locks.GRANTED;
```


| blocked_pid | blocked_user | blocking_pid | blocking_user |                   blocked_statement                   |        current_statement_in_blocking_process         | blocked_application | blocking_application |
|-------------|--------------|--------------|---------------|-------------------------------------------------------|------------------------------------------------------|---------------------|----------------------|
| 7545        | postgres     | {4681}       | postgres      | UPDATE students SET name = 'Ivanov III' WHERE id = 1; | UPDATE students SET name = 'Ivanov II' WHERE id = 1; | psql                | psql                 |
| 4681        | postgres     | {6665}       | postgres      | UPDATE students SET name = 'Ivanov II' WHERE id = 1;  | UPDATE students SET name = 'Ivanov I' WHERE id = 1;  | psql                | psql                 |


4681 блокирует 7545, 6665 блокирует 4681.


```
SELECT
row_number() over(ORDER BY pid, virtualxid, transactionid::text::bigint) as n,
locktype as locktype,
CASE
WHEN locktype = 'relation' THEN 'отношение'
WHEN locktype = 'extend' THEN 'расширение отношения'
WHEN locktype = 'frozenid' THEN 'замороженный идентификатор'
WHEN locktype = 'page' THEN 'страница'
WHEN locktype = 'tuple' THEN 'кортеж'
WHEN locktype = 'transactionid' THEN 'идентификатор транзакции'
WHEN locktype = 'virtualxid' THEN 'виртуальный идентификатор'
WHEN locktype = 'object' THEN 'объект'
WHEN locktype = 'userlock' THEN 'пользовательская блокировка'
WHEN locktype = 'advisory' THEN 'рекомендательная'
END AS locktype_desc,
relation::regclass,
CASE WHEN page IS NOT NULL AND tuple IS NOT NULL THEN (select name from students s where s.ctid::text = '(' || page || ',' || tuple || ')' limit 1) ELSE NULL END AS row, -- page, tuple,
virtualxid, transactionid, virtualtransaction,
pid,
CASE WHEN pid = 6665 THEN 'session1' WHEN pid = 4681 THEN 'session2' WHEN pid = 7545 THEN 'session3' END AS session,
mode,
CASE WHEN granted = true THEN 'блокировка получена' ELSE 'блокировка ожидается' END AS granted,
CASE WHEN fastpath = true THEN 'блокировка получена по короткому пути' ELSE 'блокировка получена через основную таблицу блокировок' END AS fastpath
FROM pg_locks WHERE pid in (6665, 4681, 7545)
ORDER BY pid, virtualxid, transactionid::text::bigint;
```


| n  | locktype      | locktype_desc             | relation      | row    | virtualxid | transactionid | virtualtransaction | pid  | session  | mode             | granted              | fastpath                                              |
|----|---------------|---------------------------|---------------|--------|------------|---------------|--------------------|------|----------|------------------|----------------------|-------------------------------------------------------|
| 1  | virtualxid    | виртуальный идентификатор |               |        | 4/216      |               | 4/216              | 4681 | session2 | ExclusiveLock    | блокировка получена  | блокировка получена по короткому пути                 |
| 2  | transactionid | идентификатор транзакции  |               |        |            | 5812942       | 4/216              | 4681 | session2 | ShareLock        | блокировка ожидается | блокировка получена через основную таблицу блокировок |
| 3  | transactionid | идентификатор транзакции  |               |        |            | 5812943       | 4/216              | 4681 | session2 | ExclusiveLock    | блокировка получена  | блокировка получена через основную таблицу блокировок |
| 4  | tuple         | кортеж                    | students      | Ivanov |            |               | 4/216              | 4681 | session2 | ExclusiveLock    | блокировка получена  | блокировка получена через основную таблицу блокировок |
| 5  | relation      | отношение                 | students_pkey |        |            |               | 4/216              | 4681 | session2 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
| 6  | relation      | отношение                 | students      |        |            |               | 4/216              | 4681 | session2 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
| 7  | virtualxid    | виртуальный идентификатор |               |        | 3/68       |               | 3/68               | 6665 | session1 | ExclusiveLock    | блокировка получена  | блокировка получена по короткому пути                 |
| 8  | transactionid | идентификатор транзакции  |               |        |            | 5812942       | 3/68               | 6665 | session1 | ExclusiveLock    | блокировка получена  | блокировка получена через основную таблицу блокировок |
| 9  | relation      | отношение                 | students      |        |            |               | 3/68               | 6665 | session1 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
| 10 | relation      | отношение                 | students_pkey |        |            |               | 3/68               | 6665 | session1 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
| 11 | virtualxid    | виртуальный идентификатор |               |        | 5/613      |               | 5/613              | 7545 | session3 | ExclusiveLock    | блокировка получена  | блокировка получена по короткому пути                 |
| 12 | transactionid | идентификатор транзакции  |               |        |            | 5812944       | 5/613              | 7545 | session3 | ExclusiveLock    | блокировка получена  | блокировка получена через основную таблицу блокировок |
| 13 | tuple         | кортеж                    | students      | Ivanov |            |               | 5/613              | 7545 | session3 | ExclusiveLock    | блокировка ожидается | блокировка получена через основную таблицу блокировок |
| 14 | relation      | отношение                 | students_pkey |        |            |               | 5/613              | 7545 | session3 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |
| 15 | relation      | отношение                 | students      |        |            |               | 5/613              | 7545 | session3 | RowExclusiveLock | блокировка получена  | блокировка получена по короткому пути                 |




* каждый сеанс держит эксклюзивные блокировки (exclusive lock) на номера своих транзакций 
  (transactionid - 3, 8, 10 строки) и виртуальной транзакции (virtualxid - 1, 7, 11 - строки)
* первый сеанс захватил эксклюзивную блокировку строки для ключа и самой строки, строки 10, 9
* оставшиеся два запроса хоть и ожидают блокировки так же повесили row exclusive lock на ключ и строку, строки - 5, 6 и 14, 15
* транзакция сеанса второго захватала эксклюзивную блокировку на версию строки, строка 4
* транзакция третьего сеанса ожидает эксклюзивную блокировку на версию строки, строка 13
* блокировка share lock в строке 2 вызвана тем, что второй сеанс ожидает завершения транзакции первого сеанса.


*Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?*

Выполним запросы в следующей последовательности:

| шаг | сеанс 1                                                | сеанс 2                                                | сеанс 3                                                |
|-----|--------------------------------------------------------|--------------------------------------------------------|--------------------------------------------------------|
| 1   | begin;                                                 | begin;                                                 | begin;                                                 |
| 2   | SELECT pg_backend_pid() as pid, txid_current() as tid; | SELECT pg_backend_pid() as pid, txid_current() as tid; | SELECT pg_backend_pid() as pid, txid_current() as tid; |
| 3   | SELECT name FROM students WHERE id = 1 FOR UPDATE;     | SELECT name FROM students WHERE id = 2 FOR UPDATE;     | SELECT name FROM students WHERE id = 3 FOR UPDATE;     |
| 4   | UPDATE students SET name = 'Ivanov I' WHERE id = 2;    | UPDATE students SET name = 'Ivanov II' WHERE id = 3;   | UPDATE students SET name = 'Ivanov III' WHERE id = 1;  |



| сеанс | pid   | tid     |
|-------|-------|---------|
| 1     | 6665  | 5812950 |
| 2     | 4681  | 5812948 |
| 3     | 7545  | 5812949 |


```
ERROR:  deadlock detected
DETAIL:  Process 7545 waits for ShareLock on transaction 5812950; blocked by process 6665.
Process 6665 waits for ShareLock on transaction 5812948; blocked by process 4681.
Process 4681 waits for ShareLock on transaction 5812949; blocked by process 7545.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,8) in relation "students"
```


 Первый сеанс завис на update,  второй сеанс обновил строку, третий сеанс закончился ошибкой.


В журнале мы видим что процесс третьего сеанса с номером 7545 поймал deadlock:

```
tail -n 30 /var/log/postgresql/postgresql-15-main.log

2025-05-11 22:47:42.685 +04 [1199] LOG:  checkpoint complete: wrote 1 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.105 s, sync=0.082 s, total=0.195 s; sync files=1, longest=0.082 s, average=0.082 s; distance=0 kB, estimate=37 kB
2025-05-11 22:48:12.691 +04 [1199] LOG:  checkpoint starting: time
2025-05-11 22:48:12.844 +04 [1199] LOG:  checkpoint complete: wrote 1 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.104 s, sync=0.003 s, total=0.154 s; sync files=1, longest=0.003 s, average=0.003 s; distance=0 kB, estimate=33 kB
2025-05-11 22:49:46.710 +04 [6665] postgres@postgres LOG:  process 6665 still waiting for ShareLock on transaction 5812948 after 202.473 ms
2025-05-11 22:49:46.710 +04 [6665] postgres@postgres DETAIL:  Process holding the lock: 4681. Wait queue: 6665.
2025-05-11 22:49:46.710 +04 [6665] postgres@postgres CONTEXT:  while updating tuple (0,5) in relation "students"
2025-05-11 22:49:46.710 +04 [6665] postgres@postgres STATEMENT:  UPDATE students SET name = 'Ivanov I' WHERE id = 2;
2025-05-11 22:50:08.357 +04 [4681] postgres@postgres LOG:  process 4681 still waiting for ShareLock on transaction 5812949 after 201.241 ms
2025-05-11 22:50:08.357 +04 [4681] postgres@postgres DETAIL:  Process holding the lock: 7545. Wait queue: 4681.
2025-05-11 22:50:08.357 +04 [4681] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "students"
2025-05-11 22:50:08.357 +04 [4681] postgres@postgres STATEMENT:  UPDATE students SET name = 'Ivanov II' WHERE id = 3;
2025-05-11 22:50:34.844 +04 [7545] postgres@postgres LOG:  process 7545 detected deadlock while waiting for ShareLock on transaction 5812950 after 200.086 ms
2025-05-11 22:50:34.844 +04 [7545] postgres@postgres DETAIL:  Process holding the lock: 6665. Wait queue: .
2025-05-11 22:50:34.844 +04 [7545] postgres@postgres CONTEXT:  while updating tuple (0,8) in relation "students"
2025-05-11 22:50:34.844 +04 [7545] postgres@postgres STATEMENT:  UPDATE students SET name = 'Ivanov III' WHERE id = 1;
2025-05-11 22:50:34.844 +04 [7545] postgres@postgres ERROR:  deadlock detected
2025-05-11 22:50:34.844 +04 [7545] postgres@postgres DETAIL:  Process 7545 waits for ShareLock on transaction 5812950; blocked by process 6665.
Process 6665 waits for ShareLock on transaction 5812948; blocked by process 4681.
Process 4681 waits for ShareLock on transaction 5812949; blocked by process 7545.
Process 7545: UPDATE students SET name = 'Ivanov III' WHERE id = 1;
Process 6665: UPDATE students SET name = 'Ivanov I' WHERE id = 2;
Process 4681: UPDATE students SET name = 'Ivanov II' WHERE id = 3;
2025-05-11 22:50:34.844 +04 [7545] postgres@postgres HINT:  See server log for query details.
2025-05-11 22:50:34.844 +04 [7545] postgres@postgres CONTEXT:  while updating tuple (0,8) in relation "students"
2025-05-11 22:50:34.844 +04 [7545] postgres@postgres STATEMENT:  UPDATE students SET name = 'Ivanov III' WHERE id = 1;
2025-05-11 22:50:34.844 +04 [4681] postgres@postgres LOG:  process 4681 acquired ShareLock on transaction 5812949 after 26688.207 ms
2025-05-11 22:50:34.844 +04 [4681] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "students"
2025-05-11 22:50:34.844 +04 [4681] postgres@postgres STATEMENT:  UPDATE students SET name = 'Ivanov II' WHERE id = 3;
2025-05-11 22:50:42.938 +04 [1199] LOG:  checkpoint starting: time
2025-05-11 22:50:43.094 +04 [1199] LOG:  checkpoint complete: wrote 2 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.105 s, sync=0.005 s, total=0.157 s; sync files=2, longest=0.004 s, average=0.003 s; distance=0 kB, estimate=30 kB
```

В журнале записано в каком процессе обнаружена взаимная блокировка (7545) и попытка взятия какой блокировки привело к deadlock.
Также в журнале есть запросы которые привели к deadlock. Можно восстановить хронологию выполнения запросов и понять причину deadlock.


*Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where),
заблокировать друг друга?*

Поиском в интернете найден способ, описанный ниже.

 Подготовим данные:

```
create table test(id integer primary key generated always as identity, n float);
insert into test(n) select random() from generate_series(1,1000000);
```

В первом сеансе выполним:

```
BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE test SET n = (select id from test order by id asc limit 1 for update);
COMMIT;
```

Во втором сеансе выполним:

```
BEGIN ISOLATION LEVEL REPEATABLE READ;
UPDATE test SET n = (select id from test order by id desc limit 1 for update);
COMMIT;
```

Первый сеанс отвалился с ошибкой:

```
ERROR:  deadlock detected
DETAIL:  Process 6665 waits for ShareLock on transaction 5812957; blocked by process 4681.
Process 4681 waits for ShareLock on transaction 5812955; blocked by process 6665.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (10810,150) in relation "test"
```
