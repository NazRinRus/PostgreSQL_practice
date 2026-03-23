## Развернуть HA кластер: PostgreSQL, Patroni, etcd, HAproxy, Keepalived
### Структура кластера:
- 3 ВМ для etcd (`192.168.0.171`, `192.168.0.172`, `192.168.0.173`);
- 2 ВМ для Patroni, PostgreSQL (`192.168.0.174`, `192.168.0.175`); 
- 2 ВМ для HAproxy (`192.168.0.176`, `192.168.0.177`, VIP `192.168.0.178`)
- 1 ВМ для сервера бэкапов pgbackrest (`192.168.0.170`)
- 1 ВМ мониторинг Prometheus, Grafana (`192.168.0.161`)
### Предварительная настройка хостов:
В качестве хостов будут выступать виртуальные машины VirtualBox, на которых настроен сетевой мост, объединяющий в подсеть `192.168.0.0/24`
```
192.168.0.171 etcd-node1
192.168.0.172 etcd-node2
192.168.0.173 etcd-node3

192.168.0.174 pg-node1
192.168.0.175 pg-node2

192.168.0.176 haproxy-node1
192.168.0.177 haproxy-node2
192.168.0.178 VIP (keepalived)

192.168.0.170 pgbackrest-node

192.168.0.161 prometheus-server
```
### Генерация и распределение сертификатов
Сгенерирую файлы на первой ноде `etcd-node1`, затем перенесу на остальные
1. Создать рабочую директорию:
```
mkdir -p ~/etcd-ssl/{ca,server,client,peer}
cd ~/etcd-ssl
```
2. Генерация корневого ключа CA:
```
cd ca
openssl genrsa -out ca-key.pem 2048
chmod 400 ca-key.pem
```
3. Создание корневого сертификата:
```
# файл с параметрами для корневого сертификата
cat > ca.conf << EOF
[ req ]
distinguished_name = req_distinguished_name
prompt = no

[ req_distinguished_name ]
C = RU
ST = Moscow
L = Moscow
O = zt.ru
OU = etcd-cluster
CN = etcd-root-ca
EOF

# Генерация корневого сертификата
openssl req -new -x509 -days 3650 -key ca-key.pem -out ca.pem -config ca.conf
```
4. Генерация сертификата для узла кластера `etcd-node1`:
```
cat > peer-node1-full.conf << 'EOF'
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = v3_req

[ dn ]
C = RU
ST = Moscow
L = Moscow
O = zt.ru
OU = etcd-cluster
CN = etcd-node1

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = etcd-node1
DNS.2 = etcd-node2
DNS.3 = etcd-node3
DNS.4 = localhost
IP.1 = 127.0.0.1
IP.2 = 192.168.0.171
IP.3 = 192.168.0.172
IP.4 = 192.168.0.173
EOF

openssl genrsa -out peer-node1-key.pem 2048
chmod 400 peer-node1-key.pem
openssl req -new -key peer-node1-key.pem -out peer-node1.csr -config peer-node1-full.conf

openssl x509 -req -days 3650 \
  -in peer-node1.csr \
  -CA ../ca/ca.pem \
  -CAkey ../ca/ca-key.pem \
  -CAcreateserial \
  -out peer-node1.pem \
  -copy_extensions copy
```
5. Генерация сертификата для узла кластера `etcd-node2`:
```
sed 's/etcd-node1/etcd-node2/g' peer-node1-full.conf > peer-node2-full.conf
openssl genrsa -out peer-node2-key.pem 2048
chmod 400 peer-node2-key.pem
openssl req -new -key peer-node2-key.pem -out peer-node2.csr -config peer-node2-full.conf
openssl x509 -req -days 3650 -in peer-node2.csr -CA ../ca/ca.pem -CAkey ../ca/ca-key.pem -CAcreateserial -out peer-node2.pem -copy_extensions copy
```
6. Генерация сертификата для узла кластера `etcd-node3`:
```
sed 's/etcd-node1/etcd-node3/g' peer-node1-full.conf > peer-node3-full.conf
openssl genrsa -out peer-node3-key.pem 2048
chmod 400 peer-node3-key.pem
openssl req -new -key peer-node3-key.pem -out peer-node3.csr -config peer-node3-full.conf
openssl x509 -req -days 3650 -in peer-node3.csr -CA ../ca/ca.pem -CAkey ../ca/ca-key.pem -CAcreateserial -out peer-node3.pem -copy_extensions copy
```
7. Верификация:
```
nazrinrus@etcd-node1:~/etcd-ssl/peer$ openssl verify -CAfile ../ca/ca.pem peer-node1.pem
peer-node1.pem: OK
nazrinrus@etcd-node1:~/etcd-ssl/peer$ openssl verify -CAfile ../ca/ca.pem peer-node2.pem
peer-node2.pem: OK
nazrinrus@etcd-node1:~/etcd-ssl/peer$ openssl verify -CAfile ../ca/ca.pem peer-node3.pem
peer-node3.pem: OK
```
8. Генерация клиентского сертификата. Будет использоваться для подключения к кластеру через `etcdctl` и другими клиентами:
```
cd ~/etcd-ssl/client

# Конфигурационный файл для клиента
cat > client.conf << EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn

[ dn ]
C = RU
ST = Moscow
L = Moscow
O = MyCompany
OU = etcd-clients
CN = client
EOF

# Генерация ключа
openssl genrsa -out client-key.pem 2048
chmod 400 client-key.pem

# Создание CSR
openssl req -new -key client-key.pem -out client.csr -config client.conf

# Подписываю клиентский сертификат
openssl x509 -req -days 3650 -in client.csr -CA ../ca/ca.pem -CAkey ../ca/ca-key.pem -CAcreateserial -out client.pem -extensions req_ext -extfile <(echo "[req_ext]"; echo "extendedKeyUsage = clientAuth")

# Проверка
openssl verify -CAfile ../ca/ca.pem client.pem
```
9. Копирование сертификатов для серверных узлов кластера:
- `etcd-node1`: `ca.pem` - корневой сертификат CA, `peer-node1.pem` - сертификат узла, `peer-node1-key.pem` - приватный ключ узла;
```
sudo mkdir -p /etc/etcd/ssl
sudo cp ~/etcd-ssl/ca/ca.pem /etc/etcd/ssl/
sudo cp ~/etcd-ssl/peer/{peer-node1.pem,peer-node1-key.pem} /etc/etcd/ssl/
sudo chown -R etcd:etcd /etc/etcd/ssl 2>/dev/null || true  # Пользователь etcd будет создан позже
sudo chmod 600 /etc/etcd/ssl/peer-node1-key.pem
sudo chmod 644 /etc/etcd/ssl/{ca.pem,peer-node1.pem}
```
- `etcd-node2`: `ca.pem` - корневой сертификат CA, `peer-node2.pem` - сертификат узла, `peer-node2-key.pem` - приватный ключ узла;
```
scp ~/etcd-ssl/ca/ca.pem nazrinrus@192.168.0.172:/tmp/
scp ~/etcd-ssl/peer/peer-node2.pem nazrinrus@192.168.0.172:/tmp/
scp ~/etcd-ssl/peer/peer-node2-key.pem nazrinrus@192.168.0.172:/tmp/

# на хосте 192.168.0.172
sudo mkdir -p /etc/etcd/ssl
sudo mv /tmp/{ca.pem,peer-node2.pem,peer-node2-key.pem} /etc/etcd/ssl/
sudo chown -R etcd:etcd /etc/etcd/ssl 2>/dev/null || true
sudo chmod 600 /etc/etcd/ssl/peer-node2-key.pem
sudo chmod 644 /etc/etcd/ssl/{ca.pem,peer-node2.pem}
```
- `etcd-node3`: `ca.pem` - корневой сертификат CA, `peer-node3.pem` - сертификат узла, `peer-node3-key.pem` - приватный ключ узла;
```
scp ~/etcd-ssl/ca/ca.pem nazrinrus@192.168.0.173:/tmp/
scp ~/etcd-ssl/peer/peer-node3.pem nazrinrus@192.168.0.173:/tmp/
scp ~/etcd-ssl/peer/peer-node3-key.pem nazrinrus@192.168.0.173:/tmp/

# на хосте 192.168.0.173
sudo mkdir -p /etc/etcd/ssl
sudo mv /tmp/{ca.pem,peer-node3.pem,peer-node3-key.pem} /etc/etcd/ssl/
sudo chown -R etcd:etcd /etc/etcd/ssl 2>/dev/null || true
sudo chmod 600 /etc/etcd/ssl/peer-node3-key.pem
sudo chmod 644 /etc/etcd/ssl/{ca.pem,peer-node3.pem}
```
10. Сертификаты для patroni:
```
cd ~/etcd-ssl/client

# сертификат для Patroni на pg-node1
cat > patroni1.conf << EOF
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = v3_req

[ dn ]
C = RU
ST = Moscow
L = Moscow
O = zt.ru
OU = patroni-cluster
CN = patroni1

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
EOF

openssl genrsa -out patroni1-key.pem 2048
chmod 400 patroni1-key.pem
openssl req -new -key patroni1-key.pem -out patroni1.csr -config patroni1.conf
openssl x509 -req -days 3650 -in patroni1.csr -CA ../ca/ca.pem -CAkey ../ca/ca-key.pem -CAcreateserial -out patroni1.pem -extensions v3_req -extfile patroni1.conf

# для pg-node2
sed 's/patroni1/patroni2/g' patroni1.conf > patroni2.conf
openssl genrsa -out patroni2-key.pem 2048
chmod 400 patroni2-key.pem
openssl req -new -key patroni2-key.pem -out patroni2.csr -config patroni2.conf
openssl x509 -req -days 3650 -in patroni2.csr -CA ../ca/ca.pem -CAkey ../ca/ca-key.pem -CAcreateserial -out patroni2.pem -extensions v3_req -extfile patroni2.conf

# дополнительный клиентский сертификат для patronictl (управление)
cat > patronictl.conf << 'EOF'
[ req ]
default_bits = 2048
prompt = no
default_md = sha256
distinguished_name = dn
req_extensions = v3_req

[ dn ]
C = RU
ST = Moscow
L = Moscow
O = zt.ru
OU = etcd-clients
CN = patronictl

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth
EOF
openssl genrsa -out patronictl-key.pem 2048
chmod 400 patronictl-key.pem
openssl req -new -key patronictl-key.pem -out patronictl.csr -config patronictl.conf
openssl x509 -req -days 3650 -in patronictl.csr -CA ../ca/ca.pem -CAkey ../ca/ca-key.pem -CAcreateserial -out patronictl.pem -extensions v3_req -extfile patronictl.conf
```
Копирование сертификатов на PostgreSQL узлы
```
# На управляющем узле (где создавались сертификаты - etcd-node1)
scp ../ca/ca.pem patroni1.pem patroni1-key.pem nazrinrus@192.168.0.174:/tmp/
# перейти на хост pg-node1
sudo mkdir -p /etc/patroni/ssl
sudo mv /tmp/{ca.pem,patroni1.pem,patroni1-key.pem} /etc/patroni/ssl/
sudo chown -R postgres:postgres /etc/patroni/ssl 2>/dev/null || true
sudo chmod 600 /etc/patroni/ssl/patroni1-key.pem
sudo chmod 644 /etc/patroni/ssl/{ca.pem,patroni1.pem}

# На pg-node2
scp ../ca/ca.pem patroni2.pem patroni2-key.pem nazrinrus@192.168.0.175:/tmp/
# перейти на хост pg-node2
sudo mkdir -p /etc/patroni/ssl
sudo mv /tmp/{ca.pem,patroni2.pem,patroni2-key.pem} /etc/patroni/ssl/
sudo chown -R postgres:postgres /etc/patroni/ssl 2>/dev/null || true
sudo chmod 600 /etc/patroni/ssl/patroni2-key.pem
sudo chmod 644 /etc/patroni/ssl/{ca.pem,patroni2.pem}

# управляющий сертификат на оба узла (для patronictl)
scp ../ca/ca.pem patronictl.pem patronictl-key.pem nazrinrus@192.168.0.174:/tmp/
scp ../ca/ca.pem patronictl.pem patronictl-key.pem nazrinrus@192.168.0.175:/tmp/

# На узлах pg-node1 и pg-node2
sudo mkdir -p /etc/patroni/ctl
sudo mv /tmp/{ca.pem,patronictl.pem,patronictl-key.pem} /etc/patroni/ctl/
sudo chown -R postgres:postgres /etc/patroni/ctl 2>/dev/null || true
sudo chmod 600 /etc/patroni/ctl/{ca.pem,patronictl.pem,patronictl-key.pem}
```
### Установка etcd на хостах:
Действия выполняются на всех хостах кластера `etcd`
#### Скачать дистрибутив etcd
```
cd /tmp
ETCD_VER=v3.5.15
DOWNLOAD_URL=https://github.com/etcd-io/etcd/releases/download
wget ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzf etcd-${ETCD_VER}-linux-amd64.tar.gz
sudo mv etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/
rm -rf etcd-${ETCD_VER}-linux-amd64*
```
#### Настройка пользователей и прав
Создаю пользователя, от которого будет работать служба etcd:
```
sudo useradd --system --home-dir /var/lib/etcd --shell /bin/false etcd || true
sudo mkdir -p /var/lib/etcd
sudo chown -R etcd:etcd /var/lib/etcd
sudo chown -R etcd:etcd /etc/etcd/ssl
```
#### Настройка службы etcd (Создание systemd unit-файлов)
Далее на каждой ноде делаю etcd службой:
```
sudo vim /etc/systemd/system/etcd.service
```
- `etcd-node1`:
```
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
User=etcd
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name etcd-node1 \
  --data-dir /var/lib/etcd \
  --initial-advertise-peer-urls https://192.168.0.171:2380 \
  --listen-peer-urls https://192.168.0.171:2380 \
  --listen-client-urls https://192.168.0.171:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.0.171:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster etcd-node1=https://192.168.0.171:2380,etcd-node2=https://192.168.0.172:2380,etcd-node3=https://192.168.0.173:2380 \
  --initial-cluster-state new \
  --client-cert-auth \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --cert-file=/etc/etcd/ssl/peer-node1.pem \
  --key-file=/etc/etcd/ssl/peer-node1-key.pem \
  --peer-client-cert-auth \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-cert-file=/etc/etcd/ssl/peer-node1.pem \
  --peer-key-file=/etc/etcd/ssl/peer-node1-key.pem
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```
- `etcd-node2`:
```
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
User=etcd
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name etcd-node2 \
  --data-dir /var/lib/etcd \
  --initial-advertise-peer-urls https://192.168.0.172:2380 \
  --listen-peer-urls https://192.168.0.172:2380 \
  --listen-client-urls https://192.168.0.172:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.0.172:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster etcd-node1=https://192.168.0.171:2380,etcd-node2=https://192.168.0.172:2380,etcd-node3=https://192.168.0.173:2380 \
  --initial-cluster-state new \
  --client-cert-auth \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --cert-file=/etc/etcd/ssl/peer-node2.pem \
  --key-file=/etc/etcd/ssl/peer-node2-key.pem \
  --peer-client-cert-auth \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-cert-file=/etc/etcd/ssl/peer-node2.pem \
  --peer-key-file=/etc/etcd/ssl/peer-node2-key.pem
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```
- `etcd-node3`:
```
[Unit]
Description=etcd key-value store
Documentation=https://github.com/etcd-io/etcd
After=network.target

[Service]
User=etcd
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name etcd-node3 \
  --data-dir /var/lib/etcd \
  --initial-advertise-peer-urls https://192.168.0.173:2380 \
  --listen-peer-urls https://192.168.0.173:2380 \
  --listen-client-urls https://192.168.0.173:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.0.173:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster etcd-node1=https://192.168.0.171:2380,etcd-node2=https://192.168.0.172:2380,etcd-node3=https://192.168.0.173:2380 \
  --initial-cluster-state new \
  --client-cert-auth \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --cert-file=/etc/etcd/ssl/peer-node3.pem \
  --key-file=/etc/etcd/ssl/peer-node3-key.pem \
  --peer-client-cert-auth \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-cert-file=/etc/etcd/ssl/peer-node3.pem \
  --peer-key-file=/etc/etcd/ssl/peer-node3-key.pem
Restart=always
RestartSec=10s
LimitNOFILE=40000

[Install]
WantedBy=multi-user.target
```
Далее настраиваю автозапуск службы etcd и ее запускаю:
```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
sudo systemctl status etcd
```
### Проверка работоспособности кластера на клиенте (на первой ноде etcd-node1 через клиентские сертификаты):
Проверка членов кластера:
```
etcdctl --endpoints=https://192.168.0.171:2379,https://192.168.0.172:2379,https://192.168.0.173:2379 \
  --cacert=/home/nazrinrus/etcd-ssl/ca/ca.pem \
  --cert=/home/nazrinrus/etcd-ssl/client/client.pem \
  --key=/home/nazrinrus/etcd-ssl/client/client-key.pem \
  member list --write-out=table
```
Проверка здоровья:
```
etcdctl --endpoints=https://192.168.0.171:2379,https://192.168.0.172:2379,https://192.168.0.173:2379 \
  --cacert=/home/nazrinrus/etcd-ssl/ca/ca.pem \
  --cert=/home/nazrinrus/etcd-ssl/client/client.pem \
  --key=/home/nazrinrus/etcd-ssl/client/client-key.pem \
  endpoint health --write-out=table
```
Статус нод:
```
etcdctl --endpoints=https://192.168.0.171:2379,https://192.168.0.172:2379,https://192.168.0.173:2379 \
  --cacert=/home/nazrinrus/etcd-ssl/ca/ca.pem \
  --cert=/home/nazrinrus/etcd-ssl/client/client.pem \
  --key=/home/nazrinrus/etcd-ssl/client/client-key.pem \
  endpoint status --write-out=table
```
Добавить путь `/usr/local/bin/` для `etcdctl` в `PATH`, если вдруг бинарник не находится
```
export PATH="/usr/local/bin:$PATH"
```
### Установка PostgreSQL
На каждом хосте
#### Добавить репозиторий PostgreSQL (для Ubuntu/Debian)
```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
```
#### Установка пакетов PostgreSQL 14
```
sudo apt-get install -y postgresql-14 postgresql-server-dev-14
```
#### Установка пакетов Patroni и зависимостей
```
sudo apt -y install python3 python3-pip python3-dev python3-psycopg2 libpq-dev
sudo pip3 install psycopg2 --break-system-packages
sudo pip3 install psycopg2-binary --break-system-packages
sudo pip3 install patroni --break-system-packages
sudo pip3 install python-etcd --break-system-packages
```
#### Пользователи и директории
```
sudo useradd -r -s /bin/false postgres
sudo mkdir -p /data/patroni
sudo chown -R postgres:postgres /data/patroni
sudo mkdir -p /etc/patroni
sudo mkdir -p /etc/patroni/ssl

sudo chown -R postgres:postgres /etc/patroni/ctl 2>/dev/null || true
sudo chmod 600 /etc/patroni/ctl/{ca.pem,patronictl.pem,patronictl-key.pem}

# на ноде etcd-node1
sudo chown -R postgres:postgres /etc/patroni/ssl 2>/dev/null || true
sudo chmod 600 /etc/patroni/ssl/patroni1-key.pem
sudo chmod 644 /etc/patroni/ssl/{ca.pem,patroni1.pem}

# на ноде etcd-node2
sudo chown -R postgres:postgres /etc/patroni/ssl 2>/dev/null || true
sudo chmod 600 /etc/patroni/ssl/patroni2-key.pem
sudo chmod 644 /etc/patroni/ssl/{ca.pem,patroni2.pem}
```
### Конфигурация Patroni
#### Настройка на pg-node1 (192.168.0.174)
Файл конфигурации `/etc/patroni/patroni.yml`:
```
sudo tee /etc/patroni/patroni.yml << 'EOF'
scope: postgres-cluster
name: pg-node1
restapi:
  listen: 192.168.0.174:8008
  connect_address: 192.168.0.174:8008
  authentication:
    username: patroni_admin
    password: cii3lu5Oe8ohdoopel
etcd3:
  hosts: 192.168.0.171:2379,192.168.0.172:2379,192.168.0.173:2379
  protocol: https
  cacert: /etc/patroni/ssl/ca.pem
  cert: /etc/patroni/ssl/patroni1.pem
  key: /etc/patroni/ssl/patroni1-key.pem
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_slots: false
      use_pg_rewind: true
      recovery_conf:
        restore_command: pgbackrest --stanza=prodpg archive-get %f %p
      parameters:
        archive_command: pgbackrest --stanza=prodpg archive-push %p
        archive_mode: 'on'
        archive_timeout: 300
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        checkpoint_timeout: 30
        max_connections: 100
        shared_buffers: 256MB
        wal_sync_method: fsync
  initdb:
    - encoding: UTF8
    - data-checksums
    - locale: en_US.UTF-8
  users:
    admin:
      password: Quahchee7Gaidokiex
      options:
        - createrole
        - createdb
  pg_hba:
    - local all all trust
    - host all all 127.0.0.1/32 trust
    - host all all 192.168.0.174/32 trust
    - host all all 192.168.0.175/32 trust
    - host replication replicator 127.0.0.1/32 md5
    - host replication replicator 192.168.0.174/32 md5
    - host replication replicator 192.168.0.175/32 md5
postgresql:
  create_replica_methods:
  - pgbackrest
  - basebackup
  pgbackrest:
      command: pgbackrest --stanza=prodpg --delta --link-all restore
      keep_data: true
      no_params: true
  basebackup:
      max-rate: 1000M
  archive_command: pgbackrest --stanza=prodpg archive-push %p
  listen: 127.0.0.1,192.168.0.174:5432
  connect_address: 192.168.0.174:5432
  data_dir: /postgres/patroni/data
  bin_dir: /usr/lib/postgresql/14/bin
  pgpass: /tmp/pgpass
  recovery_conf:
    restore_command: pgbackrest --stanza=prodpg archive-get %f %p
  authentication:
    replication:
      username: replicator
      password: Tohyohg2Tee2tuo6Ne
    superuser:
      username: postgres
      password: do3Eec7bahChah8uth
  parameters:
    unix_socket_directories: '/var/run/postgresql'
    ssl: 'off'
    listen_addresses: '192.168.0.174,localhost'
  pg_hba:
    - local all all trust
    - host all all 127.0.0.1/32 md5
    - host all all 192.168.0.174/32 md5
    - host all all 192.168.0.175/32 md5
    - host replication replicator 192.168.0.174/32 md5
    - host replication replicator 192.168.0.175/32 md5
    - host replication replicator 127.0.0.1/32 md5
    - host all all 0.0.0.0/0 md5
tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
EOF
```
#### Настройка на pg-node2 (192.168.0.175)
Файл конфигурации `/etc/patroni/patroni.yml`:
```
sudo tee /etc/patroni/patroni.yml << 'EOF'
scope: postgres-cluster
name: pg-node2
restapi:
  listen: 192.168.0.175:8008
  connect_address: 192.168.0.175:8008
  authentication:
    username: patroni_admin
    password: cii3lu5Oe8ohdoopel
etcd3:
  hosts: 192.168.0.171:2379,192.168.0.172:2379,192.168.0.173:2379
  protocol: https
  cacert: /etc/patroni/ssl/ca.pem
  cert: /etc/patroni/ssl/patroni2.pem
  key: /etc/patroni/ssl/patroni2-key.pem
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    postgresql:
      use_slots: false
      use_pg_rewind: true
      recovery_conf:
        restore_command: pgbackrest --stanza=prodpg archive-get %f %p
      parameters:
        archive_command: pgbackrest --stanza=prodpg archive-push %p
        archive_mode: 'on'
        archive_timeout: 300
        wal_level: replica
        hot_standby: "on"
        max_wal_senders: 10
        max_replication_slots: 10
        checkpoint_timeout: 30
        max_connections: 100
        shared_buffers: 256MB
        wal_sync_method: fsync
  initdb:
    - encoding: UTF8
    - data-checksums
    - locale: en_US.UTF-8
  users:
    admin:
      password: Quahchee7Gaidokiex
      options:
        - createrole
        - createdb
  pg_hba:
    - local all all trust
    - host all all 127.0.0.1/32 trust
    - host all all 192.168.0.174/32 trust
    - host all all 192.168.0.175/32 trust
    - host replication replicator 127.0.0.1/32 md5
    - host replication replicator 192.168.0.174/32 md5
    - host replication replicator 192.168.0.175/32 md5
postgresql:
  create_replica_methods:
  - pgbackrest
  - basebackup
  pgbackrest:
      command: pgbackrest --stanza=prodpg --delta --link-all restore
      keep_data: true
      no_params: true
  basebackup:
      max-rate: 1000M
  archive_command: pgbackrest --stanza=prodpg archive-push %p
  listen: 127.0.0.1,192.168.0.175:5432
  connect_address: 192.168.0.175:5432
  data_dir: /postgres/patroni/data
  bin_dir: /usr/lib/postgresql/14/bin
  pgpass: /tmp/pgpass
  recovery_conf:
    restore_command: pgbackrest --stanza=prodpg archive-get %f %p
  authentication:
    replication:
      username: replicator
      password: Tohyohg2Tee2tuo6Ne
    superuser:
      username: postgres
      password: do3Eec7bahChah8uth
  parameters:
    unix_socket_directories: '/var/run/postgresql'
    ssl: 'off'
    listen_addresses: '192.168.0.175,localhost'
  pg_hba:
    - local all all trust
    - host all all 127.0.0.1/32 md5
    - host all all 192.168.0.174/32 md5
    - host all all 192.168.0.175/32 md5
    - host replication replicator 192.168.0.174/32 md5
    - host replication replicator 192.168.0.175/32 md5
    - host replication replicator 127.0.0.1/32 md5
    - host all all 0.0.0.0/0 md5
tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false
EOF
```
#### Проверка доступности etcd с хостов pg-node1 или pg-node2
На pg-node1 или pg-node2
```
sudo su
export ETCDCTL_ENDPOINTS=https://192.168.0.171:2379,https://192.168.0.172:2379,https://192.168.0.173:2379
export ETCDCTL_CACERT=/etc/patroni/ssl/ca.pem
export ETCDCTL_CERT=/etc/patroni/ssl/patroni1.pem
export ETCDCTL_KEY=/etc/patroni/ssl/patroni1-key.pem

root@pg-node1:/tmp# etcdctl endpoint health --write-out=table
+----------------------------+--------+-------------+-------+
|          ENDPOINT          | HEALTH |    TOOK     | ERROR |
+----------------------------+--------+-------------+-------+
| https://192.168.0.173:2379 |   true | 15.741851ms |       |
| https://192.168.0.171:2379 |   true |  17.11622ms |       |
| https://192.168.0.172:2379 |   true | 18.106104ms |       |
+----------------------------+--------+-------------+-------+
root@pg-node1:/tmp# etcdctl endpoint status --write-out=table
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://192.168.0.171:2379 | 7ac5068ed9e95ece |  3.5.15 |   20 kB |     false |      false |         2 |         15 |                 15 |        |
| https://192.168.0.172:2379 | bf54abc952f1978e |  3.5.15 |   20 kB |     false |      false |         2 |         15 |                 15 |        |
| https://192.168.0.173:2379 | 1697d14554a0b10d |  3.5.15 |   20 kB |      true |      false |         2 |         15 |                 15 |        |
+----------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```
#### Systemd unit для Patroni
файл `/etc/systemd/system/patroni.service` на обоих узлах:
```
sudo tee /etc/systemd/system/patroni.service << 'EOF'
[Unit]
Description=Patroni PostgreSQL HA cluster
After=network.target etcd.service
Wants=network.target

[Service]
Type=simple
User=postgres
Group=postgres
Restart=always
RestartSec=10
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
TimeoutSec=30

[Install]
WantedBy=multi-user.target
EOF
```
Перезагрузка `systemd`
```
sudo systemctl daemon-reload
```
#### Запуск Patroni на узлах (сначала на первом, потом на втором)
```
sudo systemctl start patroni
sudo systemctl status patroni
```
Просмотреть состояние кластера можно командой
```
sudo patronictl -c /etc/patroni/patroni.yml list
```
Для failover:
```
sudo patronictl -c /etc/patroni/patroni.yml failover
```
### Установка HAProxy и Keepalived
#### Настройка узлов
```
# Добавить параметр в /etc/sysctl.conf
echo "net.ipv4.ip_nonlocal_bind = 1" | sudo tee -a /etc/sysctl.conf

# Применить изменения
sudo sysctl -p

# Разрешить протокол VRRP (IP protocol 112)
sudo ufw allow proto vrrp from 192.168.0.0/24
sudo ufw allow 22/tcp
# Разрешить порты для HAProxy (позже)
sudo ufw allow 5432/tcp comment 'PostgreSQL'
sudo ufw allow 7000/tcp comment 'HAProxy Stats'
```
#### Установка и настройка HAProxy (на обоих узлах)
Конфигурация HAProxy должна быть идентичной на обоих узлах
```
sudo apt update && sudo apt install haproxy -y
```
Настройка HAProxy (haproxy.cfg)
```
sudo mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.orig
sudo tee /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log    local0
    log /dev/log    local1 notice
    maxconn 100000
    daemon
    user haproxy
    group haproxy
    chroot /var/lib/haproxy/
    stats socket /run/haproxy/info.sock mode 666 level user

defaults
    fullconn 100000
    maxconn 100000
    log global
    mode tcp
    retries 2
    timeout connect 4s
    timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /
    stats auth haproxystats:phich7ahs4ip7eW0wo

listen postgres_master
    bind *:5432
    option httpchk /master
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node1 192.168.0.174:5432 maxconn 20000 check port 8008
    server pg-node2 192.168.0.175:5432 maxconn 20000 check port 8008

listen postgres_replica
    bind *:5433
    option httpchk /read-only
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node1 192.168.0.174:5432 maxconn 20000 check port 8008
    server pg-node2 192.168.0.175:5432 maxconn 20000 check port 8008

listen postgres_replica_timeout
    bind *:5434
    option httpchk /read-only
    http-check expect status 200
    timeout client 30m
    timeout server 30m
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server pg-node1 192.168.0.174:5432 maxconn 20000 check port 8008
    server pg-node2 192.168.0.175:5432 maxconn 20000 check port 8008
EOF
```
Запуск HAProxy
```
# Проверить конфигурацию
sudo haproxy -f /etc/haproxy/haproxy.cfg -c

# Запустить и добавить в автозагрузку
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy
```
#### Установка и настройка Keepalived
Установка Keepalived (на обоих узлах):
```
sudo apt install keepalived -y
```
Создание скрипта проверки здоровья HAProxy (на обоих узлах):
```
sudo tee /etc/keepalived/check_haproxy.sh << 'EOF'
#!/bin/bash
VIP=127.0.0.1
MASTER_PORT=5432
REPLICAS_PORT=5433
echo > /dev/tcp/${VIP}/${MASTER_PORT} && exit 0 || exit 1
EOF

sudo chmod +x /etc/keepalived/check_haproxy.sh
```
Настройка Keepalived на haproxy-node1 (Master):
```
sudo mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.orig

sudo tee /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    enable_script_security
    script_user root
}


vrrp_script chk_myscript {
    script /etc/keepalived/check-haproxy.sh
    interval 3
    fall 2
    rise 3
}

vrrp_instance VI_1 {
    state BACKUP
    virtual_router_id 14
    priority 100
    advert_int 1
    interface enp0s3
    nopreempt
    authentication {
        auth_type PASS
        auth_pass Eikohheez4chaethoh
    }
    virtual_ipaddress {
        192.168.0.178/24
    }
    track_script {
        chk_myscript
    }
}
EOF
```
Настройка Keepalived на haproxy-node2 (Backup):
```
# Создайте резервную копию и новый файл
sudo mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.orig

sudo tee /etc/keepalived/keepalived.conf << 'EOF'
global_defs {
    enable_script_security
    script_user root
}


vrrp_script chk_myscript {
    script /etc/keepalived/check-haproxy.sh
    interval 3
    fall 2
    rise 3
}

vrrp_instance VI_1 {
    state BACKUP
    virtual_router_id 14
    priority 100
    advert_int 1
    interface enp0s3
    nopreempt
    authentication {
        auth_type PASS
        auth_pass Eikohheez4chaethoh
    }
    virtual_ipaddress {
        192.168.0.178/24
    }
    track_script {
        chk_myscript
    }
}
EOF
```
Запуск Keepalived (на обоих узлах):
```
sudo systemctl enable keepalived
sudo systemctl start keepalived
sudo systemctl status keepalived
```
#### Тестирование падения мастера HAProxy
Мониторинг VIP на нодах HAProxy:
```
# Терминал на haproxy-node1, VIP должен быть тут
watch -n 1 'ip addr show enp0s3 | grep 192.168.0.178'
# Терминал на haproxy-node2, VIP отсутствует
watch -n 1 'ip addr show enp0s3 | grep 192.168.0.178'
```
В третьем терминале:
```
while true; do PGPASSWORD='do3Eec7bahChah8uth' psql -h 192.168.0.178 -p 5432 -U postgres -t -c "SELECT inet_server_addr();" 2>/dev/null | head -1; sleep 2; done
```
В еще одном терминале подключиться к `haproxy-node1`, отключить службу `haproxy`:
```
sudo systemctl stop keepalived
```
VIP переехал на `haproxy-node2`, однако при отключении службы `haproxy` VIP не переезжает, доработать скрипт

#### Тестирование падения мастер-ноды PostgreSQL - failover
- В первом терминале запустил цикл с отправкой на VIP:5432 (rw) запроса о IP-адресе сервера и его статусе:
```
while true; do PGPASSWORD='do3Eec7bahChah8uth' psql -h 192.168.0.178 -p 5432 -U postgres -t -c "SELECT inet_server_addr() || ' | ' || CASE WHEN pg_is_in_recovery() THEN 'REPLICA' ELSE 'MASTER' END;" 2>/dev/null | head -1; sleep 2; done
```
- Во втором терминале запустил цикл с отправкой на VIP:5433 (ro) запроса о IP-адресе сервера и его статусе:
```
while true; do PGPASSWORD='do3Eec7bahChah8uth' psql -h 192.168.0.178 -p 5433 -U postgres -t -c "SELECT inet_server_addr() || ' | ' || CASE WHEN pg_is_in_recovery() THEN 'REPLICA' ELSE 'MASTER' END;" 2>/dev/null | head -1; sleep 2; done
```
- В третьем терминале отключаю службу Patroni на мастер-ноде, имитируя падение с ней связи:
```
sudo systemctl stop patroni
```

### Бэкапирование
### Установка pgBackRest
на хостах: `pg-node1`, `pg-node2`, `pg-node1`, `pgbackrest-node`
```
sudo apt update
sudo apt install pgbackrest -y
```
Конфигурация на `pg-node1` и `pg-node2` - `sudo vim /etc/pgbackrest.conf`:
```
[global]
archive-async=y
archive-push-queue-max=10GB
compress-type=lz4
log-level-file=detail
process-max=12
repo1-host=192.168.0.170
repo1-retention-archive=2
repo1-retention-full=7
spool-path=/var/lib/pgbackrest/spool

[global:archive-get]
process-max=2

[global:archive-push]
compress-level=3
process-max=2

[global:restore]
process-max=2

[prodpg]
db1-socket-path=/var/run/postgresql
pg1-path=/postgres/patroni/data
```
- `sudo chown postgres:postgres /etc/pgbackrest.conf`
- `sudo chmod 664 /etc/pgbackrest.conf`

Конфигурация на `pgbackrest-node` - `sudo vim /etc/pgbackrest.conf`:
```
[global]
process-max=2
repo1-path=/data/backup/pgbackrest
repo1-retention-full=3
repo1-retention-full-type=count
repo1-retention-archive=1
start-fast=y
compress-type=lz4
log-level-console=detail
log-level-file=detail
repo1-block=y
repo1-bundle=y
repo1-bundle-limit=256MiB
repo1-bundle-size=1024MiB
repo1-hardlink=y
```
- `sudo mkdir /etc/pgbackrest.conf.d`
- `sudo vim /etc/pgbackrest.conf.d/prodpg.conf`
```
[prodpg]
pg1-host=192.168.0.174
pg1-path=/postgres/patroni/data
db1-socket-path=/var/run/postgresql
pg2-host=192.168.0.175
pg2-path=/postgres/patroni/data
db2-socket-path=/var/run/postgresql
backup-standby=y
repo1-retention-full=2
repo1-retention-archive=1
```
Пользователь и группа:
```
sudo groupadd -g 1002 pgbackrest
sudo useradd -u 1002 -g 1002 -m -d /home/pgbackrest -s /bin/bash -c ",,," pgbackrest
sudo chown pgbackrest:pgbackrest -R /etc/pgbackrest.conf.d
sudo chown pgbackrest:pgbackrest /etc/pgbackrest.conf
```
Директория для хранения бэкапов:
```
sudo mkdir -p /data/backup/pgbackrest
sudo chown pgbackrest:pgbackrest -R /data/backup/pgbackrest
sudo chmod 751 -R /data/backup/pgbackrest
```
Запуск в кроне `sudo vim /etc/cron.d/pgbackrest` на `pgbackrest-node`:
```
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

0 2 * * * pgbackrest /home/pgbackrest/backup_pg.sh prodpg > /dev/null
```
текст скрипта `vim /home/pgbackrest/backup_pg.sh`:
```
#!/bin/bash

set -o pipefail

STANZAS=( "$@" )

if [[ -z $STANZAS ]];then printf "Specify a list of stanzas.\nExample: ./backup_pg.sh prodpg testpg\n" && exit -1;fi

if [[ `date +%u` -eq 1 ]]; then type=full; else type=incr; fi

for STANZA in "${STANZAS[@]}"
do
    pgbackrest --config-include-path=/etc/pgbackrest.conf.d/ --stanza=${STANZA} backup --log-level-console=info --type=$type --archive-copy 2>&1 | tee /tmp/pgbackrest_errors
    if [[ $? -ne 0 ]]
    then
/usr/sbin/sendmail nazrinrus@gmail.com << EOF
subject: Failed to backup ${STANZA}
from: pg@$(hostname)

What happened:
`cat /tmp/pgbackrest_errors`

Contents of log file:
`tail -n 50 /var/log/pgbackrest/${STANZA}-backup.log || echo "Unable to read log."`
EOF
    fi
done

exit 0
```
Создать станзу:
```
pgbackrest --config-include-path=/etc/pgbackrest.conf.d/ --stanza=prodpg --log-level-console=info stanza-create
pgbackrest --config-include-path=/etc/pgbackrest.conf.d/ --stanza=prodpg --log-level-console=info check
pgbackrest --config-include-path=/etc/pgbackrest.conf.d/ --stanza=prodpg backup --log-level-console=info --type=full --archive-copy
```

sudo wget https://edu.postgrespro.ru/demo-medium.zip -O /tmp/demo-medium.zip
sudo unzip /tmp/demo-medium.zip -d /tmp/
PGPASSWORD='do3Eec7bahChah8uth' psql -h 192.168.0.178 -p 5432 -U postgres < /tmp/demo-medium-20170815.sql
