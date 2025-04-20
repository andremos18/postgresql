## Цели
* создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
* переносить содержимое базы данных PostgreSQL на дополнительный диск
* переносить содержимое БД PostgreSQL между виртуальными машинами


Настроена виртуальная машина (ВМ) virtualbox с ubuntu.
Установлен postgresql 17.

Создана таблица test
```
create table test(c1 text);
```

Добавлена запись в таблицу:
```
insert into test values('1');
```

остановлен postgresql
```
sudo -u postgres pg_ctlcluster 17 main stop
```

Создан новый диск к ВМ

форматирование с созданием файловой системы ext4
```
mkfs.ext4 /dev/sdb
```

создали каталог /mnt/data
```
sudo mkdir /mnt/data
```

выполнили монтирование
```
sudo mount -t ext4 -o rw /dev/sdb /mnt/data
```

остановили кластер
```
sudo -u postgres pg_ctlcluster 17 main stop
```

перенесли каталог с данными
```
sudo mv /var/lib/postgresql/17 /mnt/data
```

запустили кластер
```
sudo -u postgres pg_ctlcluster 17 main start
```

Получили ошибку
```
Error: /var/lib/postgresql/17/main is not accessible or does not exist
```
Ошибка запуска, потому что перенесли каталог, содержащий данные и не сообщили об этом postgresql

исправляем в конфигурационом файле
```
/etc/postgresql/17/main/postgresql.conf
```
параметр
```
data_directory = '/var/lib/postgresql/17/main'
```
меняем на
```
data_directory = '/mnt/data/17/main'
```

Запускаем кластер
```
sudo -u postgres pg_ctlcluster 17 main start
```

Прверяем, что кластер запустился командой:
```
pg_lsclusters
```

Запускам psql
```
sudo -u postgres psql
```

выполняем
```
select * from test;
```

получаем содержимое запись, добавленную ранее

*
Создали еще одну виртуальную машину

удалили /var/lib/postgresql
```
sudo rm -r /var/lib/postgresql
```

Добавли к новой ВМ диск, созданный ранее в первой ВМ

создали каталог /mnt/data
```
sudo mkdir /mnt/data
```

выполнили монтирование
```
sudo mount -t ext4 -o rw /dev/sdb /mnt/data
```

исправляем в конфигурационом файле
```
/etc/postgresql/17/main/postgresql.conf
```
параметр
```
data_directory = '/var/lib/postgresql/17/main'
```
меняем на
```
data_directory = '/mnt/data/17/main'
```

запускаем
```
sudo -u postgres pg_ctlcluster 17 main start
```

Проверили командой
```
pg_lsclusters
```

Запускам psql
```
sudo -u postgres psql
```

выполняем
```
select * from test;
```
получаем содержимое запись, добавленную ранее
