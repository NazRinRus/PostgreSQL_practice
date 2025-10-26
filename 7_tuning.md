## Тюнинг PostgreSQL и тестирование производительности
Для тестирования производительности, подняты две виртуальные машины с идентичными параметрами, но для данных в одном случае примонтирован HDD, а в другом SSD
### Описание стенда:
1. Сервер с настройками по умолчанию: 
- имя хоста - `pg-default-conf`
- ОЗУ 4ГБ
- ЦПУ 2 ядра
- ip-адрес - 192.168.0.190
- СУБД - PostgreSQL 17
- Примонтированный диск HDD `/dev/sdb1  20G  1.3G   18G   7% /data`
- ОС - `Ubuntu 24.02`
2. Сервер с настройками для производительности: 
- имя хоста - `pg-custom-conf`
- ОЗУ 4ГБ
- ЦПУ 2 ядра
- ip-адрес - 192.168.0.191
- СУБД - PostgreSQL 17
- Примонтированный диск SSD `/dev/sdb1  20G  1.3G   18G   7% /data`
- ОС - `Ubuntu 24.02`
### Подготовка сервера

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
sudo apt install postgresql-${PGVERSION?} postgresql-${PGVERSION?}-repack postgresql-${PGVERSION?}-dbgsym postgresql-client-${PGVERSION?} postgresql-contrib unzip -y
```
### Мониторинг
#### Плейбук для автоматической раскатки:
Устанавливает экпортеры, настраивает конфиги и добавляет пользователей
```
ansible-playbook --diff -i '192.168.0.190,'  main.yml --tags=pgscv -u nazrinrus -e '{"included_roles": ["pgscv_exporter", "node_exporter"]}'
ansible-playbook --diff -i '192.168.0.190,'  main.yml --tags=node_exporter -u nazrinrus -e '{"included_roles": ["node_exporter"]}'
```
#### Ручная установка экспортеров
Пользователь БД монитаринга
```
CREATE USER pgscv WITH LOGIN PASSWORD 'pgscv';
GRANT pg_read_server_files, pg_monitor TO pgscv;
GRANT EXECUTE on FUNCTION pg_current_logfile() TO pgscv;
GRANT SELECT ON pg_stat_database TO pgscv;
GRANT SELECT ON pg_stat_bgwriter TO pgscv;
GRANT SELECT ON pg_stat_user_tables TO pgscv;
GRANT SELECT ON pg_stat_user_indexes TO pgscv;
GRANT SELECT ON pg_locks TO pgscv;
GRANT SELECT ON pg_stat_activity TO pgscv;
```
Установка prometheus-агента pgSCV
```
cd /tmp
wget https://github.com/CHERTS/pgscv/releases/download/v0.14.2/pgscv_0.14.2_linux_x86_64.tar.gz
tar -xzf pgscv_0.14.2_linux_x86_64.tar.g
sudo cp pgscv /usr/bin/pgscv
sudo chmod +x /usr/local/bin/pgscv
sudo useradd --system --no-create-home --shell /bin/false -G postgres pgscv
```
Настройка конфигурационного файла `sudo vim /etc/pgscv.yaml`:
```
sudo tee /etc/pgscv.yaml > /dev/null << 'EOF'
# PGSCV configuration
listen_address: 0.0.0.0:9890
no_track_mode: yes
log_level: "info"

defaults: 
    postgres_username: "pgscv"
    postgres_password: "pgscv"

services:
    postgres:
        service_type: "postgres"
        conninfo: "postgres://pgscv:pgscv@127.0.0.1:5432/postgres"
EOF

sudo chown pgscv:postgres /etc/pgscv.yaml
```
Настройка юнита сервиса `sudo vim /etc/systemd/system/pgscv.service`:
```
[Unit]
Description=pgSCV is the Weaponry platform agent for PostgreSQL ecosystem
Requires=network-online.target
After=network-online.target

[Service]
Type=simple
User=postgres
Group=postgres

# Start the agent process
ExecStart=/usr/bin/pgscv --config-file=/etc/pgscv.yaml

# Kill all processes in the cgroup
KillMode=control-group

# Wait reasonable amount of time for agent up/down
TimeoutSec=5

# Restart agent if it crashes
Restart=on-failure
RestartSec=10

# if agent leaks during long period of time, let him to be the first person for eviction
OOMScoreAdjust=1000

[Install]
WantedBy=multi-user.target
```
Перезапуск сервиса:
```
systemctl daemon-reload
systemctl enable pgscv
systemctl start pgscv
```
Проверка:
```
curl -s http://127.0.0.1:9890/metrics | grep -c ^postgres
772
curl -s http://127.0.0.1:9890/metrics | grep -c ^node
391
```
### Тестирование производительности с настройками по умолчанию
Для тестирования производительности применяется расширение `pg_bench` https://postgrespro.ru/docs/postgresql/17/pgbench
```
PGPASSWORD=postgres pgbench -i test_db # инициализация
PGPASSWORD=postgres pgbench -h 192.168.0.190 -U postgres -d test_db -T 300 -c 100 # выполняем стандартную нагрузку в течении 5 минут для 100 клиентов

transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 1
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 68288
number of failed transactions: 0 (0.000%)
latency average = 436.441 ms
initial connection time = 2394.075 ms
tps = 229.125987 (without initial connection time)
```
### Тестирование производительности с выставленными оптимальными настройками
#### Настройка конфигурации PostgreSQL
```
effective_cache_size = 2GB
shared_buffers = 1GB
temp_buffers = 256MB
work_mem = 128MB
maintenance_work_mem = 256MB
```
#### Тестирование с новыми параметрами конфигурации
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 1
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 63051
number of failed transactions: 0 (0.000%)
latency average = 473.346 ms
initial connection time = 2168.020 ms
tps = 211.261902 (without initial connection time)
```
### Тестирование производительности с настройками на оптимальную производительность на хосте 192.168.0.191
Предложенные в конфигураторе https://pgconfigurator.cybertec.at/
```
# Connectivity
max_connections = 540
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '1024 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '3 GB'
effective_io_concurrency = 100 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1.25 # speed of random disk access relative to sequential access (1.0)

# Monitoring
shared_preload_libraries = 'pg_stat_statements'    # per statement resource usage stats
track_io_timing=on        # measure exact block IO times
track_functions=pl        # track execution times of pl-language procedures if any

# Replication
wal_level = replica		# consider using at least 'replica'
max_wal_senders = 0
synchronous_commit = on

# Checkpointing: 
checkpoint_timeout  = '15 min' 
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'


# WAL writing
wal_compression = on
wal_buffers = -1    # auto-tuned by Postgres till maximum of segment size (16MB by default)
wal_writer_delay = 200ms
wal_writer_flush_after = 1MB


# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Parallel queries: 
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_maintenance_workers = 1
max_parallel_workers = 2
parallel_leader_participation = on

# Advanced features 
enable_partitionwise_join = on 
enable_partitionwise_aggregate = on
jit = on
max_slot_wal_keep_size = '1000 MB'
track_wal_io_timing = on
maintenance_io_concurrency = 100
wal_recycle = on
```
#### Тестирование с новыми параметрами конфигурации
```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 1
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 80652
number of failed transactions: 0 (0.000%)
latency average = 369.922 ms
initial connection time = 2093.119 ms
tps = 270.326962 (without initial connection time)
```
### Вывод
На быстрых дисках и найтройках с максимальной производительностью, количество обработанных запросов за 5 минут увеличилось с ~63051 до ~80652
**Для реального тестирования необходимо протестировать на виртуалке с большим количеством ядер и памятью, увеличить количество данных, подготовить запросы с 
большей селективностью. На данном калькуляторе разница не ощущается**

