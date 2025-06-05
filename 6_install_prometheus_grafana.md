## Установка Prometheus и Grafana
### Предварительная настройка хостов:
В качестве хоста будет выступать виртуальная машина VirtualBox, на которой настроен сетевой мост, объединяющий в подсеть `192.168.0.0/24`

- На хосте должен быть установлен SSH-сервер;
- Передать на хосты публичный SSH-ключ управляющей машины `ssh-copy-id nazrinrus@192.168.0.161`, при необходимости удаляем 
данные о предыдущем хосте `ssh-keygen -f "/home/nazrinrus/.ssh/known_hosts" -R "192.168.0.161"`;
- Настройка имени сервера и файла `/etc/hosts`
```
sudo hostnamectl set-hostname prometheus-server
sudo vim /etc/hosts
127.0.0.1 localhost prometheus-server
127.0.1.1 prometheus-server
192.168.0.161 prometheus-server
```
- Для перехода по SSH хосты должны иметь статичные ip-адреса (можно и динамичные, но они часто меняются при перезагурке/
удалении/добавлении новых виртуальных машин). Настроить в `netplan`:
1. Настроить на виртуальной машине тип подключения `Сетевой мост`, тогда она будет в локальной сети роутера;
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
        - 192.168.0.161/24
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
        macaddress: 08:00:27:b8:50:8c
```
4. Применить `sudo netplan apply`
### Установка Prometheus
Релизы пакетов можно посмотреть [тут](https://github.com/prometheus/prometheus/releases/)
#### Скачать и установить пакет
```
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v3.1.0/prometheus-3.1.0.linux-386.tar.gz
sudo tar -xzf prometheus*.tar.gz
cd prometheus*/
```
После этого переместите два исполняемых файла Prometheus в каталог `/usr/local/bin`:
```
sudo mv prometheus /usr/local/bin
sudo mv promtool /usr/local/bin
```
Создать директорию для конфигурационных файлов Prometheus и поместить в нее файл конфигурации:.
```
sudo mkdir /etc/prometheus
sudo mv prometheus.yml /etc/prometheus
```
Еще нам нужна папка, в которой будет находиться база данных Prometheus (она работает на движке TSDB):
```
sudo mkdir /var/lib/prometheus
```
#### Создание группы и пользователя `prometheus` и предоставление прав
```
sudo groupadd --system prometheus
sudo useradd -s /sbin/nologin --system -g prometheus prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
sudo chown prometheus:prometheus /etc/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus
```
#### Создание службы Prometheus
`sudo vim /etc/systemd/system/prometheus.service`
```
[Unit]
Description=Background service of Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
	--config.file /etc/prometheus/prometheus.yml \
	--storage.tsdb.path /var/lib/prometheus/ 
[Install]
WantedBy=multi-user.target
```
```
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```
**Веб-интерфейс по адресу `http://127.0.0.1:9090` - с локального хоста или по адресу `http://192.168.0.161:9090` - с удаленного хоста**
#### Открыть порт 9090 (в случае необходимости)
`sudo ufw allow 9090/tcp`
### Установка Grafana
С официального сайта Grafana дистрибутив не скачать, необходимо воспользоваться зеркалом, например
`https://mirrors.cloud.tencent.com/grafana/apt/pool/main/g/grafana-enterprise/`
#### Скачать и установить пакеты
```
cd /tmp
wget https://mirrors.cloud.tencent.com/grafana/apt/pool/main/g/grafana-enterprise/grafana-enterprise_11.5.1_amd64.deb
sudo apt install adduser libfontconfig1 musl -y
sudo dpkg -i grafana-enterprise_11.5.1_amd64.deb
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
systemctl status grafana-server
```
**Веб-интерфейс по адресу `http://127.0.0.1:3000` - с локального хоста или по адресу `http://192.168.0.161:3000` - с удаленного хоста**

По умолчанию логин и пароль: `admin`
### Подключение Grafana к Prometheus
В web-интерфейсе Grafana в разделе `Data Sources` щелкните по ссылке `Add your first data source` и в открывшемся списке 
выберите `Prometheus`. Затем в поле `Connections` введите `http://127.0.0.1:9090`. Затем в нижней части экрана нажмите `Save & Test`. 
Должно появиться сообщение `"Successfully queried the Prometheus API"`
#### Добавление дашборда
Вернитесь на начальную страницу Web-интерфейса Grafana и в разделе `Dashboards` щелкните по ссылке `Create your first dashboard`.
В открывшемся окне щелкните по ссылке `Import Dashboard` и затем нажмите на кнопку `Discard`. Откроется окно `Import Dashboard`. 
В этом окне в поле `Find and import dashboards for common applications at grafana.com/dashboards` введите номер дашбоарда `3662` и нажмите на кнопку `Load`.
Затем в следующем окне в поле Prometheus выберите настроенный вами в п.2 экземпляр Prometheus и нажмите на кнопку `Import`. 
### Установка Alertmanager
Список релизов Alertmanager по адресу: `https://github.com/prometheus/alertmanager/releases/`
#### Скачать и установить пакеты
```
cd /tmp
wget https://github.com/prometheus/alertmanager/releases/download/v0.28.0/alertmanager-0.28.0.linux-386.tar.gz
tar -xvzf alertmanager-0.28.0.linux-386.tar.gz
cd alertmanager-0.28.0.linux-386
# Создаем папку конфигурации и копируем в нее файл конфигурации
sudo mkdir /etc/alertmanager
sudo cp alertmanager.yml /etc/alertmanager
# Создаем папку для данных Alertmanager
sudo mkdir /var/lib/prometheus/alertmanager
# Копируем исполняемые файлы Alertmanager в стандартную папку:
sudo cp amtool alertmanager /usr/local/bin/
```
#### Создаем пользователя, от имени которого будет работать Alertmanager и выдаем ему права на папки:
```
sudo groupadd --system alertmanager
sudo useradd -s /sbin/nologin --system -g alertmanager alertmanager
sudo chown -R alertmanager:alertmanager /etc/alertmanager
sudo chown -R alertmanager:alertmanager /var/lib/prometheus/alertmanager
```
#### Настраиваем Alertmanager как службу, переводим в режим автозапуска и запускаем ее:
`sudo vim /etc/systemd/system/alertmanager.service`
```
[Unit]
Description=Alertmanager Service
After=network.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
         --config.file=/etc/alertmanager/alertmanager.yml \
         --storage.path=/var/lib/prometheus/alertmanager \
         --cluster.advertise-address=127.0.0.1:9093
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl daemon-reload
sudo systemctl enable alertmanager
sudo systemctl start alertmanager
systemctl status alertmanager
```
И в окне Web-браузера подключаемся к Web-интерфейсу Alertmanager: `http://127.0.0.1:9093`
### Установка Node Exporter и получение данных о состоянии сервера Linux
Релизы находятся по адресу: `https://github.com/prometheus/node_exporter/releases`
#### Скачать и установить пакеты
```
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.9.0/node_exporter-1.9.0.linux-386.tar.gz
tar -xvf node_exporter-1.9.0.linux-386.tar.gz
sudo mv node_exporter-1.9.0.linux-386/node_exporter /usr/local/bin/
```
#### Создать учетную запись, от имени которой будет работать Node Exporter:
```
sudo useradd -rs /bin/false node_exporter
```
#### Настройка Node Exporter службой Linux
`sudo vim /etc/systemd/system/node_exporter.service`
```
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
sudo systemctl status node_exporter
```
Можно проверить в браузере информацию о метриках, которые предоставляет экпортер: `http://127.0.0.1:9100/metrics` (`http://192.168.0.161:9100/metrics`)
### Подключение Node Explorer к Prometheus
Для подключения Node Explorer к Prometheus добавить в конец файла конфигурации Prometheus: `sudo vim /etc/prometheus/prometheus.yml`:
```
  - job_name: node
    static_configs:
      - targets: ['localhost:9100']
```
```
sudo systemctl restart prometheus
sudo systemctl status prometheus
```
Затем проверяем в Web-интерфейсе Prometheus, что Node Exporter виден как цель мониторинга Prometheus: `http://192.168.0.161:9090/targets`

Мы можем, например, получить информацию о файловой системе на ноде:
- Перейти в Web-интерфейсе Prometheus, во вкладке `Query` (`http://192.168.0.161:9090/query`)
- Добавить запрос `Add Query` вида `node_filesystem_avail_bytes`, нажать `Execute`

### Отображение данных Node Exporter в дашбоарде Grafana
Перейдите в Web-интерфейс Grafana по адресу `http://192.168.0.161:3000/`
- Перейти в окно импорта дашбордов: `Create your first dashboard`->`Import dashboard`->`Discard` или `http://192.168.0.161:3000/dashboard/import`
- В поле `Find and import dashboards for common applications at grafana.com/dashboards` введите номер `11074` 
(стандартный дашбоард для Node Exporter) и нажмите на кнопку `Load`
- В окне `Importing Dashboards from Grafana.com` в поле `VictoriaMetrics` выберите `Prometheus` и нажмите на кнопку `Import`
### Включение дополнительного коллектора logind в Node Exporter
Убедиться, что `logind` не установлен `http://192.168.0.161:9100/metrics` проверить на отсутствие слова `logind` 
1. Отредактировать файл конфигурации службы Node Exporter `sudo vim /etc/systemd/system/node_exporter.service`, дописав 
к строке запуска службы `node_exporter` параметр:
```
\
    --collector.logind
```
должен принять вид
```
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
    --collector.logind

[Install]
WantedBy=multi-user.target
```
2. Перезапустить службу
```
sudo systemctl daemon-reload
sudo systemctl start node_exporter
systemctl status node_exporter
```
Обновить web-интерфейс и снова поискать `logind`
### Применение Postgres Exporter для мониторинга базы данных на сервере Postgres
- В качестве PostgreSQL сервера будет использоваться ВМ с адресом `192.168.0.108`
- Конфигурационные файлы настроены на подключение хостов с подсети `192.168.0.0./24`
- Пользователь `postgres` имеет пароль `postgres`
- Тестовая база `test_db`
1. Создать пользователя, от имени которого Postgres Exporter будет подключаться к Postgres и дать права мониторинга:
```
create user postgresql_exporter with password 'password';
GRANT pg_monitor TO postgresql_exporter;
```
2. Установка и настройка Postgres Exporter
Нужный релиз Postgres Exporter можно найти по адресу `https://github.com/prometheus-community/postgres_exporter/releases/`
```
cd /tmp
wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.16.0/postgres_exporter-0.16.0.linux-amd64.tar.gz
tar -zxvf postgres_exporter-0.16.0.linux-amd64.tar.gz
sudo mv postgres_exporter-0.16.0.linux-amd64/postgres_exporter /usr/local/bin/
```
3. Создаем службу экспортера `sudo vim /etc/systemd/system/postgres-exporter.service`
```
[Unit]
Description=Prometheus PostgreSQL Exporter
After=network.target

[Service]
Type=simple
Restart=always
User=postgres
Group=postgres
Environment=DATA_SOURCE_NAME="postgresql://postgresql_exporter:password@localhost/test_db?sslmode=disable"
ExecStart=/usr/local/bin/postgres_exporter

[Install]
WantedBy=multi-user.target
```
```
sudo systemctl daemon-reload
sudo systemctl start postgres-exporter
sudo systemctl enable postgres-exporter
sudo systemctl status postgres-exporter
```
В адресной строке браузера проверяем доступность метрик Postgres Exporter: `http://192.168.0.108:9187/metrics`
4. Подключаем Postgres Exporter: `sudo vim /etc/prometheus/prometheus.yml` (на хосте с Prometheus `192.168.0.161`)

В конец файла в раздел `scrape_configs` дописываем строки:
```
  - job_name: 'postgres_exporter'
    scrape_interval: 15s
    static_configs:
      - targets: ['192.168.0.108:9187']
```
```
sudo systemctl restart prometheus 
systemctl status prometheus
```
Проверить список целей для Prometheus: `http://192.168.0.161:9090/targets`, в списке должен появиться Postgres Exporter с состоянием `Up`
5. Подключение дашбоарда Grafana для мониторинга Postgres

На странице импорта дашбоардов Grafana `http://192.168.0.161:3000/dashboard/import`, в поле `Find and import dashboards for common applications at grafana.com/dashboards` 
введите номер дашбоарда `9628` и нажмите на кнопку `Load`, а затем выберите в списке источников `Prometheus` и нажмите `Import`.

PGPASSWORD=postgres pgbench -d test_db -T 300 -c 100 -h 192.168.0.108 -U postgres
