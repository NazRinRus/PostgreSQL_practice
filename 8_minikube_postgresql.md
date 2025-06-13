## Практическое задание: развернуть PostgreSQL в minikube
### Задачи
#### Задача №1: Развернуть PostgreSQL через манифест
- Создайте Deployment и Service для PostgreSQL
- Указать имя пользователя и пароль через переменные окружения в Deployment (например, POSTGRES_USER и POSTGRES_PASSWORD)
- Убедиться, что база данных поднимается и отвечает на подключения (kubectl port-forward + psql).
#### Задача №2: Развернуть PostgreSQL через Helm
- Установить Helm
- Найти и установить подходящий Helm-чарт PostgreSQL 14 (например, Bitnami PostgreSQL)
- Указать параметры подключения в values.yaml
- Обеспечить масштабируемость: задать replicaCount: 3 или использовать StatefulSet.
### Предварительная подготовка хоста
- minikube сам разворачивается в контейнере, поэтому должен быть предустановлен Docker
- установить терминал kubectl
- установить minikube
- установить клиент psql, для удаленного подключения
#### Установка Docker
1. `sudo apt update`
2. `sudo apt install apt-transport-https ca-certificates curl software-properties-common`
3. `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`
4. `sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"`
5. `sudo apt update`
6. `apt-cache policy docker-ce`
7. `sudo apt install docker-ce`
#### Установка пакетов kubectl
```
curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```
#### Установка пакетов minikube
```
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```
#### Установка клиента psql
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

PGVERSION=17
sudo apt install postgresql-client-${PGVERSION?}
```
#### Скачать файлы тестового приложения на Python FastApi
```
scp -r nazrinrus@192.168.0.119:/home/nazrinrus/test_app /home/nazrinrus
```
#### Запуск кластера miniKube
```
minikube start --driver=docker
```
#### Создание директории `manifests` для .yaml-файлов
```
mkdir /home/nazrinrus/manifests && cd /home/nazrinrus/manifests
```
### Решение задачи №1
#### Создание Namespace с именем `postgres-ns`
Как и любой объект, создается командой `kubectl create namespace postgres-ns` или описанием в манифесте `vim namespace.yaml`, содержание:
```
apiVersion: v1
kind: Namespace
metadata:
  name: postgres-ns
  labels:
    name: postgres
```
после чего применяется созданный манифест, командой: `kubectl apply -f namespace.yaml`
```
# Проверяем
nazrinrus@ubuntuonlyforclone:~/manifests$ kubectl get namespace
NAME              STATUS   AGE
default           Active   7m20s
kube-node-lease   Active   7m20s
kube-public       Active   7m20s
kube-system       Active   7m20s
postgres-ns       Active   60s
```
Переключаемся на новый namespace
```
kubectl config set-context --current --namespace=postgres-ns
```
#### Создание манифеста для Deployment
`vim postgres-deployment.yaml`:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: postgres-ns
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:17
          env:
            - name: POSTGRES_USER
              value: "postgres"
            - name: POSTGRES_PASSWORD
              value: "postgres"
            - name: POSTGRES_DB
              value: "test_db"
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: postgres-storage
          emptyDir: {}
```
#### Создание манифеста для Service
`vim postgres-service.yaml`:
```
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: postgres-ns
  labels:
    app: postgres
spec:
  type: ClusterIP
  ports:
    - port: 5432
      targetPort: 5432
  selector:
    app: postgres
```
#### Применение манифестов Deployment и Service
```
kubectl apply -f postgres-deployment.yaml
kubectl apply -f postgres-service.yaml
```
#### Проверка подключения к PostgreSQL
Настройте port-forwarding для доступа к PostgreSQL:
```
kubectl port-forward svc/postgres 5432:5432 -n postgres-ns
```
Подключение к PostgreSQL через psql:
```
psql -h localhost -U postgres -d test_db -c "SELECT datname FROM pg_database;"
```
### Решение задачи №2
#### Скачать и установить последнюю версию Helm:
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
#### Добавить репозиторий Bitnami
```
helm repo add bitnami  https://mirror.yandex.ru/helm/charts.bitnami.com/
helm repo update
```
#### Создание файла values.yaml для настройки PostgreSQL
```
global:
  postgresql:
    auth:
      username: nazrinrus
      password: nazrinrus
      database: test_db
      postgresPassword: postgres
    primary:
      persistence:
        enabled: false
    replica:
      replicaCount: 3
      persistence:
        enabled: false
```
#### Установка Helm-чарта PostgreSQL версии 14
Необходимо найти доступные версии postgresql
```
helm search repo bitnami/postgresql --versions
```
14.3.3 максимальный релиз postgresql 14
```
helm install postgresql14 bitnami/postgresql -n pgsql --create-namespace --set metrics.enabled=true --version 14.3.3 -f values.yaml

nazrinrus@ubuntuonlyforclone:~/manifests$ helm list -n pgsql
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
postgresql14    pgsql           1               2025-06-13 16:13:48.595147586 +0000 UTC deployed        postgresql-14.3.3       16.2.0     
nazrinrus@ubuntuonlyforclone:~/manifests$ kubectl get pods -n pgsql
NAME             READY   STATUS    RESTARTS   AGE
postgresql14-0   2/2     Running   0          2m16s

```
#### Проверка возможности подключения
Форвардинг портов:
```
nazrinrus@ubuntuonlyforclone:~/manifests$ kubectl port-forward svc/postgresql14 5432:5432 -n pgsql
Forwarding from 127.0.0.1:5432 -> 5432
Forwarding from [::1]:5432 -> 5432
Handling connection for 5432
Handling connection for 5432

```
Во второй консоли виртуальной машины с Kubernetes:
```
nazrinrus@ubuntuonlyforclone:~$ psql -h localhost -U postgres -d test_db -c "SELECT datname FROM pg_database;"
Пароль пользователя postgres: 
  datname  
-----------
 postgres
 template1
 template0
 test_db
(4 строки)
```
#### Масштабирование
Изменить количество реплик в values.yaml
```
replica:
  replicaCount: 5
```
Выполнить команду upgrade
```
helm upgrade postgresql14 bitnami/postgresql -n pgsql --create-namespace --set metrics.enabled=true --version 14.3.3 -f values.yaml
```
