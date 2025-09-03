[презентация patroni.pptx](res/patroni.pptx)

# Содержание

<!-- TOC -->
* [Создание и тестирование высоконагруженного отказоустойчивого кластера PostgreSQL на базе Patroni](#создание-и-тестирование-высоконагруженного-отказоустойчивого-кластера-postgresql-на-базе-patroni)
  * [Задачи](#задачи)
  * [Описание кластера](#описание-кластера)
  * [Устанавка Postgres 17](#устанавка-postgres-17)
  * [Установка Установка ETCD](#установка-установка-etcd)
  * [Установка Patroni](#установка-patroni)
    * [Как динамически изменить кофигурацию PostgreSQL](#как-динамически-изменить-кофигурацию-postgresql)
    * [Создание базы данных](#создание-базы-данных)
    * [Автоматический failover](#автоматический-failover)
  * [Установка haproxy](#установка-haproxy)
  * [Установка keepalived](#установка-keepalived)
<!-- TOC -->

# Создание и тестирование высоконагруженного отказоустойчивого кластера PostgreSQL на базе Patroni

## Задачи
1. Развернуть кластер Patroni
2. Настроить HAProxy для обеспечения высокой доступноси и балансировки нагрузки и службу keepalived для обслуживания vip

## Описание кластера

Кластер Patroni состоит из трех узлов pg0,pg1,pg2. 
Два узла haproxy1, haproxy2 обеспечивают балансировку запросов на чтение между репликами и доступ к лидеру для чтения и записи.
В качестве точки входа используется виртуальный ip адрес (VIP). 
Служба keepalived автоматически переключает VIP между серверами haproxy1 и haproxy2 

| узел     | ip адрес      | службы                    |
|----------|---------------|---------------------------|
| pg0      | 192.168.0.90  | postgrsql, etcd, patroni  | 
| pg1      | 192.168.0.91  | postgresql, etcd, patroni | 
| pg2      | 192.168.0.92  | postgresql, etcd, patroni |
| haproxy1 | 192.168.0.100 | haproxy, keepalived       |
| haproxy2 | 192.168.0.102 | haproxy, keepalived       |
| grafana  | 192.168.0.101 | grafana, prometheus       |


## Устанавка Postgres 17

1. Выполняем установку и настройку Postgres 17 на каждом узле (pg0, pg1, pg2).

Устанавливаем Postgres 17:
```
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
sudo apt install postgresql-17
```

Задаем пароль для роли postgres и создаем пользователя replicator:
```
sudo -u postgres psql
\password

create user replicator replication login encrypted password '123';
```

Отрываем файл pg_hba.config на редактирование:
```
sudo nano /etc/postgresql/17/main/pg_hba.conf
```
Добавляем строки чтобы разрешить удаленные подключения:

```
host    all             all             0.0.0.0/0               scram-sha-256
host    replication     all             0.0.0.0/0               scram-sha-256
```

Отрываем файл postgresql.conf на редактирование:
```
sudo nano /etc/postgresql/17/main/postgresql.conf
```
Рзрешаем прослушивание для всех IP-адресов:
```
listen_addresses = '*'
```
Выполняем перезапуск ноды pg0:
```
systemctl restart postgresql
```

На ноде pg1 и pg2 удаляем содержимое каталога pgdata, поскольку оно будет отреплицировано с ноды pg0 при развертывании кластера:
```
sudo systemctl stop postgresql
sudo -i
rm -rf /var/lib/postgresql/17/main/*
```

Если не удалить каталог pgdata, то в дельнейшем получим ошибку репликации вида:
```
CRITICAL: system ID mismatch, node pg1 belongs to a different cluster: 7527332087491039359 != 7527757238893566933
```
Версии каталога pgdata должны совпадать на всех узлах. Узнать версию можно командой:
```
pg_controldata  /var/lib/postgresql/17/main/data
```

## Установка Установка ETCD

Скачиваем и распаковываем etcd:
```
cd /tmp
wget https://github.com/etcd-io/etcd/releases/download/v3.5.5/etcd-v3.5.5-linux-amd64.tar.gz
tar xzvf etcd-v3.5.5-linux-amd64.tar.gz
sudo mv /tmp/etcd-v3.5.5-linux-amd64/etcd* /usr/local/bin/
```

Создаем пользователя, от которого будет работать служба etcd:
```
sudo groupadd --system etcd
sudo useradd -s /sbin/nologin --system -g etcd etcd
```
Создаем каталоги etcd, меняем владельца и права:
```
mkdir /opt/etcd
mkdir /etc/etcd
mkdir /var/lib/etcd
chown -R etcd:etcd /opt/etcd /var/lib/etcd /etc/etcd
chmod -R 700 /opt/etcd/ /var/lib/etcd /etc/etcd
```

Далее создаем конфиги etcd. Для каждой ноды:
```
sudo nano /etc/etcd/etcd.conf
```
Записываем соответствующий ноде текст конфига:

[конфиг для pg0](res/etcd-pg0.conf)

[конфиг для pg1](res/etcd-pg1.conf)

[конфиг для pg2](res/etcd-pg2.conf)


Далее на каждой ноде делаем etcd службой:

```
sudo nano /etc/systemd/system/etcd.service
```
[содержание конфигурации](res/etcd.service)

Далее настраиваем автозапуск службы etcd и ее запускаем:
```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```
Проверяем статус службы:
```
systemctl status etcd
```

Для проверки можно посмотреть информацию о нодах кластера (и узнать, кто лидер):
```
ETCDCTL_API=2 etcdctl member list
```
и проверить здоровье нод:
```
etcdctl endpoint health --cluster -w table
```

## Установка Patroni

Устанавливаем пакеты для работы с Python:
```
apt -y install python3 python3-pip python3-dev python3-psycopg2 libpq-dev
```
Через PIP ставим пакеты Python:
```
pip3 install psycopg2 --break-system-packages
pip3 install psycopg2-binary --break-system-packages
pip3 install patroni --break-system-packages
pip3 install python-etcd --break-system-packages
```
Создаем каталог конфигов Patroni:
```
mkdir /etc/patroni/
```
Создаем файл конфигурации Patroni:
```
sudo nano /etc/patroni/patroni.yml
```

После этого первую ноду (pg0) нужно перезапустить:
```
systemctl restart postgresql
```

[Текст patroni.yml для первой ноды (pg0)](res/patroni-pg0.yml)
[Текст patroni.yml для второй ноды (pg1)](res/patroni-pg1.yml)
[Текст patroni.yml для третьей ноды (pg2)](res/patroni-pg2.yml)


Назначаем права на каждой ноде:
```
sudo chown postgres:postgres -R /etc/patroni
sudo chmod 700 /etc/patroni
sudo mkdir /var/lib/pgsql_stats_tmp
sudo chown postgres:postgres /var/lib/pgsql_stats_tmp
```

Чтобы проверить, все ли сделано правильно, на всех нодах выполняем команду
```
sudo -u postgres patroni /etc/patroni/patroni.yml
```
Каждый из серверов должен запуститься и написать, он в роли лидера или в роли secondary.

Определяем Patroni как службу (на всех трех нодах одинаково):
```
sudo nano /etc/systemd/system/patroni.service
```

[patroni.service](res/patroni.service)


Переводим Patroni в автозапуск, стартуем и проверяем:
```
sudo systemctl daemon-reload
sudo  systemctl enable patroni
sudo systemctl start patroni
systemctl status patroni
```

Посмотреть лог patroni можно командой:
```
sudo journalctl -f -n 10 -u patroni
```

Просмотреть состояние кластера можно командой
```
sudo patronictl -c /etc/patroni/patroni.yml list
```

```
+ Cluster: postgres-cluster (7527332087491039359) -+----+-----------+
| Member | Host         | Role         | State     | TL | Lag in MB |
+--------+--------------+--------------+-----------+----+-----------+
| pg0    | 192.168.0.90 | Leader       | running   |  3 |           |
| pg1    | 192.168.0.91 | Sync Standby | streaming |  3 |         0 |
| pg2    | 192.168.0.92 | Replica      | streaming |  3 |         0 |
+--------+--------------+--------------+-----------+----+-----------+
```

TL - идентификатор временной шкалы
это целочисленный идентификатор, связанный с конкретной временной шкалой кластера базы данных. 
Временные шкалы критически важны для восстановления на определенный момент времени (PITR) и репликации.

```
SELECT timeline_id FROM pg_control_checkpoint();
```

Для failover:
```
sudo patronictl -c /etc/patroni/patroni.yml failover
```

настройки кластера в etcd можно посмтореть командой:

```
ETCDCTL_API=2 etcdctl ls /service/postgres-cluster
```

```
/service/postgres-cluster/config
/service/postgres-cluster/history
/service/postgres-cluster/initialize
/service/postgres-cluster/leader
/service/postgres-cluster/members
/service/postgres-cluster/status
/service/postgres-cluster/sync
```

```
ETCDCTL_API=2 etcdctl get /service/postgres-cluster/members/pg0
```

```
{"conn_url":"postgres://192.168.0.90:5432/postgres","api_url":"http://192.168.0.90:8008/patroni","state":"running","role":"primary","version":"4.0.6","xlog_location":134320400,"timeline":4}
```

Посмотрим некоторые настройки PostgreSQL:

```
select * from pg_settings
where name = 'in_hot_standby'
```

Убедимся, что значение настройки `in_hot_standby` у лидера off, у реплики on

```
select * from pg_settings
where name = 'transaction_read_only'
```
Значение у лидера off, у реплики on

### Как динамически изменить кофигурацию PostgreSQL

С помощью команды вносим измение в динамическую конфигурацию:
```
sudo patronictl -c /etc/patroni/patroni.yml edit-config
```
Парметру `maintenance_work_mem`, присвоим значение  `320MB`

На каждом узле patroni убедимся, что параметр maintenance_work_mem 320MB
```
sudo -u postgres psql
select * from pg_file_settings where name='maintenance_work_mem';
```

Patroni сделал reloading. Но Patroni не будет автоматически перезагружать кластер. 

Если меняем параметр, который требует перезапуска PostrgreSQL, то выполнить команду:
```
sudo patronictl -c /etc/patroni/patroni.yml restart postgres-cluster
```

Другой способ изменить конфигурацию - Rest API.
Пример запроса:
```
curl -u "patroni:123"  -X PUT -H "Content-Type: application/json" -d '{"shared_buffers": "1024MB"}' http://192.168.0.90:8008/config
```

Конфиг сохраняется в etcd, посмотреть можно командой:
```
ETCDCTL_API=2 etcdctl get /service/postgres-cluster/config
```

при изменении значений loop_wait , retry_timeout или ttl необходимо следовать правилу:
```
loop_wait + 2 * retry_timeout <= ttl
```
https://patroni.readthedocs.io/en/latest/dynamic_configuration.html


### Создание базы данных

Подключимся к лидеру и создадим пользователя и базу данных:

```
CREATE USER user_people WITH LOGIN NOSUPERUSER INHERIT NOCREATEDB NOCREATEROLE NOREPLICATION PASSWORD '123';

CREATE DATABASE people WITH OWNER = user_people
TEMPLATE = template0
ENCODING = 'UTF8'
TABLESPACE = pg_default
CONNECTION LIMIT = -1;
```

Создадим схему и таблицу в ней

```
/c people;

create schema people;

create table people.persons (
    person_id      int GENERATED ALWAYS AS IDENTITY,
    first_name     varchar    not null,
    patronymic     varchar, 
    surname        varchar,
    birthday       date, 
    constraint pk_persons primary key (person_id)
);
```

Наполним таблицу тестовыми данными
```
insert into people.persons(first_name, surname, birthday)
values ('Борис', 'Губкин', '1980-01-02'),
  ('Костя', 'Суперов', '1990-07-20'),
  ('Люда', 'Баг', '1990-07-20');
```
Подключаемся к любой реплике проверяем репликацию:
```
select * from people.persons;
```

### Автоматический failover

Останавливаем patroni на лидере:

```
service patroni stop
```

Убеждаемся, что выбран новый лидер:
```
sudo patronictl -c /etc/patroni/patroni.yml list
```

### Возможные ошибки и способы решения

Если реплика не стратует. Ошибка восстановления реплики pg_rewind.
В логе (SELECT  pg_current_logfile();) postgreSQL запись:

```
requested timeline 12 does not contain minimum recovery point 0/240000D8 on timeline 11
```

Решается переинициализацией члена кластера:

```
sudo patronictl -c /etc/patroni/patroni.yml reinit postgres-cluster pg0 
```

## Установка haproxy

Распространенной реализацией принципа высокой доступности (HA - high availability) в среде PostgreSQL является использование прокси-сервера: 
вместо прямого подключения к серверу базы данных приложение подключается к прокси-серверу, 
который перенаправляет запрос к PostgreSQL.

Произведем установку HAProxy и настройку балансировки нагрузки на чтение между репликами

Устанавливаем HAProxy:
```
apt -y install haproxy
```
Сохраняем исходный файл конфигурации:
```
mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.origin
```
Создаем новый файл конфигурации:
```
sudo nano /etc/haproxy/haproxy.cfg
```
И записываем в него текст: [haproxy.cfg](res/haproxy.cfg)

Далее перезагружаем HAProxy:
```
sudo systemctl restart haproxy
```
и проверяем работоспособность:
```
sudo systemctl status haproxy
```

Можно проверить состояние HAProxy по адресу http://192.168.0.102:7000/

https://habr.com/ru/companies/otus/articles/753294/

## Установка keepalived

На двух серверах Устанавливаем keepalived:
```
sudo apt-get install keepalived
```

В файл /etc/sysctl.conf и добавляем строки:
```
net.ipv4.ip_forward=1
net.ipv4.ip_nonlocal_bind=1
```
Для проверки выполняем:
```
sudo sysctl -p
```

Откарываем на редактирование файл keepalived.conf:
```
sudo nano /etc/keepalived/keepalived.conf
```

На основном сервере записываем конфигурацию: [keepalived-primary.cfg](res/keepalived-primary.cfg)
На дополнительном сервере записываем конфигурацию: [keepalived-standby.cfg](res/keepalived-standby.cfg)

Далее перезагружаем keepalived:
```
sudo systemctl restart keepalived
```
и проверяем работоспособность:
```
sudo systemctl status keepalived
```


## Нагрузочное тестирование

Выполнили инциализацию pgbench:

```
sudo -u postgres pgbench -i postgres -p 5432

user@vubuntu-22:~/sysbench-tpcc$ sudo -u postgres pgbench -i postgres -p 5432
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

```
sudo -u postgres pgbench -S -c 8 -P 6 -T 60 -U postgres postgres -p 5432
```
