## Пример восстановления БД на определенное время утилитой pg_basebackup
### Описание стенда:
1. Сервер: 
- ip-адрес - 192.168.0.108
- СУБД - PostgreSQL 17
- Примонтированный диск `/dev/sdb1  9,8G   63M  9,2G   1% /mnt/data`
- ОС - `Ubuntu 20.04`
2. Реплика: 
- ip-адрес - 192.168.0.109
- СУБД - PostgreSQL 17
- ОС - `Ubuntu 20.04`
### Подготовка сервера
#### SSH подключение:
- Очистка данных о старых хостах: `ssh-keygen -f "/home/nazrinrus/.ssh/known_hosts" -R "192.168.0.108"`
- Импорт ключа `ssh-copy-id nazrinrus@192.168.0.108`
#### Установка PostgreSQL
Добавляем официальный PostgreSQL репозиторий:
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
sudo apt install postgresql-${PGVERSION?} postgresql-${PGVERSION?}-repack postgresql-${PGVERSION?}-dbgsym postgresql-client-${PGVERSION?}
```
Выданы права пользователю `postgres` на директорию `/mnt/data/`
```
sudo chown -R postgres:postgres /mnt/data/
```
Создание директории на виртуальном диске для WAL-архива и бэкапа, от имени `postgres`
```
mkdir /mnt/data/archive_wal /mnt/data/fullbackup
```
#### Конфигурирование PostgreSQL
Мастер `vim /etc/postgresql/17/main/postgresql.conf`
```
listen_addresses = '*'
wal_level = replica
archive_mode = on # архив WAL для восстановления на определенный момент времени
# директория для архива указывается параметром archive_library = '' или командой ОС в параметре archive_command = ''(в данном случае командой ОС)
archive_command = 'test ! -f /mnt/data/archive_wal/%f && cp %p /mnt/data/archive_wal/%f' # из документации
```
Перезапуск:`sudo systemctl restart postgresql`
#### Задание пароля пользователю `postgres` и создание пользователя `replica`:
```
nazrinrus@otus-practice3:~$ sudo -iu postgres
postgres@otus-practice3:~$ psql
postgres=# ALTER USER postgres WITH PASSWORD 'postgres';
postgres=# CREATE USER replica WITH PASSWORD 'replica' REPLICATION;
```
#### Редактирование файла `sudo vim /etc/postgresql/17/main/pg_hba.conf`:
```
host    all             postgres        192.168.0.0/24          scram-sha-256
host    replication     replica        192.168.0.0/24           scram-sha-256
```
Применение изменений: `psql --command='select pg_reload_conf();'`
```
postgres@otus-practice3:~$ psql --command='select pg_reload_conf();'
 pg_reload_conf 
----------------
 t
(1 строка)
```
#### Развернуть тестовую БД из дампа :
Создать тестовую БД, для загрузки в нее дампа:
```
psql -h 192.168.0.108 -U postgres -d postgres -W -c 'CREATE DATABASE test_db;'
```
Развернуть дамп:
```
pg_restore -h 192.168.0.108 -U postgres -d test_db /home/nazrinrus/dumps/hr_poc.dump

test_db=# \dt+ hr_poc.*
                                        Список отношений
 Схема  |     Имя     |   Тип   | Владелец |  Хранение  | Метод доступа |   Размер   | Описание 
--------+-------------+---------+----------+------------+---------------+------------+----------
 hr_poc | customers   | таблица | postgres | постоянное | heap          | 16 kB      | 
 hr_poc | departments | таблица | postgres | постоянное | heap          | 8192 bytes | 
 hr_poc | employees   | таблица | postgres | постоянное | heap          | 40 kB      | 
 hr_poc | jobs        | таблица | postgres | постоянное | heap          | 8192 bytes | 
 hr_poc | locations   | таблица | postgres | постоянное | heap          | 8192 bytes | 
 hr_poc | order_items | таблица | postgres | постоянное | heap          | 8192 bytes | 
 hr_poc | orders      | таблица | postgres | постоянное | heap          | 8192 bytes | 
 hr_poc | products    | таблица | postgres | постоянное | heap          | 8192 bytes | 
(8 строк)
```
Сгенерировать множество записей для генерации WAL-файлов
```
test_db=# CREATE TABLE test_table (col1 INT);
CREATE TABLE
test_db=# INSERT INTO test_table SELECT random()*100.00 FROM generate_series(1, 1000000);
INSERT 0 1000000

postgres@otus-practice3:~$ ls -la /mnt/data/archive_wal
total 65544
drwxrwxr-x 2 postgres postgres     4096 мая 27 10:01 .
drwxr-xr-x 6 postgres postgres     4096 мая 27 08:50 ..
-rw------- 1 postgres postgres 16777216 мая 27 10:01 000000010000000000000001
-rw------- 1 postgres postgres 16777216 мая 27 10:01 000000010000000000000002
-rw------- 1 postgres postgres 16777216 мая 27 10:01 000000010000000000000003
-rw------- 1 postgres postgres 16777216 мая 27 10:01 000000010000000000000004
```
### Работа с резервным копированием/восстановлением
Полное резервное копирование утилитой pg_BaseBackup является точкой отсчета для инкрементального копирования (появилось в 17 версии PostgreSQL)
#### Полное резервное копирование pg_BaseBackup
от имени `postgres`
```
pg_basebackup -v -D /mnt/data/fullbackup
```
В результате, появился файл-метка `/var/lib/postgresql/17/main/pg_wal/цифры.backup`:
```
START WAL LOCATION: 0/6000028 (file 000000010000000000000006)
STOP WAL LOCATION: 0/6000120 (file 000000010000000000000006)
CHECKPOINT LOCATION: 0/6000080
BACKUP METHOD: streamed
BACKUP FROM: primary
START TIME: 2025-05-27 10:15:00 UTC
LABEL: pg_basebackup base backup
START TIMELINE: 1
STOP TIME: 2025-05-27 10:15:00 UTC
STOP TIMELINE: 1
```
#### Внесение изменений в БД после снятия полной резервной копии
На данный момент (сразу после резервного копирования) в таблице `test_table` 1100 000 записей:
```
test_db=# SELECT COUNT(1) FROM test_table;
  count  
---------
 1100000
(1 строка)
```
После копирования было внесено еще 1000 000 записей, не попавших в бэкап:
```
test_db=# INSERT INTO test_table SELECT random()*100.00 FROM generate_series(1, 1000000);
INSERT 0 1000000
test_db=# SELECT COUNT(1) FROM test_table;
  count  
---------
 2100000
(1 строка)

test_db=# SELECT now();
              now              
-------------------------------
 2025-05-27 10:29:03.923768+00
(1 строка)
```
"Случайно" удаляем записи:
```
test_db=# DELETE FROM test_table WHERE true;
DELETE 2100000
```
#### Восстановление данных
Часть данных имеется в бэкапе, часть в WAL-архиве, но воспроизвести надо не все WAL-файлы, а только до времени ошибки. 
В нашем случае это `2025-05-27 10:29:03.923768+00`
```
sudo systemcrl stop postgresql
```
На всякий случай копируем текущую директорию с данными:
```
sudo mkdir /mnt/data/old_data
sudo mv /var/lib/postgresql/17/main /mnt/data/old_data
sudo mkdir /var/lib/postgresql/17/main
```
Копируем ранее созданную полную версию бэкапа в директорию с данными:
```
sudo cp -a /mnt/data/fullbackup/. /var/lib/postgresql/17/main
```
Очищаем каталог `/var/lib/postgresql/17/main/pg_wal/*`
```
sudo rm -rf /var/lib/postgresql/17/main/pg_wal/*
```
и добавляем в него wal-файлы старого каталога данных (они содержат ошибку, но мы будем восстанавливаться до ошибки)
```
sudo cp -a /mnt/data/old_data/main/pg_wal/. /var/lib/postgresql/17/main/pg_wal
```
эти файлы надо сложить с файлами из архива, что делает команда в параметре `restore_command` файла `vim /etc/postgresql/17/main/postgresql.conf`
```
restore_command = 'cp /mnt/data/archive_wal/%f "%p"'
recovery_target_time = '2025-05-27 10:29:03.923768+00' # указали на какое время должна восстановиться система
# так же можно указать на какой транзакции остановить восстановление в параметре recovery_target_xid = ''
```
создаем файл `sudo touch /var/lib/postgresql/17/main/recovery.signal` для указания системе запуститься в режиме восстановления. 
Файл сам удалится при удачном восстановлении.

Меняем права доступа и владельца директории с данными:
```
sudo chown -R postgres:postgres /var/lib/postgresql/17/main/
sudo chmod -R 750 /var/lib/postgresql/17/main/
```
Запуск PostgreSQL
```
sudo systemctl start postgresql
```
Сервер запущен в режиме чтения, проверяем целостность данных:
```
test_db=# SELECT COUNT(1) FROM test_table;
  count  
---------
 2100000
(1 строка)
```
Открыть доступ на запись: `SELECT pg_wal_replay_resume();`

**в конфигурационном файле параметры `restore_command` и `recovery_target_time` можно закомментировать, но в любом случае 
они будут игнорироваться пока нет файла `recovery.signal`. Так же после восстановления следует сразу сделать полный бэкап, 
чтобы подчистить WAL-файлы в архиве, содержащие ошибки**

### Настройка хоста-реплики
#### Установка пакетов на реплике аналогична мастеру

#### Настройка конфигурационного файла `/etc/postgresql/17/main/postgresql.conf`
По умолчанию параметр, отвечающий за дочтуп к читающим запросам в режиме восстановления (а реплика всегда в режиме 
восстановления) `hot_standby` должен быть `on` (`hot_standby = on`), но стоит проверить и исправить:
```
sudo -su postgres psql -c "SHOW hot_standby;"
 hot_standby 
-------------
 on
(1 row)
```
открыть доступ для других хостов:
```
listen_addresses = '*'
```
#### Удаление директории с данными, созданной при автоматической инициализации во время установки
```
sudo systemctl stop postgresql

sudo -u postgres rm -rf /var/lib/postgresql/17/main/
```
#### Копирование директории с данными с мастера
```
sudo -u postgres pg_basebackup -h 192.168.0.108 -D /var/lib/postgresql/17/main -U replica -P -v -R

sudo systemctl start postgresql 
```
- `h 10.10.2.78` — адрес главного сервера;
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

test_db=# SELECT COUNT(1) FROM test_table;
  count  
---------
 3100000
(1 строка)
```
#### Настройка снятия бэкапа с реплики
Для снятия бэкапа на реплике необходимо настроить конфигурационные файлы аналогично мастеру
#### Редактирование файла `sudo vim /etc/postgresql/17/main/pg_hba.conf`:
```
host    all             postgres        192.168.0.0/24          scram-sha-256
host    replication     replica        192.168.0.0/24           scram-sha-256
```
Применение изменений: `psql --command='select pg_reload_conf();'`
### Снятие бэкапа с реплики
На высоконагруженных системах, для снятия нагрузки с мастера, бэкапы снимаются с реплики, например:
```
rm -rf /mnt/data/fullbackup/*
pg_basebackup -h 192.168.0.109 -v -D /mnt/data/fullbackup -U replica -P
```
