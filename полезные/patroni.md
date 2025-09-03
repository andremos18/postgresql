
grafana
https://grafana.com/grafana/download?pg=oss-graf&plcmt=hero-btn-1

http://192.168.0.101:9090/targets
http://192.168.0.101:3000/login





Запустите keepalived командой /etc/init.d/keepalived start. Проверить работоспособность сервиса можно командой: sudo service keepalived status, а команой ip addr show ens33 (имя сетевого интерфейса) можно проверить поднялся ли виртуальный IP адрес.
Для перезагрузки службы используйте команду: sudo service keepalived restart



sudo apt install psmisc



sudo -u postgres rm -rf /var/lib/postgresql/17/main

sudo -u postgres mkdir /var/lib/postgresql/17/main

ls -l /var/lib/postgresql/17/main

sudo chown -R postgres:postgres /var/lib/postgresql/17/main

sudo nano /etc/etcd/etcd.conf
sudo nano /etc/systemd/system/etcd.service

etcdctl version
pg_controldata  /var/lib/postgresql/17/main/data


sudo nano /etc/patroni/patroni.yml

Далее — назначаем права на каждой ноде:

sudo chown postgres:postgres -R /etc/patroni
sudo chmod 700 /etc/patroni
sudo mkdir /var/lib/pgsql_stats_tmp
sudo chown postgres:postgres /var/lib/pgsql_stats_tmp

Чтобы проверить, все ли сделано правильно, на всех нодах выполняем команду
sudo -u postgres patroni /etc/patroni/patroni.yml

sudo nano /etc/systemd/system/patroni.service

Делаем службой

sudo systemctl daemon-reload
...



Ошибка
CRITICAL: system ID mismatch, node pg1 belongs to a different cluster: 7527332087491039359 != 7527757238893566933

pg0@pg0:/usr/lib/postgresql/17/bin$ sudo ./pg_controldata  /var/lib/postgresql/17/main/



etcdctl ls / service/postgres-cluster/config

ETCDCTL_API=3 etcdctl get / service/postgres-cluster/config --prefix

ETCDCTL_API=2 etcdctl ls / service/postgres-cluster/config



In PostgreSQL, the pg_settings view provides information about the server's run-time parameters. The source column within this view indicates where a particular setting's value originated. When the source column shows 'override', it signifies that the parameter's value was set at a level that takes precedence over other common configuration methods.
Specifically, a source of 'override' typically refers to parameter settings that were established during the server's initialization, either through:
Command-line options passed to the postgres command during server startup:
For example, using -c name=value or --name=value when starting the PostgreSQL server. These settings override values found in postgresql.conf or set via ALTER SYSTEM.
Settings derived from the initdb process:
These include certain default character encoding and locale settings that are established when the data directory is initialized.
These 'override' settings are generally considered more fundamental and are less easily changed dynamically compared to settings from other sources like postgresql.conf, ALTER SYSTEM, or per-session SET commands. They represent a foundational layer of configuration for the PostgreSQL instance.

select * from pg_settings where source = 'override'

select * from pg_settings
where name='transaction_read_only'