# Задание 3
## Установка и настройка PostgreSQL
> [!NOTE]
> ДЗ выполнялось на машине с Win 10. Был установлен WSL и развернута виртуалка с Ubuntu 22.04
### Установка postgres
Устанавливаем пакет:
```
sogris@HOME:sudo apt install postgresql-15
```
Проверяем кластер после установки:
```
sogris@HOME:/mnt$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory     Log file
15  main    5432 online postgres /mnt/pgsql/15/main /var/log/postgresql/postgresql-15-main.log
```
Заходим в клиент psql и создаем тестовую таблицу:
```
sogris@HOME:/mnt$ sudo -u postgres psql
postgres=# create table test(c1 text);
postgres=# insert into test values('1');
postgres=# select * from test;
 c1
----
 1
(1 row)
```
Таблица готова, останвливаем кластер:
```
sogris@HOME:/mnt$ sudo -u postgres pg_ctlcluster 15 main stop
```
Монтируем диск в виртуалку (учитывая что мы под Win, команда выглядит посложнее)
```
sogris@HOME:sudo mount -o rwx,uid=109,gid=119,umask=027 -t drvfs D: /mnt/pgsql
```
> [!IMPORTANT]
> Выставляем права на папки в 750, иначе postgres будет ругаться и откажется запускать кластер, делаем владельцем целевой папки также postgres.
Переносим директорию с данными СУБД:
```
sogris@HOME:mv /var/lib/postgresql/15 /mnt/pgsql
```
После этой манипуляции попытка запустить кластер приводит к ошибке:
```
sogris@HOME:/etc/postgresql/15main$ sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
> [!IMPORTANT]
> Кластер перестал работать, т.к. мы унесли директорию с данными из папки по умолчанию. Нужно перенастроить СУБД.
```
sogris@HOME: sudo vi /etc/postgresql/15/main/postgresql.conf
```
В файле находим строку вида - *data_directory = ' /var/lib/postgresql/15'           # use data in another directory* и актуализируем ее на **/mnt/pgsql/15/main**
После такой правки кластер успешно стартует и можно убедиться запросом что данные в таблице *test* сохранились.
