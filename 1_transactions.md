1. Создана виртуальная машина Oracle VM VirtualBOX для тестирования:
```
CPU: 2 ядра 
RAM: 4 ГБ
IP: 192.168.0.104
ОС: Ubuntu
```
2. Добавлен публичный SSH ключ:
```
ssh-copy-id nazrinrus@192.168.0.104
```
3. Установка PostgreSQL:
```
sudo apt install curl ca-certificates
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

. /etc/os-release
sudo sh -c "echo 'deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $VERSION_CODENAME-pgdg main' > /etc/apt/sources.list.d/pgdg.list"

sudo apt update
PGVERSION=17
sudo apt install postgresql-${PGVERSION?} postgresql-${PGVERSION?}-repack postgresql-${PGVERSION?}-dbgsym postgresql-client-${PGVERSION?}
```
4. Подключение к PostgreSQL двумя сессиями:
```
ssh nazrinrus@192.168.0.104
sudo -iu postgres
psql
```
5. Работа с транзакциями:

Отключить автоматическую фиксацию транзакций в обеих сессиях:
```
air_db=# \set AUTOCOMMIT off
air_db=# \echo :AUTOCOMMIT
off
```
В первой сессии создайте таблицу `shipments` (перевозки) и добавьте в неё данные:
```
air_db=# CREATE TABLE shipments(id serial, product_name text, quantity int, destination text);
CREATE TABLE
air_db=*# INSERT INTO shipments(product_name, quantity, destination) VALUES ('bananas', 1000, 'Europe');
INSERT 0 1
air_db=*# INSERT INTO shipments(product_name, quantity, destination) VALUES ('coffee', 500, 'USA');
INSERT 0 1
air_db=*# COMMIT;
COMMIT
```
6. Изучение уровней изоляции:

Уровень изоляции:
```
air_db=# SHOW TRANSACTION ISOLATION LEVEL;
 transaction_isolation 
-----------------------
 read committed
(1 строка)
```
В первой сессии добавлена новая запись **(команды сессии №1 будут начинаться с `1|`, №2 - `2|`)** без фиксации транзакции:
```
1| air_db=# INSERT INTO shipments(product_name, quantity, destination) VALUES ('sugar', 300, 'Asia');
1| INSERT 0 1
```
Проверка содержимого таблицы `shipments` во второй сессии:
```
2| air_db=# table shipments;
 id | product_name | quantity | destination 
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
(2 строки)
```
Во второй сессии не видно запись, добавленную (но не зафиксированную) первой сессией. Такое поведение свойственно для уровня 
изоляции `read committed`, предотвращающее аномалию `Read uncommited`

При фиксации изменений в первой сессии:
```
1| air_db=*# COMMIT;
COMMIT
```
изменения видны во второй сессии:
```
2| air_db=*# table shipments;
 id | product_name | quantity | destination 
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia
(3 строки)
```
7. Эксперименты с уровнем изоляции `Repeatable Read`:

Начать новые транзакции в обеих сессиях с уровнем изоляции `repeatable read`:
```
1| air_db=# set transaction isolation level repeatable read;
SET

2|set transaction isolation level repeatable read;
SET
```
В первой сессии добвить запись:
```
1| air_db=# INSERT INTO shipments(product_name, quantity, destination) VALUES ('bananas', 2000, 'Africa');
INSERT 0 1
```
Проверка содержимого таблицы `shipments` во второй сессии:
```
2| air_db=*# table shipments;
 id | product_name | quantity | destination 
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia
(3 строки)
```
Во второй сессии не видно запись, добавленную (но не зафиксированную) первой сессией. Так как уровень изоляции `Repeatable Read` 
строже `read committed`, она так же предотвращает аномалию `Read uncommited`

При фиксировании изменений в первой транзакции:
```
1| air_db=*# COMMIT;
COMMIT
```
Вторая сессия по прежнему не видит изменения, так как уровень изоляции `Repeatable Read` обеспечивает то, что в рамках одной 
транзикции будут видны одни и те же данные, т.е. исключается аномалия "неповторяющееся чтение". После завершения транзакции 
во второй сессии, изменения, внесенные в первой сессии, будут видны, так как начата новая транзакция:
```
2| air_db=*# table shipments;
 id | product_name | quantity | destination 
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia
(3 строки)

2| air_db=*# COMMIT;
COMMIT
2| air_db=# table shipments;
 id | product_name | quantity | destination 
----+--------------+----------+-------------
  1 | bananas      |     1000 | Europe
  2 | coffee       |      500 | USA
  3 | sugar        |      300 | Asia
  5 | bananas      |     2000 | Africa
(4 строки)
```
