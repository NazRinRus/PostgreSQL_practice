## Настройка дисков для Постгреса
### Цель:
- создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
- переносить содержимое базы данных PostgreSQL на дополнительный диск
- переносить содержимое БД PostgreSQL между виртуальными машинами
### Описание тестового стенда:
- Виртуальная машина на базе VirtualBox
- CPU - 2 ядра, ОЗУ - 4 ГБ
- ip - 192.168.0.102
- ОС Ubuntu 20.04
## Пошаговое описание:
#### создание виртуальной машины c Ubuntu 20.04 LTS (bionic):
1. Загружен образ ubuntu-20.04.6-live-server-amd64.iso с сайта ubuntu.com
2. Импорт публичного ключа на сервер: `ssh-copy-id nazrinrus@192.168.0.102`
#### Установка PostgreSQL 15:
```
sudo apt install curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

. /etc/os-release
sudo sh -c "echo 'deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $VERSION_CODENAME-pgdg main' > /etc/apt/sources.list.d/pgdg.list"

sudo apt update
PGVERSION=15
sudo apt install postgresql-${PGVERSION?} postgresql-${PGVERSION?}-repack postgresql-${PGVERSION?}-dbgsym postgresql-client-${PGVERSION?}
```
Проверка: `sudo -u postgres pg_lsclusters`:
```
nazrinrus@otus-practice3:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
### Настройка доступа с удаленного хоста:
#### Задание пароля пользователю `postgres`:
```
nazrinrus@otus-practice3:~$ sudo -iu postgres
postgres@otus-practice3:~$ psql
postgres=# ALTER USER postgres WITH PASSWORD '*****'; # Если забуду - стандартный для тестовых стендов
```
#### Редактирование файла `sudo vim /etc/postgresql/15/main/postgresql.conf`:
```
listen_addresses = '*'     # what IP address(es) to listen on;
```
Перезапуск:`sudo systemctl restart postgresql`
#### Редактирование файла `sudo vim /etc/postgresql/15/main/pg_hba.conf`:
```
host    all             postgres        192.168.0.0/24          scram-sha-256
```
Применение изменений: `psql --command='select pg_reload_conf();'`
```
postgres@otus-practice3:~$ psql --command='select pg_reload_conf();'
 pg_reload_conf 
----------------
 t
(1 строка)
```
### Развернуть тестовую БД из дампа:
Создать тестовую БД, для загрузки в нее дампа:
```
psql -h 192.168.0.102 -U postgres -d postgres -W -c 'CREATE DATABASE test_db;'
```
Развернуть дамп:
```
pg_restore -h 192.168.0.102 -U postgres -d test_db /home/nazrinrus/dumps/hr_poc.dump

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
### Остановить postgres `sudo -u postgres pg_ctlcluster 15 main stop`
```
nazrinrus@otus-practice3:~$ sudo -u postgres pg_ctlcluster 15 main status
pg_ctl: сервер не работает
```
### Создать новый диск объемом 10 ГБ и добавить к виртуальной машине:
Создан виртуальный диск и подключен к виртуальной машине:
```
nazrinrus@otus-practice3:~$ lsblk
NAME                      MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0                       7:0    0 49,9M  1 loop /snap/snapd/18357
loop1                       7:1    0 91,9M  1 loop /snap/lxd/24061
loop2                       7:2    0 63,3M  1 loop /snap/core20/1828
sda                         8:0    0   25G  0 disk 
├─sda1                      8:1    0    1M  0 part 
├─sda2                      8:2    0    2G  0 part /boot
└─sda3                      8:3    0   23G  0 part 
  └─ubuntu--vg-ubuntu--lv 253:0    0 11,5G  0 lvm  /
sdb                         8:16   0   10G  0 disk 
sr0                        11:0    1 1024M  0 rom  
```
### Инициализация диска и монтирование в `/mnt/data`:
#### Создание раздела на диске `sdb`, утилитой `fdisk`:
```
sudo fdisk  /dev/vdb
Command (m for help): g
Created a new GPT disklabel (GUID: 932E5A2D-D571-5D4B-81A4-A8875113E109).

Command (m for help): n
Partition number (1-128, default 1): 
First sector (2048-20971486, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-20971486, default 20971486): 

Created a new partition 1 of type 'Linux filesystem' and of size 10 GiB.

Command (m for help): p
Disk /dev/sdb: 10 GiB, 10737418240 bytes, 20971520 sectors
Disk model: VBOX HARDDISK   
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 932E5A2D-D571-5D4B-81A4-A8875113E109

Device     Start      End  Sectors Size Type
/dev/sdb1   2048 20971486 20969439  10G Linux filesystem

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

nazrinrus@otus-practice3:~$ lsblk -f
NAME                      FSTYPE      LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINT
loop0                     squashfs                                                       0   100% /snap/snapd/18357
loop1                     squashfs                                                       0   100% /snap/lxd/24061
loop2                     squashfs                                                       0   100% /snap/core20/1828
loop3                     squashfs                                                       0   100% /snap/snapd/24505
loop4                     squashfs                                                       0   100% /snap/core20/2571
loop5                     squashfs                                                       0   100% /snap/lxd/32662
sda                                                                                               
├─sda1                                                                                            
├─sda2                    ext4              4f7e2420-777e-4efb-984e-423ce028b6cd      1,7G     6% /boot
└─sda3                    LVM2_member       I1RHNf-ps9P-tvrK-AXAK-7O8b-3Q5g-q6S6Xg                
  └─ubuntu--vg-ubuntu--lv ext4              a5b51819-d7b7-46cc-aba7-de693953ead9      5,5G    46% /
sdb                                                                                               
└─sdb1                                                                                            
sr0                 
```
#### Создание файловой системы ext4 на разделе /dev/sdb1:
```
sudo mkfs.ext4 /dev/sdb1
```
#### Создание точки монтирования `/mnt/data`:
```
sudo mkdir /mnt/data
sudo mount /dev/sdb1 /mnt/data

nazrinrus@otus-practice3:~$ df -hT
Filesystem                        Type      Size  Used Avail Use% Mounted on
udev                              devtmpfs  1,9G     0  1,9G   0% /dev
tmpfs                             tmpfs     392M  1,1M  391M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv ext4       12G  5,2G  5,5G  49% /
tmpfs                             tmpfs     2,0G  1,1M  2,0G   1% /dev/shm
tmpfs                             tmpfs     5,0M     0  5,0M   0% /run/lock
tmpfs                             tmpfs     2,0G     0  2,0G   0% /sys/fs/cgroup
/dev/loop0                        squashfs   50M   50M     0 100% /snap/snapd/18357
/dev/sda2                         ext4      2,0G  111M  1,7G   7% /boot
/dev/loop1                        squashfs   92M   92M     0 100% /snap/lxd/24061
/dev/loop2                        squashfs   64M   64M     0 100% /snap/core20/1828
tmpfs                             tmpfs     392M     0  392M   0% /run/user/1000
/dev/loop3                        squashfs   51M   51M     0 100% /snap/snapd/24505
/dev/loop4                        squashfs   64M   64M     0 100% /snap/core20/2571
/dev/loop5                        squashfs   92M   92M     0 100% /snap/lxd/32662
/dev/sdb1                         ext4      9,8G   24K  9,3G   1% /mnt/data
```
Выдать права пользователю 'postgres' на директорию `/mnt/data`:
```
sudo chown -R postgres:postgres /mnt/data/

nazrinrus@otus-practice3:~$ sudo ls -lah /mnt/
total 12K
drwxr-xr-x  3 root     root     4,0K мая 12 12:21 .
drwxr-xr-x 20 root     root     4,0K мая 12 07:52 ..
drwxr-xr-x  3 postgres postgres 4,0K мая 12 12:17 data
```
#### Редактирование `/etc/fstab` для автоматического монтирования, при перезагрузке:
```
sudo cp /etc/fstab /etc/fstab_backup
sudo vim /etc/fstab
/dev/sdb1       /mnt/data       ext4    defaults        0       0
```
Проверка после перезагрузки хоста:
```
nazrinrus@otus-practice3:~$ df -hT
Filesystem                        Type      Size  Used Avail Use% Mounted on
udev                              devtmpfs  1,9G     0  1,9G   0% /dev
tmpfs                             tmpfs     392M  1,1M  391M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv ext4       12G  5,2G  5,5G  49% /
tmpfs                             tmpfs     2,0G  1,1M  2,0G   1% /dev/shm
tmpfs                             tmpfs     5,0M     0  5,0M   0% /run/lock
tmpfs                             tmpfs     2,0G     0  2,0G   0% /sys/fs/cgroup
/dev/sdb1                         ext4      9,8G   24K  9,3G   1% /mnt/data
/dev/loop1                        squashfs   92M   92M     0 100% /snap/lxd/24061
/dev/loop2                        squashfs   64M   64M     0 100% /snap/core20/1828
/dev/loop0                        squashfs   64M   64M     0 100% /snap/core20/2571
/dev/loop3                        squashfs   92M   92M     0 100% /snap/lxd/32662
/dev/sda2                         ext4      2,0G  111M  1,7G   7% /boot
/dev/loop4                        squashfs   51M   51M     0 100% /snap/snapd/24505
/dev/loop5                        squashfs   50M   50M     0 100% /snap/snapd/18357
tmpfs                             tmpfs     392M     0  392M   0% /run/user/1000
```
### Перенос содержимого `/var/lib/postgresql/15` в `/mnt/data`:
```
sudo -iu postgres
mv /var/lib/postgresql/15 /mnt/data
```
**На данном этапе, запустить PostgreSQL не получиться, так как в конфигурационном файле `/etc/postgresql/15/main/postgresql.conf` 
параметр `data_directory = '/var/lib/postgresql/15/main'` ссылается на ранее перемещенную директорию**
#### Редактирование конфигурационного файла под новую `data_directory`:
```
vim /etc/postgresql/15/main/postgresql.conf
# редактируем строку
data_directory = '/mnt/data/15/main'
```
#### Запуск сервера, проверка содержимого:
```
sudo -u postgres pg_ctlcluster 15 main start

nazrinrus@otus-practice3:~$ sudo -u postgres pg_ctlcluster 15 main status
pg_ctl: сервер работает (PID: 1351)
/usr/lib/postgresql/15/bin/postgres "-D" "/mnt/data/15/main" "-c" "config_file=/etc/postgresql/15/main/postgresql.conf"
nazrinrus@otus-practice3:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log

nazrinrus@nazrinrusPC:~/Рабочий стол/PGMech$ psql -h 192.168.0.102 -U postgres -d test_db -c "SELECT schemaname::regnamespace, tablename FROM pg_tables WHERE schemaname::regnamespace = 'hr_poc'::regnamespace;"
Пароль пользователя postgres: 
Секундомер включён.
 schemaname |  tablename  
------------+-------------
 hr_poc     | locations
 hr_poc     | departments
 hr_poc     | employees
 hr_poc     | jobs
 hr_poc     | customers
 hr_poc     | orders
 hr_poc     | order_items
 hr_poc     | products
(8 строк)

Время: 2,677 мс
```
### Задание со звездочкой *
Не удаляя существующий инстанс, сделайте новый. Поставьте на него PostgreSQL. Удалите файлы с данными из /var/lib/postgresql.
Перемонтируйте внешний диск, который сделали ранее от первой виртуальной машины, ко второй и запустите PostgreSQL на второй машине, 
так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.
#### Создание дополнительного инстанса:
Создан инстанс аналогичный первому:
- Виртуальная машина на базе VirtualBox
- CPU - 2 ядра, ОЗУ - 4 ГБ
- ip - 192.168.0.104
- ОС Ubuntu 20.04
#### Процесс установки PostgreSQL аналогичен предыдущему инстансу
#### Удаление стандартной директории данных:
```
sudo -u postgres pg_ctlcluster 15 main stop
sudo rm -fr /var/lib/postgresql/15
```
#### Отключение PostgreSQL на первом инстансе, размонтирование виртуального диска с данными
```
sudo -u postgres pg_ctlcluster 15 main stop
sudo umount /mnt/data
```
#### Создание директории и монтирование диска с данными на втором инстансе:
```
sudo mkdir /mnt/data
sudo mount /dev/sdb1 /mnt/data
```
#### Редактирование конфигурационного файла под новую `data_directory`:
```
vim /etc/postgresql/15/main/postgresql.conf
# редактируем строку
data_directory = '/mnt/data/15/main'
```
#### Запуск PostgreSQL на втором инстансе:
```
sudo -u postgres pg_ctlcluster 15 main start
```
**С виртуальным диском данных не возможно работать двум инстансам одновременно, приходится отмонтировать диск от одного инстанса 
и примонтировать к другому**
