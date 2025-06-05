## Тюнинг PostgreSQL и тестирование производительности
### Описание стенда:
1. Сервер: 
- ОЗУ 4ГБ
- ЦПУ 2 ядра
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
### Тестирование производительности с настройками по умолчанию
Для тестирования производительности применяется расширение `pg_bench` https://postgrespro.ru/docs/postgresql/17/pgbench
```
pgbench -i test_db # инициализация
pgbench -d test_db -T 300 -c 100 # выполняем стандартную нагрузку в течении 5 минут для 100 клиентов

postgres@otus-practice3:~$ pgbench -d test_db -T 300 -c 100
pgbench (17.5 (Ubuntu 17.5-1.pgdg20.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 1
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 127727
number of failed transactions: 0 (0.000%)
latency average = 234.939 ms
initial connection time = 260.718 ms
tps = 425.642186 (without initial connection time)
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
postgres@otus-practice3:~$ pgbench -d test_db -T 300 -c 100
pgbench (17.5 (Ubuntu 17.5-1.pgdg20.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 1
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 124827
number of failed transactions: 0 (0.000%)
latency average = 240.499 ms
initial connection time = 153.542 ms
tps = 415.801477 (without initial connection time)
```
### Тестирование производительности с настройками на оптимальную производительность не обращая внимания на стабильность БД
Предложенные в конфигураторе https://pgconfigurator.cybertec.at/
```
# Connectivity
max_connections = 100
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '1024 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '3 GB'
effective_io_concurrency = 200 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1.25 # speed of random disk access relative to sequential access (1.0)

# Monitoring
shared_preload_libraries = 'pg_stat_statements'    # per statement resource usage stats
track_io_timing=on        # measure exact block IO times
track_functions=pl        # track execution times of pl-language procedures if any

# Replication
wal_level = replica 		# consider using at least 'replica'
max_wal_senders = 10
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
wal_keep_size = '3650 MB'


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
maintenance_io_concurrency = 200
wal_recycle = on
```
#### Тестирование с новыми параметрами конфигурации
```
postgres@otus-practice3:~$ pgbench -d test_db -T 300 -c 100
pgbench (17.5 (Ubuntu 17.5-1.pgdg20.04+1))
starting vacuum...end.
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 100
number of threads: 1
maximum number of tries: 1
duration: 300 s
number of transactions actually processed: 103630
number of failed transactions: 0 (0.000%)
latency average = 289.714 ms
initial connection time = 164.771 ms
tps = 345.167742 (without initial connection time)
```
### Вывод
**Протестировать на виртуалке с большим количеством ядер и памятью, увеличить количество данных, подготовить запросы с 
большей селективностью. На данном калькуляторе разница не ощущается**
