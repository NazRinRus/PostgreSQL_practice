### Установка и настройка PostgreSQL в контейнере Docker
1. Создана виртуальная машина Yandex для тестирования:
```
CPU: 2 ядра 
RAM: 4 ГБ
IP: 10.10.3.2
ОС: Ubuntu 20.04
```
2. Установка Docker Engine:

https://docs.docker.com/engine/install/ubuntu/
```
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
3. Создание каталога `/var/lib/postgres` для хранения данных:
```
sudo mkdir /var/lib/postgres
```
4. Развернуть контейнер с PostgreSQL 14, смонтировав в него `/var/lib/postgres`:
```
sudo docker run --name postgres-14 -p 5432:5432 -e POSTGRES_USER=nazrinrus -e POSTGRES_PASSWORD=nazrinrus -e POSTGRES_DB=test_db -d -v /var/lib/postgres:/var/lib/postgresql/data postgres:14.17 
```
5. Развернуть контейнер с клиентом PostgreSQL:
```
mkdir docker_postgresql_client
cd docker_postgresql_client
```
Создание `vim dockerfile`:
```
FROM alpine:latest

RUN apk update && apk add --no-cache postgresql-client

RUN apk add --no-cache bash

RUN apk add --no-cache vim
```
Построение и запуск контейнера:
```
sudo docker build -t postgres-client .
sudo docker run --rm -it postgres-client /bin/bash
```
Поодключение к БД в контейнере-сервере `postgres-14`:
```
PGPASSWORD=nazrinrus psql -h 10.10.3.2 -U nazrinrus -d test_db
```
6. Подключиться из контейнера с клиентом к контейнеру с сервером и создайте таблицу с данными о перевозках:

Запуск контейнера `postgres-client`:
```
sudo docker run --rm -it postgres-client /bin/bash
```
Создание файла-скрипта - `3f91d91bb9db:/# vim query.sql`
```
create table shipments(id serial, product_name text, quantity int, destination text);

insert into shipments(product_name, quantity, destination) values('bananas', 1000, 'Europe');
insert into shipments(product_name, quantity, destination) values('bananas', 1500, 'Asia');
insert into shipments(product_name, quantity, destination) values('bananas', 2000, 'Africa');
insert into shipments(product_name, quantity, destination) values('coffee', 500, 'USA');
insert into shipments(product_name, quantity, destination) values('coffee', 700, 'Canada');
insert into shipments(product_name, quantity, destination) values('coffee', 300, 'Japan');
insert into shipments(product_name, quantity, destination) values('sugar', 1000, 'Europe');
insert into shipments(product_name, quantity, destination) values('sugar', 800, 'Asia');
insert into shipments(product_name, quantity, destination) values('sugar', 600, 'Africa');
insert into shipments(product_name, quantity, destination) values('sugar', 400, 'USA');
```
Запуск скрипта на удаленной базе в контейнере-сервере:
```                                                                                                                                                                                           
3f91d91bb9db:/# 
```
Вывод результата запроса - `3f91d91bb9db:/# vim query_out.txt`
```
CREATE TABLE 
INSERT 0 1 
INSERT 0 1 
INSERT 0 1 
INSERT 0 1 
INSERT 0 1 
INSERT 0 1 
INSERT 0 1 
INSERT 0 1 
INSERT 0 1 
INSERT 0 1
```
7. Подключиться к контейнеру с сервером с ноутбука или компьютера:
```
nazrinrus@nazrinrusPC:~/Рабочий стол/PGMech$ PGPASSWORD=nazrinrus psql -h 10.10.3.2 -U nazrinrus -d test_db
Секундомер включён.
psql (17.4 (Ubuntu 17.4-1.pgdg22.04+2), сервер 14.17 (Debian 14.17-1.pgdg120+1))
Введите "help", чтобы получить справку.

nazrinrus@test_db=# \dt
             Список отношений
 Схема  |    Имя    |   Тип   | Владелец  
--------+-----------+---------+-----------
 public | shipments | таблица | nazrinrus
(1 строка)

nazrinrus@test_db=# table shipments;
 id | product_name | quantity | destination 
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | bananas      |     1500 | Asia
  3 | bananas      |     2000 | Africa
  4 | coffee       |      500 | USA
  5 | coffee       |      700 | Canada
  6 | coffee       |      300 | Japan
  7 | sugar        |     1000 | Europe
  8 | sugar        |      800 | Asia
  9 | sugar        |      600 | Africa
 10 | sugar        |      400 | USA
(10 строк)

Время: 25,808 мс
```
8. Удалить контейнер с сервером и создать его заново:
```
s22157660@s22157660-01:~/docker_postgresql_client$ sudo docker stop postgres-14
postgres-14
s22157660@s22157660-01:~/docker_postgresql_client$ sudo docker rm postgres-14
postgres-14
s22157660@s22157660-01:~/docker_postgresql_client$ sudo docker run --name postgres-14 -p 5432:5432 -e POSTGRES_USER=nazrinrus -e POSTGRES_PASSWORD=nazrinrus -e POSTGRES_DB=test_db -d -v /var/lib/postgres:/var/lib/postgresql/data postgres:14.17 
733c2743ba7de6bc87439249f2f6746f69221700d0cbf3649bb9b3b72515d6be
s22157660@s22157660-01:~/docker_postgresql_client$ sudo docker ps
CONTAINER ID   IMAGE            COMMAND                  CREATED          STATUS          PORTS                                         NAMES
733c2743ba7d   postgres:14.17   "docker-entrypoint.s…"   15 seconds ago   Up 15 seconds   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp   postgres-14
```
9. Проверить, что данные остались на месте:
```
nazrinrus@nazrinrusPC:~/Рабочий стол/PGMech$ PGPASSWORD=nazrinrus psql -h 10.10.3.2 -U nazrinrus -d test_db
Секундомер включён.
psql (17.4 (Ubuntu 17.4-1.pgdg22.04+2), сервер 14.17 (Debian 14.17-1.pgdg120+1))
Введите "help", чтобы получить справку.

nazrinrus@test_db=# table shipments;
 id | product_name | quantity | destination 
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | bananas      |     1500 | Asia
  3 | bananas      |     2000 | Africa
  4 | coffee       |      500 | USA
  5 | coffee       |      700 | Canada
  6 | coffee       |      300 | Japan
  7 | sugar        |     1000 | Europe
  8 | sugar        |      800 | Asia
  9 | sugar        |      600 | Africa
 10 | sugar        |      400 | USA
(10 строк)

Время: 26,726 мс
```
### Вариант с Docker Compose
Создание рабочей директории:
```
mkdir docker_postgresql_client && cd docker_postgresql_client
```
Создание `Dockerfile` клиента - `vim Dockerfile`
```
FROM alpine:latest

RUN apk update && apk add --no-cache postgresql-client

RUN apk add --no-cache bash

RUN apk add --no-cache vim
```
Создание `docker-compose.yml` с сервисами `postgres-server`, `postgres-server` - `vim docker-compose.yml`:
```
services:
  postgres-server:
    container_name: postgres-server
    image: postgres:14.17
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: "nazrinrus"
      POSTGRES_PASSWORD: "nazrinrus"
      POSTGRES_DB: "test_db"
      PGDATA: "/var/lib/postgresql/data"
    volumes:
      - /var/lib/postgres:/var/lib/posttgresql/data
  
  postgres-client:
    build:
      context: .
      dockerfile: Dockerfile
    image: postgres-client:latest
    container_name: postgres-client
    tty: true
    stdin_open: true
    
```
Построение образов и запуск контейнеров:
```
docker pull postgres:14.17
docker-compose build
docker-compose up
```
