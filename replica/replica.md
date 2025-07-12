# Репликация

## Цель
реализовать свой миникластер на 3 ВМ.

## Описание/Пошаговая инструкция выполнения домашнего задания

1. На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.
2. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.
3. На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.
4. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.
5. 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).

# Отчет по выполнению задания

Имеем 3 виртуальные машины:

vm1 host: 192.168.0.12 port: 5432
vm2 host: 192.168.0.14 port: 5432
vm3 host 192.168.0.13 port 5432

Выполним настройку:

Путь к pg_hba.config получили командой:
```
show hba_file;
```

В pg_hba.config добавили строки:
```
host    all             all             0.0.0.0/0               scram-sha-256
host    replication     all             0.0.0.0/0               scram-sha-256
```

Путь к postgresql.conf получили командой:
```
show config_file;
```

В postgresql.conf добавили строки:

```
listen_addresses = '*'
wal_level = 'logical'
```

Выполнили рестарт:
```
sudo pg_ctlcluster 17 main restart
```

*1. На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.*
*3. На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.*

На vm1, vm2, vm3  выполнили:

```
create database test_db;
create table test(id integer PRIMARY KEY, text varchar);
create table test2(id integer PRIMARY KEY, number integer, text varchar);
```

*2. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.*
*4. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.*

На vm1 выполнили:

```
create publication test_pub FOR TABLE test;
```

На vm2 выполнили:

```
create subscription test_sub connection 'host=192.168.0.12 port=5432 user=postgres password=123 dbname=test_db' publication test_pub WITH (copy_data = true);
create publication test2_pub FOR TABLE test2;
```

На vm1 выполнили:

```
create subscription test2_sub connection 'host=192.168.0.14 port=5432 user=postgres password=123 dbname=test_db' publication test2_pub WITH (copy_data = true);
```

*5. 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).*

```
sudo -u postgres psql test_db -c "CREATE SUBSCRIPTION test_sub CONNECTION 'host=192.168.0.12 port=5432 user=postgres password=123 dbname=test_db' PUBLICATION test_pub WITH (copy_data = true)"
sudo -u postgres psql test_db -c "CREATE SUBSCRIPTION test2_sub CONNECTION 'host=192.168.0.14 port=5432 user=postgres password=123 dbname=test_db' PUBLICATION test2_pub WITH (copy_data = true)"
```

*реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3*

```
sudo systemctl stop postgresql
sudo -u postgres rm -rf /var/lib/postgresql/17/main/*
sudo -u postgres pg_basebackup --host=192.168.0.13 --port=5432 --username=postgres --pgdata=/var/lib/postgresql/17/main --progress --write-recovery-conf --create-slot --slot=replica1
sudo systemctl start postgresql
```