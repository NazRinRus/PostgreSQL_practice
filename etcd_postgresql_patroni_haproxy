## Развернуть HA кластер
### Структура кластера:
- 3 ВМ для etcd (192.168.0.181, 192.168.0.182, 192.168.0.183);
- 3 ВМ для Patroni, PostgreSQL (192.168.0.191, 192.168.0.192, 192.168.0.193); 
- 1 ВМ для HAproxy (192.168.0.171)
- настраиваем бэкапы через wal-g или pg_probackup
### Предварительная настройка хостов:
В качестве хостов будут выступать виртуальные машины VirtualBox, на которых настроен сетевой мост, объединяющий в подсеть `192.168.0.0/24`

- На хостах должен быть установлен SSH-сервер;
- Передать на хосты публичный SSH-ключ управляющей машины `ssh-copy-id nazrinrus@192.168.0.104`;
- Для перехода по SSH хосты должны иметь статичные ip-адреса (можно и динамичные, но они часто меняются при перезагурке/
удалении/добавлении новых виртуальных машин). Настроить в `netplan`:
1. Настроить на виртуальных машинах тип подключения `Сетевой мост`, тогда они будут в локальной сети роутера;
2. Вывести информацию о сетевых интерфейсах: `ip a`, имя интерфейса подключенного к локальной сети, например, `enp0s3`
3. Создать файл `sudo vim /etc/netplan/manual.yaml`, примерное содержание:
```
network:
  ethernets:
    enp0s3:  # логическое название 
        # отключаем DHCP
      dhcp4: false  
      dhcp6: false
        # задаём IP-адрес и маску
      addresses:
        - 192.168.0.180/24
        # задаём маршрут по умолчанию
      routes:
        - to: default
          via: 192.168.0.1
        # задаём DNS-настройки
      nameservers:
        # домен поиска
        search:
          - ru-central1.internal
          - auto.internal
        # адреса DNS-серверов
        addresses:
          - 192.168.0.1
        # определяем, к какому устройству применить настройки
      match:
        macaddress: 08:00:27:ff:58:15
```
4. Применить `sudo netplan apply`
### Настройка хостов `etcd`
#### Отредактировать файлы `sudo vim /etc/hosts`:
Хост `etcdnode1: 192.168.0.181` 
```
127.0.0.1 localhost etcdnode1
192.168.0.181 etcdnode1
192.168.0.182 etcdnode2
192.168.0.183 etcdnode3
192.168.0.191 postgresql-node1
192.168.0.192 postgresql-node2
192.168.0.193 postgresql-node3
192.168.0.171 haproxy-node
```
Хост `etcdnode2: 192.168.0.182`
```
127.0.0.1 localhost etcdnode2
192.168.0.181 etcdnode1
192.168.0.182 etcdnode2
192.168.0.183 etcdnode3
192.168.0.191 postgresql-node1
192.168.0.192 postgresql-node2
192.168.0.193 postgresql-node3
192.168.0.171 haproxy-node
```
Хост `etcdnode3: 192.168.0.183`
```
127.0.0.1 localhost etcdnode3
192.168.0.181 etcdnode1
192.168.0.182 etcdnode2
192.168.0.183 etcdnode3
192.168.0.191 postgresql-node1
192.168.0.192 postgresql-node2
192.168.0.193 postgresql-node3
192.168.0.171 haproxy-node
```
### Установка etcd на хостах:
#### Скачать дистрибутив etcd
```
cd /tmp
wget https://github.com/etcd-io/etcd/releases/download/v3.5.5/etcd-v3.5.5-linux-amd64.tar.gz
```
Разархивируем:
```
tar xzvf etcd-v3.5.5-linux-amd64.tar.gz
```
Перемещаем исполняемые файлы etcd в `usr/local/bin`:
```
sudo mv /tmp/etcd-v3.5.5-linux-amd64/etcd* /usr/local/bin/
```
#### Настройка пользователей и прав
Создаем пользователя, от которого будет работать служба etcd:
```
sudo groupadd --system etcd
sudo useradd -s /sbin/nologin --system -g etcd etcd
```
Создаем каталоги etcd, меняем владельца и права:
```
sudo su
mkdir /opt/etcd
mkdir /etc/etcd
mkdir /var/lib/etcd
chown -R etcd:etcd /opt/etcd /var/lib/etcd /etc/etcd
chmod -R 700 /opt/etcd/ /var/lib/etcd /etc/etcd
exit
```
#### Настройка конфигурационных файлов
Далее создаем конфиги etcd. Для каждой ноды — свой отдельный.
```
sudo vim /etc/etcd/etcd.conf
```
Текст конфига для `etcdnode1`:
```
ETCD_NAME="etcdnode1"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.181:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.181:2379"
ETCD_LISTEN_PEER_URLS="http://192.168.0.181:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.181:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-postgres-cluster"
ETCD_INITIAL_CLUSTER="etcdnode1=http://192.168.0.181:2380,etcdnode2=http://192.168.0.182:2380,etcdnode3=http://192.168.0.183:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```
Текст конфига для `etcdnode2`:
```
ETCD_NAME="etcdnode2"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.182:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.182:2379"
ETCD_LISTEN_PEER_URLS="http://192.168.0.182:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.182:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-postgres-cluster"
ETCD_INITIAL_CLUSTER="etcdnode1=http://192.168.0.181:2380,etcdnode2=http://192.168.0.182:2380,etcdnode3=http://192.168.0.183:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```
Текст конфига для `etcdnode3`:
```
ETCD_NAME="etcdnode3"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.183:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.183:2379"
ETCD_LISTEN_PEER_URLS="http://192.168.0.183:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.183:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-postgres-cluster"
ETCD_INITIAL_CLUSTER="etcdnode1=http://192.168.0.181:2380,etcdnode2=http://192.168.0.182:2380,etcdnode3=http://192.168.0.183:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```
На всякий случай повторить назначение прав:
```
sudo chown -R etcd:etcd /opt/etcd /var/lib/etcd /etc/etcd
sudo chmod -R 700 /opt/etcd/ /var/lib/etcd /etc/etcd
```
#### Настройка службы etcd
Далее на каждой ноде делаем etcd службой (конфиг одинаковый):
```
sudo vim /etc/systemd/system/etcd.service
```
Текст конфига:
```
[Unit]
Description=Etcd Server
Documentation=https://github.com/etcd-io/etcd
After=network.target
After=network-online.target
Wants=network-online.target
  
[Service]
User=etcd
Type=notify
#WorkingDirectory=/var/lib/etcd/
WorkingDirectory=/opt/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/local/bin/etcd"
Restart=on-failure
LimitNOFILE=65536
IOSchedulingClass=realtime
IOSchedulingPriority=0
Nice=-20
 
[Install]
WantedBy=multi-user.target
```
Далее настраиваем автозапуск службы etcd и ее запускаем:
```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```
И проверяем, что она запустилась:
```
sudo systemctl status etcd
```
#### Добавить путь `/usr/local/bin/` для `etcdctl` в `PATH`
```
export PATH="/usr/local/bin:$PATH"
```
#### Проверка нод etcd
Для проверки можно посмотреть информацию о нодах кластера (и узнать, кто лидер):
```
ETCDCTL_API=2 etcdctl member list
```
и проверить здоровье нод:
```
etcdctl endpoint health --cluster -w table
```
### Установка PostgreSQL
Производится на `postgresql-node1`, `postgresql-node2`, `postgresql-node3`. Выполняется из под пользователя `root`.
#### Отредактировать файлы `sudo vim /etc/hosts`:
Хост `postgresql-node1: 192.168.0.191` 
```
127.0.0.1 localhost postgresql-node1
192.168.0.181 etcdnode1
192.168.0.182 etcdnode2
192.168.0.183 etcdnode3
192.168.0.191 postgresql-node1
192.168.0.192 postgresql-node2
192.168.0.193 postgresql-node3
192.168.0.171 haproxy-node
```
Хост `postgresql-node2: 192.168.0.192`
```
127.0.0.1 localhost postgresql-node2
192.168.0.181 etcdnode1
192.168.0.182 etcdnode2
192.168.0.183 etcdnode3
192.168.0.191 postgresql-node1
192.168.0.192 postgresql-node2
192.168.0.193 postgresql-node3
192.168.0.171 haproxy-node
```
Хост `postgresql-node3: 192.168.0.193`
```
127.0.0.1 localhost postgresql-node3
192.168.0.181 etcdnode1
192.168.0.182 etcdnode2
192.168.0.183 etcdnode3
192.168.0.191 postgresql-node1
192.168.0.192 postgresql-node2
192.168.0.193 postgresql-node3
192.168.0.171 haproxy-node
```
#### Региструем репозитарий `postgresql.org`:
```
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
```
#### Устанавливаем Postgres 17:
```
apt install postgresql-17
```
#### Задаем пароль для роли `postgres`, создаем роль с паролем `replica` `replica` :
```
sudo su postgres
psql
\password
```
Далее два раза вводим пароль (в нашем случае — `postgres`).
```
create user replica replication login encrypted password 'replica';
```
Создаем пользователя `pgbouncer` (если он будет использоваться):
```
create user pgbouncer login encrypted password 'pgbouncer';
```
и выходим из psql, а затем возвращаемся на `root`: `Ctrl+D`
#### Редактируем файл `pg_hba.conf`, чтобы разрешить удаленные подключения (обычные и для репликации):
```
vim /etc/postgresql/17/main/pg_hba.conf
```
и меняем строки 
```
host all all 127.0.0.1/32 scram-sha-256
host replication all 127.0.0.1/32 scram-sha-256
```
на 
```
host all all 0.0.0.0/0 scram-sha-256
host replication all 0.0.0.0/0 scram-sha-256
```
Редактируем файл `postgresql.conf`, чтобы разрешить прослушивание для всех IP-адресов:
```
vim /etc/postgresql/17/main/postgresql.conf
```
В строке 
```
#listen_address = 'localhost'
```
снимаем коментарий и ставим звездочку:
```
listen_address = '*'
```
После этого первую ноду (`postgresql-node1`) нужно перезапустить:
```
systemctl restart postgresql
```
(остальные необязательно).
#### Дальнейшее производится только на `postgresql-node2` и `postgresql-node3` (второй и третьей нодах):
На второй и третьей ноде нужно удалить содержимое каталога `pgdata`, поскольку оно будет отреплицировано с первой ноды при развертывании кластера:
```
systemctl stop postgresql
rm -rf /var/lib/postgresql/17/main/*
```
### Установка Patroni
#### Устанавливаем пакеты для работы с Python:
```
sudo apt -y install python3 python3-pip python3-dev python3-psycopg2 libpq-dev
```
Через PIP ставим пакеты Python (обязательно от `root`):
```
sudo su
pip3 install psycopg2 --break-system-packages
pip3 install psycopg2-binary --break-system-packages
pip3 install patroni --break-system-packages
pip3 install python-etcd --break-system-packages
```
#### Конфигурационный файл Patroni
Создаем каталог и файл конфигов Patroni:
```
sudo mkdir /etc/patroni/
sudo vim /etc/patroni/patroni.yml
```
текст конфигурации `/etc/patroni/patroni.yml` на хосте `postgresql-node1`:
```
scope: postgres-cluster # одинаковое значение на всех узлах
name: postgresql-node1 # разное значение на всех узлах
namespace: /service/ # одинаковое значение на всех узлах

restapi:
  listen: 192.168.0.191:8008 # разное значение на всех узлах
  connect_address: 192.168.0.191:8008 # разное значение на всех узлах
  authentication:
    username: patroni
    password: 'patroni'

etcd:
  hosts: 192.168.0.181:2379, 192.168.0.182:2379, 192.168.0.183:2379 # список всех узлов, на которых установлен etcd

bootstrap:
  method: initdb
  dcs:
    ttl: 60
    loop_wait: 10
    retry_timeout: 27
    maximum_lag_on_failover: 2048576
    master_start_timeout: 300
    synchronous_mode: true
    synchronous_mode_strict: false
    synchronous_node_count: 1
    # standby_cluster:
      # host: 127.0.0.1
      # port: 1111
      # primary_slot_name: patroni
    postgresql:
      use_pg_rewind: false
      use_slots: true
      parameters:
        max_connections: 800
        superuser_reserved_connections: 5
        max_locks_per_transaction: 64
        max_prepared_transactions: 0
        huge_pages: try
        shared_buffers: 512MB
        work_mem: 128MB
        maintenance_work_mem: 256MB
        effective_cache_size: 4GB
        checkpoint_timeout: 15min
        checkpoint_completion_target: 0.9
        min_wal_size: 2GB
        max_wal_size: 4GB
        wal_buffers: 32MB
        default_statistics_target: 1000
        seq_page_cost: 1
        random_page_cost: 4
        effective_io_concurrency: 2
        synchronous_commit: on
        autovacuum: on
        autovacuum_max_workers: 5
        autovacuum_vacuum_scale_factor: 0.01
        autovacuum_analyze_scale_factor: 0.02
        autovacuum_vacuum_cost_limit: 200
        autovacuum_vacuum_cost_delay: 20
        autovacuum_naptime: 1s
        max_files_per_process: 4096
        archive_mode: on
        archive_timeout: 1800s
        archive_command: cd .
        wal_level: replica
        wal_keep_segments: 130
        max_wal_senders: 10
        max_replication_slots: 10
        hot_standby: on
        hot_standby_feedback: True
        wal_log_hints: on
        shared_preload_libraries: pg_stat_statements,auto_explain
        pg_stat_statements.max: 10000
        pg_stat_statements.track: all
        pg_stat_statements.save: off
        auto_explain.log_min_duration: 10s
        auto_explain.log_analyze: true
        auto_explain.log_buffers: true
        auto_explain.log_timing: false
        auto_explain.log_triggers: true
        auto_explain.log_verbose: true
        auto_explain.log_nested_statements: true
        track_io_timing: on
        log_lock_waits: on
        log_temp_files: 0
        track_activities: on
        track_counts: on
        track_functions: all
        log_checkpoints: on
        logging_collector: on
        log_statement: mod
        log_truncate_on_rotation: on
        log_rotation_age: 1d
        log_rotation_size: 0
        log_line_prefix: '%m [%p] %q%u@%d '
        log_filename: 'postgresql-%a.log'
        log_directory: /var/log/postgresql

  initdb:  # List options to be passed on to initdb
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

  pg_hba:  # должен содержать адреса ВСЕХ машин, используемых в кластере
    - host all all 0.0.0.0/0 scram-sha-256
    - host replication replica scram-sha-256

postgresql:
  listen: 192.168.0.191,127.0.0.1:5432 # разное значение на всех узлах
  connect_address: 192.168.0.191:5432 # разное значение на всех узлах
  use_unix_socket: true
  data_dir: /var/lib/postgresql/17/main
  bin_dir: /usr/lib/postgresql/17/bin
  config_dir: /etc/postgresql/17/main
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: replica
      password: replica
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: /var/run/postgresql
    stats_temp_directory: /var/lib/pgsql_stats_tmp

  remove_data_directory_on_rewind_failure: false
  remove_data_directory_on_diverged_timelines: false

#  callbacks:
#    on_start:
#    on_stop:
#    on_restart:
#    on_reload:
#    on_role_change:

  create_replica_methods:
    - basebackup
  basebackup:
    max-rate: '100M'
    checkpoint: 'fast'

watchdog:
  mode: off  # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false

  # specify a node to replicate from (cascading replication)
#  replicatefrom: (node name)
```
текст конфигурации `/etc/patroni/patroni.yml` на хосте `postgresql-node2`:
```
scope: postgres-cluster # одинаковое значение на всех узлах
name: postgresql-node2 # разное значение на всех узлах
namespace: /service/ # одинаковое значение на всех узлах

restapi:
  listen: 192.168.0.192:8008 # разное значение на всех узлах
  connect_address: 192.168.0.192:8008 # разное значение на всех узлах
  authentication:
    username: patroni
    password: 'patroni'

etcd:
  hosts: 192.168.0.181:2379, 192.168.0.182:2379, 192.168.0.183:2379 # список всех узлов, на которых установлен etcd

bootstrap:
  method: initdb
  dcs:
    ttl: 60
    loop_wait: 10
    retry_timeout: 27
    maximum_lag_on_failover: 2048576
    master_start_timeout: 300
    synchronous_mode: true
    synchronous_mode_strict: false
    synchronous_node_count: 1
    # standby_cluster:
      # host: 127.0.0.1
      # port: 1111
      # primary_slot_name: patroni
    postgresql:
      use_pg_rewind: false
      use_slots: true
      parameters:
        max_connections: 800
        superuser_reserved_connections: 5
        max_locks_per_transaction: 64
        max_prepared_transactions: 0
        huge_pages: try
        shared_buffers: 512MB
        work_mem: 128MB
        maintenance_work_mem: 256MB
        effective_cache_size: 4GB
        checkpoint_timeout: 15min
        checkpoint_completion_target: 0.9
        min_wal_size: 2GB
        max_wal_size: 4GB
        wal_buffers: 32MB
        default_statistics_target: 1000
        seq_page_cost: 1
        random_page_cost: 4
        effective_io_concurrency: 2
        synchronous_commit: on
        autovacuum: on
        autovacuum_max_workers: 5
        autovacuum_vacuum_scale_factor: 0.01
        autovacuum_analyze_scale_factor: 0.02
        autovacuum_vacuum_cost_limit: 200
        autovacuum_vacuum_cost_delay: 20
        autovacuum_naptime: 1s
        max_files_per_process: 4096
        archive_mode: on
        archive_timeout: 1800s
        archive_command: cd .
        wal_level: replica
        wal_keep_segments: 130
        max_wal_senders: 10
        max_replication_slots: 10
        hot_standby: on
        hot_standby_feedback: True
        wal_log_hints: on
        shared_preload_libraries: pg_stat_statements,auto_explain
        pg_stat_statements.max: 10000
        pg_stat_statements.track: all
        pg_stat_statements.save: off
        auto_explain.log_min_duration: 10s
        auto_explain.log_analyze: true
        auto_explain.log_buffers: true
        auto_explain.log_timing: false
        auto_explain.log_triggers: true
        auto_explain.log_verbose: true
        auto_explain.log_nested_statements: true
        track_io_timing: on
        log_lock_waits: on
        log_temp_files: 0
        track_activities: on
        track_counts: on
        track_functions: all
        log_checkpoints: on
        logging_collector: on
        log_statement: mod
        log_truncate_on_rotation: on
        log_rotation_age: 1d
        log_rotation_size: 0
        log_line_prefix: '%m [%p] %q%u@%d '
        log_filename: 'postgresql-%a.log'
        log_directory: /var/log/postgresql

  initdb:  # List options to be passed on to initdb
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

  pg_hba:  # должен содержать адреса ВСЕХ машин, используемых в кластере
    - host all all 0.0.0.0/0 scram-sha-256
    - host replication replica scram-sha-256

postgresql:
  listen: 192.168.0.192,127.0.0.1:5432 # разное значение на всех узлах
  connect_address: 192.168.0.192:5432 # разное значение на всех узлах
  use_unix_socket: true
  data_dir: /var/lib/postgresql/17/main
  bin_dir: /usr/lib/postgresql/17/bin
  config_dir: /etc/postgresql/17/main
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: replica
      password: replica
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: /var/run/postgresql
    stats_temp_directory: /var/lib/pgsql_stats_tmp

  remove_data_directory_on_rewind_failure: false
  remove_data_directory_on_diverged_timelines: false

#  callbacks:
#    on_start:
#    on_stop:
#    on_restart:
#    on_reload:
#    on_role_change:

  create_replica_methods:
    - basebackup
  basebackup:
    max-rate: '100M'
    checkpoint: 'fast'

watchdog:
  mode: off  # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false

  # specify a node to replicate from (cascading replication)
#  replicatefrom: (node name)
```
текст конфигурации `/etc/patroni/patroni.yml` на хосте `postgresql-node3`:
```
scope: postgres-cluster # одинаковое значение на всех узлах
name: postgresql-node3 # разное значение на всех узлах
namespace: /service/ # одинаковое значение на всех узлах

restapi:
  listen: 192.168.0.193:8008 # разное значение на всех узлах
  connect_address: 192.168.0.193:8008 # разное значение на всех узлах
  authentication:
    username: patroni
    password: 'patroni'

etcd:
  hosts: 192.168.0.181:2379, 192.168.0.181:2379, 192.168.0.181:2379 # список всех узлов, на которых установлен etcd

bootstrap:
  method: initdb
  dcs:
    ttl: 60
    loop_wait: 10
    retry_timeout: 27
    maximum_lag_on_failover: 2048576
    master_start_timeout: 300
    synchronous_mode: true
    synchronous_mode_strict: false
    synchronous_node_count: 1
    # standby_cluster:
      # host: 127.0.0.1
      # port: 1111
      # primary_slot_name: patroni
    postgresql:
      use_pg_rewind: false
      use_slots: true
      parameters:
        max_connections: 800
        superuser_reserved_connections: 5
        max_locks_per_transaction: 64
        max_prepared_transactions: 0
        huge_pages: try
        shared_buffers: 512MB
        work_mem: 128MB
        maintenance_work_mem: 256MB
        effective_cache_size: 4GB
        checkpoint_timeout: 15min
        checkpoint_completion_target: 0.9
        min_wal_size: 2GB
        max_wal_size: 4GB
        wal_buffers: 32MB
        default_statistics_target: 1000
        seq_page_cost: 1
        random_page_cost: 4
        effective_io_concurrency: 2
        synchronous_commit: on
        autovacuum: on
        autovacuum_max_workers: 5
        autovacuum_vacuum_scale_factor: 0.01
        autovacuum_analyze_scale_factor: 0.02
        autovacuum_vacuum_cost_limit: 200
        autovacuum_vacuum_cost_delay: 20
        autovacuum_naptime: 1s
        max_files_per_process: 4096
        archive_mode: on
        archive_timeout: 1800s
        archive_command: cd .
        wal_level: replica
        wal_keep_segments: 130
        max_wal_senders: 10
        max_replication_slots: 10
        hot_standby: on
        hot_standby_feedback: True
        wal_log_hints: on
        shared_preload_libraries: pg_stat_statements,auto_explain
        pg_stat_statements.max: 10000
        pg_stat_statements.track: all
        pg_stat_statements.save: off
        auto_explain.log_min_duration: 10s
        auto_explain.log_analyze: true
        auto_explain.log_buffers: true
        auto_explain.log_timing: false
        auto_explain.log_triggers: true
        auto_explain.log_verbose: true
        auto_explain.log_nested_statements: true
        track_io_timing: on
        log_lock_waits: on
        log_temp_files: 0
        track_activities: on
        track_counts: on
        track_functions: all
        log_checkpoints: on
        logging_collector: on
        log_statement: mod
        log_truncate_on_rotation: on
        log_rotation_age: 1d
        log_rotation_size: 0
        log_line_prefix: '%m [%p] %q%u@%d '
        log_filename: 'postgresql-%a.log'
        log_directory: /var/log/postgresql

  initdb:  # List options to be passed on to initdb
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

  pg_hba:  # должен содержать адреса ВСЕХ машин, используемых в кластере
    - host all all 0.0.0.0/0 scram-sha-256
    - host replication replica scram-sha-256

postgresql:
  listen: 192.168.0.193,127.0.0.1:5432 # разное значение на всех узлах
  connect_address: 192.168.0.193:5432 # разное значение на всех узлах
  use_unix_socket: true
  data_dir: /var/lib/postgresql/17/main
  bin_dir: /usr/lib/postgresql/17/bin
  config_dir: /etc/postgresql/17/main
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: replica
      password: replica
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: /var/run/postgresql
    stats_temp_directory: /var/lib/pgsql_stats_tmp

  remove_data_directory_on_rewind_failure: false
  remove_data_directory_on_diverged_timelines: false

#  callbacks:
#    on_start:
#    on_stop:
#    on_restart:
#    on_reload:
#    on_role_change:

  create_replica_methods:
    - basebackup
  basebackup:
    max-rate: '100M'
    checkpoint: 'fast'

watchdog:
  mode: off  # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false

  # specify a node to replicate from (cascading replication)
#  replicatefrom: (node name)
```
#### Назначаем права на каждой ноде:
```
sudo chown postgres:postgres -R /etc/patroni
sudo chmod 700 /etc/patroni
sudo mkdir /var/lib/pgsql_stats_tmp
sudo chown postgres:postgres /var/lib/pgsql_stats_tmp
```
#### Проверка Patroni
Чтобы проверить, все ли сделано правильно, на всех нодах выполняем команду:
```
sudo -u postgres /usr/local/bin/patroni /etc/patroni/patroni.yml
# или
sudo -u postgres /usr/local/bin/patronictl version
```
Каждый из серверов должен запуститься и написать, он в роли лидера или в роли secondary.
#### Определяем Patroni как службу (на всех трех нодах одинаково):
Редактируем файл `sudo vim /etc/systemd/system/patroni.service`:
```
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres

# Read in configuration file if it exists, otherwise proceed
EnvironmentFile=-/etc/patroni_env.conf

# Start the patroni process
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml

# Send HUP to reload from patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID

# only kill the patroni process, not it's children, so it will gracefully stop postgres
KillMode=process

# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=60

# Do not restart the service if it crashes, we want to manually inspect database on failure
Restart=no

[Install]
WantedBy=multi-user.target
```
Переводим Patroni в автозапуск, стартуем и проверяем:
```
sudo systemctl daemon-reload
sudo systemctl enable patroni
sudo systemctl start patroni
sudo systemctl status patroni
```
### Проверить состояние кластера
Просмотреть состояние кластера можно командой
```
patronictl -c /etc/patroni/patroni.yml list
```
Для failover:
```
patronictl -c /etc/patroni/patroni.yml failover
```
### Установка PGBouncer
Производится на всех нодах (`postgresql-node1`, `postgresql-node2`, `postgresql-node3`). Выполняется от пользователя `root`
#### Устанавливаем PGBouncer:
```
sudo apt -y install pgbouncer
```
Сохраняем исходный вариант файла конфигурации:
```
sudo mv /etc/pgbouncer/pgbouncer.ini /etc/pgbouncer/pgbouncer.ini.origin
```
Открываем его на редактирование:
```
sudo vim /etc/pgbouncer/pgbouncer.ini
```
Вставляем текст (на каждой ноде одинаковый):
```
[databases]
postgres = host=127.0.0.1 port=5432 dbname=postgres
* = host=127.0.0.1 port=5432

[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = *
listen_port = 6432
unix_socket_dir = /var/run/postgresql
auth_type = md5
#auth_type = trust
auth_file = /etc/pgbouncer/userlist.txt
auth_user = postgres
auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1
#admin_users = pgbouncer, postgres
admin_users = postgres
ignore_startup_parameters = extra_float_digits,geqo,search_path

pool_mode = session
#pool_mode = transaction
server_reset_query = DISCARD ALL
max_client_conn = 10000
#default_pool_size = 20
reserve_pool_size = 1
reserve_pool_timeout = 1
max_db_connections = 1000
#max_client_conn = 900
default_pool_size = 500
pkt_buf = 8192
listen_backlog = 4096
log_connections = 1
log_disconnections = 1
```
#### Создаем файл `userlist.txt` со списком пользователей для работы через PGBouncer:
```
sudo vim /etc/pgbouncer/userlist.txt
```
Добавляем в него пользователей:
```
"postgres" "postgres"
"pgbouncer" "pgbouncer"
```
#### Перезапускаем и проверяем PGBouncer:
```
sudo systemctl restart pgbouncer

sudo -u postgres psql -p 6432 -h 127.0.0.1 -U postgres postgres
```
### Установка HAProxy
Производится на компьютере `haproxy-node`. Выполняется от пользователя `root`
#### Отредактировать файлы `sudo vim /etc/hosts`:
Хост `haproxy-node: 192.168.0.171` 
```
127.0.0.1 localhost haproxy-node
192.168.0.181 etcdnode1
192.168.0.182 etcdnode2
192.168.0.183 etcdnode3
192.168.0.191 postgresql-node1
192.168.0.192 postgresql-node2
192.168.0.193 postgresql-node3
192.168.0.171 haproxy-node
```
#### Устанавливаем HAProxy:
```
sudo apt -y install haproxy
```
#### Настройка конфигурации HAproxy
Сохраняем исходный файл конфигурации:
```
sudo mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.origin
```
Создаем новый файл конфигурации `sudo vim /etc/haproxy/haproxy.cfg`:
```
global

        maxconn 10000
        log     127.0.0.1 local2

defaults
        log global
        mode tcp
        retries 2
        timeout client 30m
        timeout connect 4s
        timeout server 30m
        timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres
    bind *:7432
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server postgresql-node1 192.168.0.191:6432 maxconn 10000 check port 8008
    server postgresql-node2 192.168.0.192:6432 maxconn 100 check port 8008
    server postgresql-node3 192.168.0.193:6432 maxconn 100 check port 8008
```
Далее перезагружаем HAProxy:
```
sudo systemctl restart haproxy
```
и проверяем работоспособность:
```
sudo systemctl status haproxy
```
Для проверки можно подключиться через psql на IP-адрес HAProxy к порту 7432.
```
psql -h 192.168.0.171 -p 7432 -U postgres -d postgres
```
