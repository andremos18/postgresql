# Бэкапы

## Цель
применить логический бэкап. Восстановиться из бэкапа.

## Описание/Пошаговая инструкция выполнения домашнего задания:
1. Создаем ВМ/докер c ПГ.
2. Создаем БД, схему и в ней таблицу.
3. Заполним таблицы автосгенерированными 100 записями.
4. Под линукс пользователем Postgres создадим каталог для бэкапов
5. Сделаем логический бэкап используя утилиту COPY
6. Восстановим в 2 таблицу данные из бэкапа.
7. Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц
8. Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!

# Отчет по выполнянию задания

```
CREATE DATABASE backups_db;
CREATE SCHEMA backup_schema;
create table backup_schema.test_table(id integer, n text);
insert into backup_schema.test_table(id, n) select generate_series(1,100) as id,  random()::text;
```

Содадим каталог и меняем владельца на postgres

```
sudo mkdir /var/lib/postgresql/backups
sudo chown postgres:postgres /var/lib/postgresql/backups
```

Сделали логический бэкап используя утилиту psql COPY:

```
\copy backup_schema.test_table to /var/lib/postgresql/backups/test_table_backup.csv
```

Востановили данные из бэкапа во вторую таблицу:

```
create table backup_schema.test_table2(id integer, n text);
\copy backup_schema.test_table2 from /var/lib/postgresql/backups/test_table_backup.csv
```

Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц

```
sudo su postgres
pg_dump -d backups_db -C -U postgres -p 5433 -Fc >  /var/lib/postgresql/backups/backups_bd_backup.gz
```

Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!

```
CREATE DATABASE backups_db2;
CREATE SCHEMA backup_schema;
pg_restore -d backups_db2 -U postgres -p 5433 -n backup_schema -t test_table2 /var/lib/postgresql/backups/backups_bd_backup.gz
```
