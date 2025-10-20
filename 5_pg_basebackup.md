## Пример восстановления БД на определенное время утилитой pg_basebackup
### Описание тестового стенда:
Используются виртуальные машины VM VirtualBox в подсети `192.168.0.0/24`:
1. NFS-сервер
- ОС - `Ubuntu 24.04`
- ip-адрес - 192.168.0.150
- Примонтированный диск `/dev/sdb1  98G  5.9G  88G  7% /data`
```
sudo vim /etc/exports
/data/postgresql_master 192.168.0.0/24(rw,sync,no_subtree_check)
/data/postgresql_replica 192.168.0.0/24(rw,sync,no_subtree_check)
/data/restore_host 192.168.0.0/24(rw,sync,no_subtree_check)
/data/backups 192.168.0.0/24(rw,sync,no_subtree_check)
```
2. Мастер: 
- ОС - `Ubuntu 24.04`
- ip-адрес - 192.168.0.151
- СУБД - PostgreSQL 17
- Примонтированный диски:
```
192.168.0.150:/data/postgresql_master   98G  5.9G   88G   7% /data
192.168.0.150:/data/backups             98G  5.9G   88G   7% /backup
```
3. Реплика: 
- ОС - `Ubuntu 24.04`
- ip-адрес - 192.168.0.152
- СУБД - PostgreSQL 17
- Примонтированный диски:
```
192.168.0.150:/data/postgresql_replica   98G  5.9G   88G   7% /data
192.168.0.150:/data/backups              98G  5.9G   88G   7% /backup
```
4. Хост для восстановления данных:
- ОС - `Ubuntu 24.04`
- ip-адрес - 192.168.0.153
- СУБД - PostgreSQL 17
- Примонтированный диски:
```
192.168.0.150:/data/restore_host    98G  5.9G   88G   7% /data
192.168.0.150:/data/backups         98G  5.9G   88G   7% /backup
```
### Подготовка хостов
Имена хостов: `sudo vim /etc/hosts`
```
127.0.0.1 localhost
127.0.1.1 restore-hosts # заменить на текущее имя хоста

192.168.0.150     nfs.local nfs-server
192.168.0.151     pg-node1.local pg-node1
192.168.0.152     pg-node2.local pg-node2
192.168.0.153     restore-host restore-host
```
#### Настройка пользователей на NFS-сервере
На NFS-сервер требуется установить пользователя `postgres`:
```
sudo groupadd postgres
sudo useradd --system --no-create-home --shell /bin/bash -g postgres postgres
```
Если создать пользователя и группу `postgres:postgres` на NFS-сервере то его uid может не совпадать с номером пользователя и группы на клиенте:
```
nazrinrus@pg-node1:/data$ id postgres
uid=112(postgres) gid=111(postgres) groups=111(postgres),110(ssl-cert)

nazrinrus@nfs-server:~$ id postgres
uid=999(postgres) gid=988(postgres) groups=988(postgres)
```
В таком случае нужно выровнить номера, заново установить владельца, перемонтировать и перезагрузить сервис
```
nazrinrus@nfs-server:~$ sudo groupmod -g 111 postgres
nazrinrus@nfs-server:~$ sudo usermod -u 112 postgres
nazrinrus@nfs-server:~$ sudo chown postgres:postgres /data/postgresql_master/postgresql
nazrinrus@nfs-server:~$ sudo exportfs -a
nazrinrus@nfs-server:~$ sudo systemctl reload nfs-kernel-server
```
#### Установка PostgreSQL на хостах
Добавляю официальный PostgreSQL репозиторий:
```
# Import the repository signing key:
sudo apt install curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

# Create the repository configuration file:
. /etc/os-release
sudo sh -c "echo 'deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $VERSION_CODENAME-pgdg main' > /etc/apt/sources.list.d/pgdg.list"

# Update the package lists:
sudo apt update
```
Установка пакетов PostgreSQL:
```
PGVERSION=17 
sudo apt install postgresql-${PGVERSION?} postgresql-${PGVERSION?}-repack postgresql-${PGVERSION?}-dbgsym postgresql-client-${PGVERSION?} unzip -y
```
Выданы права пользователю `postgres` на директорию `/data/postgresql`
```
sudo chown -R postgres:postgres /data/postgresql
```
Создание директории на виртуальном диске для WAL-архива и бэкапа, от имени `postgres`
```
mkdir /backup/db_postgresql/archive_wal /backup/db_postgresql/fullbackup
```
#### Конфигурирование PostgreSQL
Мастер. `vim /etc/postgresql/17/main/postgresql.conf`
```
listen_addresses = '*'
wal_level = replica
archive_mode = on # архив WAL для восстановления на определенный момент времени
# директория для архива указывается параметром archive_library = '' или командой ОС в параметре archive_command = ''(в данном случае командой ОС)
archive_command = 'test ! -f /backup/db_postgresql/archive_wal/%f && cp %p /backup/db_postgresql/archive_wal/%f' # из документации
```
Реплика. По умолчанию параметр, отвечающий за доступ к читающим запросам в режиме восстановления (а реплика всегда в режиме 
восстановления) `hot_standby` должен быть `on` (`hot_standby = on`), но стоит проверить и исправить:
```
sudo -su postgres psql -c "SHOW hot_standby;"
 hot_standby 
-------------
 on
(1 row)
```
Открыть доступ для внешнего подключения:
```
listen_addresses = '*'
```
Перезапуск:`sudo systemctl restart postgresql`
#### Задание пароля пользователю `postgres` и создание пользователя `replica` (на мастере):
```
nazrinrus@pg-node1:~$ sudo -iu postgres
postgres@pg-node1:~$ psql
postgres=# ALTER USER postgres WITH PASSWORD 'postgres';
postgres=# CREATE USER replica WITH PASSWORD 'replica' REPLICATION;
```
#### Редактирование файла `sudo vim /etc/postgresql/17/main/pg_hba.conf` (на всех хостах):
```
host    all             postgres        192.168.0.0/24          scram-sha-256
host    replication     replica        192.168.0.0/24           scram-sha-256
```
Применение изменений: `psql --command='select pg_reload_conf();'`
```
postgres@pg-node1:~$ psql --command='select pg_reload_conf();'
 pg_reload_conf 
----------------
 t
(1 строка)
```
#### Развернуть тестовую БД из дампа :
Создать тестовую БД, для загрузки в нее дампа:
```
psql -h 192.168.0.151 -U postgres -d postgres -W -c 'CREATE DATABASE demo;'
```
Развернуть дамп:
```
sudo wget https://edu.postgrespro.ru/demo-medium.zip -O /tmp/demo-medium.zip
sudo unzip /tmp/demo-medium.zip -d /tmp/
sudo psql -U postgres -h 127.0.0.1 -d demo < /tmp/demo-medium-20170815.sql
```
Сгенерировать множество записей для генерации WAL-файлов
```
demo=# CREATE TABLE test_table (col1 INT);
CREATE TABLE
demo=# INSERT INTO test_table SELECT random()*100.00 FROM generate_series(1, 1000000);
INSERT 0 1000000

postgres@pg-node1:~$ ls -la /backup/db_postgresql/archive_wal/
total 278536
drwxrwxr-x 2 postgres postgres     4096 Oct 20 12:48 .
drwxrwxrwx 5 postgres postgres     4096 Oct 20 12:58 ..
-rw------- 1 postgres postgres 16777216 Oct 20 12:23 000000010000000000000025
-rw------- 1 postgres postgres 16777216 Oct 20 12:23 000000010000000000000026
-rw------- 1 postgres postgres 16777216 Oct 20 12:23 000000010000000000000027
-rw------- 1 postgres postgres 16777216 Oct 20 12:23 000000010000000000000028
```
#### Реплика. Удаление директории с данными, созданной при автоматической инициализации во время установки
```
sudo systemctl stop postgresql

sudo -u postgres rm -rf /data/postgresql/*
```
#### Копирование директории с данными с мастера
```
sudo -u postgres pg_basebackup -h 192.168.0.151 -D /data/postgresql -U replica -P -v -R -c fast

sudo systemctl start postgresql 
```
- `h 192.168.0.151` — адрес главного сервера;
- `D` — папка, куда нужно положить бэкап.
- `U` — пользователь для подключения.
- `P` — запрашивает ввод пароля.
- `v` — выводит подробный лог выполнения команды.
- `R` — создаёт в папке с базами данных файл `standby.signal`. Это маркер для сервера PostgreSQL, что нужно запуститься в резервном режиме.

Проверить, что экземпляр в режиме `recovery` и содержимое тестовой таблицы
```
sudo -iu postgres psql --command='SELECT pg_is_in_recovery();'
 pg_is_in_recovery 
-------------------
 t
(1 row)
```
### Работа с резервным копированием/восстановлением
Полное резервное копирование утилитой pg_BaseBackup является точкой отсчета для инкрементального копирования (появилось в 17 версии PostgreSQL)
#### Полное резервное копирование pg_BaseBackup (на реплике)
от имени `postgres`
```
pg_basebackup -v -D /backup/db_postgresql/fullbackup
```
В результате, появился файл-метка `/data/postgresql/pg_wal/цифры.backup`:
```
START WAL LOCATION: 0/6000028 (file 000000010000000000000006)
STOP WAL LOCATION: 0/6000120 (file 000000010000000000000006)
CHECKPOINT LOCATION: 0/6000080
BACKUP METHOD: streamed
BACKUP FROM: primary
START TIME: 2025-10-20 10:15:00 UTC
LABEL: pg_basebackup base backup
START TIMELINE: 1
STOP TIME: 2025-10-20 10:15:00 UTC
STOP TIMELINE: 1
```
#### Внесение изменений в БД после снятия полной резервной копии
На данный момент (сразу после резервного копирования) в таблице `test_table` 1000000 записей:
```
demo=# SELECT COUNT(1) FROM test_table;
  count  
---------
 1000000
(1 row)
```
После копирования было внесено еще 1 000 000 записей, не попавших в бэкап:
```
demo=# INSERT INTO test_table SELECT random()*100.00 FROM generate_series(1, 1000000);
INSERT 0 1000000
demo=# SELECT COUNT(1) FROM test_table;
  count  
---------
 2000000
(1 row)

demo=# SELECT now();
              now              
-------------------------------
 2025-10-20 14:42:30.084931+00
(1 row)
```
"Случайно" удаляем записи:
```
demo=# DELETE FROM test_table WHERE true;
DELETE 2000000
```
#### Восстановление данных
Часть данных имеется в бэкапе, часть в WAL-архиве, но воспроизвести надо не все WAL-файлы, а только до времени ошибки. 
В нашем случае это `2025-10-20 14:42:30.084931+00`

Копируем текущую директорию с данными:
```
sudo mkdir /backup/db_postgresql/old_data
sudo cp -r /data/postgresql /backup/db_postgresql/old_data
```
Копируем ранее созданную полную версию бэкапа в директорию с данными хоста `restore-host`:
```
sudo cp -a /backup/db_postgresql/fullbackup/. /data/postgresql
```
Очищаем каталог `/data/postgresql/pg_wal/*`
```
sudo rm -rf /data/postgresql/pg_wal/*
```
и добавляем в него wal-файлы старого каталога данных (они содержат ошибку, но мы будем восстанавливаться до ошибки)
```
sudo cp -a /backup/db_postgresql/old_data/postgresql/pg_wal/. /data/postgresql/pg_wal
```
эти файлы надо сложить с файлами из архива, что делает команда в параметре `restore_command` файла `vim /etc/postgresql/17/main/postgresql.conf`
```
restore_command = 'cp /backup/db_postgresql/archive_wal/%f "%p"'
recovery_target_time = '2025-10-20 14:42:30.084931+00' # указали на какое время должна восстановиться система
# так же можно указать на какой транзакции остановить восстановление в параметре recovery_target_xid = ''
```
создаем файл `sudo touch /data/postgresql/recovery.signal` для указания системе запуститься в режиме восстановления. 
Файл сам удалится при удачном восстановлении.

Меняем права доступа и владельца директории с данными:
```
sudo chown -R postgres:postgres /data/postgresql
sudo chmod -R 750 /data/postgresql
```
Запуск PostgreSQL
```
sudo systemctl start postgresql
```
Сервер запущен в режиме чтения, проверяем целостность данных:
```
demo=# SELECT COUNT(1) FROM test_table;
  count  
---------
 2000000
(1 row)
```
Открыть доступ на запись: `SELECT pg_wal_replay_resume();`

**в конфигурационном файле параметры `restore_command` и `recovery_target_time` можно закомментировать, но в любом случае 
они будут игнорироваться пока нет файла `recovery.signal`. Так же после восстановления следует сразу сделать полный бэкап, 
чтобы подчистить WAL-файлы в архиве, содержащие ошибки**
