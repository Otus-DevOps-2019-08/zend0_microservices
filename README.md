# zend0_microservices
zend0 microservices repository

# Docker
## Docker Machine
[Install Docker Machine](https://docs.docker.com/machine/install-machine/)

Для работы с GCP
```shell script
export GOOGLE_PROJECT=<идентификатор проекта>
```
Создаём удаленный хост для управления им через docker-machine
```shell script
$ docker-machine create --driver google \
--google-machine-image https://www.googleapis.com/compute/v1/projects/ubuntu-os-cloud/global/images/family/ubuntu-1604-lts \
--google-machine-type n1-standard-1 \
--google-zone europe-west1-b \
docker-host
```
Проверяем
```shell script
docker-machine ls
```
и заходим в неё для дальнейшей работы
```shell script
eval $(docker-machine env <docker-host>)
```
Переключение на локальный докер
```shell script
eval $(docker-machine env --unset)
```
Подключение к машине по SSH
```shell script
docker-machine ssh <docker-host>
```
Посмотреть IP машины можно через
```shell script
docker-machine ip <docker-host>
```
Удаляем созданную машину
```shell script
docker-machine rm <docker-host>
```

## Docker Hub

аутентифакация
```shell script
docker login
```
добавляем тег
```shell script
docker tag reddit:latest <your-login>/otus-reddit:1.0
```
заливаем образ
```shell script
docker push <your-login>/otus-reddit:1.0
```

## Docker и микросервисы
Сборка образа(ов)
```shell script
docker build -t <your-dockerhub-login>/post:1.0 ./post-py
docker build -t <your-dockerhub-login>/comment:1.0 ./comment
docker build -t <your-dockerhub-login>/ui:1.0 ./ui
```
Создание отдельной сети для контейнеров
```shell script
docker network create reddit
```
Создание volume для постоянных данных
```shell script
docker volume create reddit_db
```
Запуск контейнера в ранее созданной сети, создание псевданима для контенера, и подключем volume 
```shell script
docker run -d --network=reddit --network-alias=post_db --network-alias=comment_db -v reddit_db:/data/db mongo:latest
```
## Docker и сети
Docker при инициализации контейнера может подключить к нему только 1 сеть.  
Дополнительные сети подключаются командой: 
```shell script
docker network connect <network> <container>
```
список сетей
```shell script
docker network ls
```
Вывод сгенерированных правил iptables
```shell script
sudo iptables -nL -t nat (флаг -v даст чуть больше инфы)
```
## docker-compose
Запуск контейнеров
```shell script
docker-compose up
```
Остановка и удаление контейнеров
```shell script
docker-compose down
```
По-умолчанию, запущенный docker-compose будет всем контейнерам,volume и сетям добавлять префикс с названием дирректории 
где лежит `docker-compose.yml`.  
Если такое поведени не устраивает, то можно указать префикс самостоятельно (`-p, --project-name NAME`):
```yaml
docker-compose -p 'test' up
```
или добавить в `.env` `COMPOSE_PROJECT_NAME=<progect_name>`  

Избавиться от префикса в именах контейнеров можно путем добавления `container_name: <name>` в описании сервиса
```shell script
  ui:
    build: ./ui
    image: ${USERNAME}/ui:${TAG}
    container_name: ui
```
# GitLab CI
## GitLab runner
Запуск раннера в контейнере
```shell script
docker run -d --name gitlab-runner --restart always \
-v /srv/gitlab-runner/config:/etc/gitlab-runner \
-v /var/run/docker.sock:/var/run/docker.sock \
gitlab/gitlab-runner:latest
```
Регистрация раннера
```shell script
docker exec -it gitlab-runner gitlab-runner register --run-untagged --locked=false
```
# Мониторинг
## Prometheus
### Формат метрик
```shell script
prometheus_build_info{branch="HEAD",goversion="go1.13.5",instance="localhost:9090",job="prometheus",revision="d9613e5c466c6e9de548c4dae1b9aabf9aaf7c57",version="2.15.2"}
```
* **prometheus_build_info**: название метрики - идентификатор собранной информации
* **branch**, **goversion**, и т.д.: лейбл - добавляет метаданных метрике, уточняет ее. 
Использование лейблов дает нам возможность неограничиваться лишь одним названием метрик для идентификации получаемой 
информации. Лейблы содержаться в {} скобках и представлены наборами "ключ=значение".
* значение метрики - численное значение метрики, либо `NaN`, если значение не доступно.

### Настройка целей для мониторинга
Описание сервисов для мониторнга описываем в файле `prometheus.yml`
```yaml
...
scrape_configs:
...
  - job_name: 'prometheus'
    static_configs:
      - targets:
          - 'localhost:9090'

  - job_name: 'ui'
    static_configs:
      - targets:
          - 'ui:9292'
...
```
### Настройка алертов
Описание алертов в файле `alerts.yml`
```yaml
groups:
  - name: alert.rules
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: page
        annotations:
          description: '{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute'
          summary: 'Instance {{ $labels.instance }} down'
```
за сами алерты отвечает доп. сервис - [Alertmanager](https://prometheus.io/docs/alerting/alertmanager/)

docker hub [с готовыми образами](https://hub.docker.com/u/bbilder)

## Мониторинг Docker контейнеров
Рассмотрен вариант сбора метрик с помощью cAdvisor.

# Логирование и распределенная трассировка
## Логирование
Для сбора логов используем EFK стек (обработка логов происходит через fluentd). Он позволяет
работать как со структурированными логами (подготовленные сообщение в JSON формате) так и 
неструктурированными (plain-text и тут regexp и grok наше все).

## Распределенный трейсинг
Zipkin - один из таких инструментов, добавляет метки в сообщения сервисов и позволяет отслеживать куда и как долго
шли запросы от сервиса к сервису.

# Kubernetes

## Kubernetes The Hard Way

[Установка Kubernetes](https://github.com/kelseyhightower/kubernetes-the-hard-way) от инженерома Google Kelsey Hightower

_В ходе выполнения данной работы, на триальном периоде (300$) GCP не давал запустить 6 VM, только 4 и всё, смена региона не помогает_

Список установленных компонентов
```shell script
kubectl get componentstatuses --kubeconfig <kubeconfig>
```
Попадаем в под
```shell script
kubectl exec -ti $POD_NAME -- nslookup kubernetes
kubectl exec -ti $POD_NAME -- nginx -v
```
Лог пода
```shell script
kubectl logs $POD_NAME
```

## Minikube

Запускаем кластер
```yaml
minikube start
```
## Kubectl

### Описание конфига
Файл `~/.kube/config` - это такой же манифест kubernetes в YAML-формате (есть и Kind, и ApiVersion).
```yaml
apiVersion: v1
clusters: # Список кластеров
- cluster:
    certificate-authority: /home/sema/.minikube/ca.crt
    server: https://192.168.39.42:8443
  name: minikube
contexts: # Список контекстов
- context:
    cluster: minikube
    user: minikube
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users: # Список пользователей
- name: minikube
  user:
    client-certificate: /home/sema/.minikube/client.crt
    client-key: /home/sema/.minikube/client.key
``` 

Кластер (cluster) - это:
1. **server** - адрес kubernetes API-сервера 
2. **certificate-authority** - корневой сертификат (которым подписан SSL-сертификат самого сервера), что бы убедиться, что нас не обманывают и перед нами тот самый сервер  
\+ **name** (Имя) для идентификации в конфиге.

Пользователь (**user**) - это:
1. Данные для аутентификации (зависит от того, как настроен сервер). Это могут быть: 
* username + password (Basic Auth)
* client key + client certificate
* token
* auth-provider config (напримерGCP)  
\+ **name** (Имя) дляидентификациивконфиге

Контекст (**контекст**) - это:
1. **cluster** - имя кластера из списка clusters
2. **user** - имя пользователя из списка users
3. **namespace** - область видимости по-умолчанию (необязательно)
\+ **name** (Имя) для идентификации в конфиге

### Пример конфигурирования kubectl
Обычно порядок конфигурирования kubectl следующий:
1. Создать cluster:
    ```shell script
    kubectl config set-cluster ... cluster_name
    ```
2. Создать данные пользователя (credentials)
    ```shell script
    kubectl config set-credentials ... user_name
    ```
3. Создать контекст
    ```
    kubectl config set-context context_name \
    --cluster=cluster_name \
    --user=user_name
    ```
4. Использовать контекст
    ```shell script
    kubectl config use-context context_name
    ```

Список всех контекстов можно увидеть так
```shell script
kubectl config get-contexts
```

`kubectl apply -f <filename>` может принимать не только отдельный файл, 
но и папку с ними. Например:
```shell script
kubectl apply -f ./kubernetes/reddit
```

kubectl умеет пробрасывать сетевые порты POD-ов на локальную машину.  
Найдем, используя selector, POD-ы приложения:
```shell script
kubectl get pods --selector component=ui
```
```shell script
kubectl port-forward <pod-name> 8080:9292 # local-port : pod-port
```

### Services

По label-ам найти соответствующие POD-ы. Посмотреть можно с помощью:
```shell script
kubectl describe service comment | grep Endpoints
```
изнутри любого POD-а разрешаем имя сервиса:
```shell script
kubectl exec -ti <pod-name> nslookup comment
```
Удалеие сервиса
```shell script
kubectl delete -f mongodb-service.yml
# или
kubectl delete service mongodb
```
По-умолчанию все сервисы имеют тип **ClusterIP** - это значит, что сервис распологается 
на внутреннем диапазоне IP-адресов кластера. Снаружи до него нет доступа.

Тип **NodePort** - на каждой ноде кластера открывает порт из диапазона **30000-32767** и 
переправляет трафик с этого порта на тот, который указан в **targetPort** Pod (похоже на стандартный expose в docker)

#### Services и minikube

Minikube может выдавать web-странцы с сервисами которые были помечены типом NodePort
```shell script
minikube service ui
```
Minikube может перенаправлять на web-странцы с сервисами которые были помечены типом 
NodePort
```shell script
minikube service list
```
Minikube также имеет в комплекте несколько стандартных аддонов, получить список можно так
```shell script
minikube addons list
```
Открываем работающий Dashboard
```shell script
minikube dashboard
```

## Ingress

подготовим сертификат используя IP как CN

```shell script
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/CN= 35.190.66.90"
```
загрузим сертификат в кластер kubernetes

```shell script
kubectl create secret tls ui-ingress --key tls.key --cert tls.crt -n dev
```

## Network Policy

Определяем имя кластера

```shell script
gcloud beta container clusters  list
```

Включим network-policy для GKE.

```shell script
gcloud beta container clusters update <cluster-name> --zone=us-central1-a --update-addons=NetworkPolicy=ENABLED
gcloud beta container clusters update <cluster-name> --zone=us-central1-a  --enable-network-policy
```

## Volume

Вместо того, чтобы хранить данные локально на ноде, имеет смысл подключить удаленное хранилище. В нашем случае можем использовать Volume gcePersistentDisk, который будет складывать данные в хранилище GCE.  

Создаём диск
```shell script
gcloud compute disks create --size=25GB --zone=europe-west1-c reddit-mongo-disk
```

## PersistentVolume

Используемый механизм Volume-ов можно сделать удобнее. Мы можем использовать не целый выделенный диск для каждого пода, а целый ресурс хранилища, общий для всего кластера.  
Тогда при запуске Stateful-задач в кластере, мы сможем запросить хранилище в виде такого же ресурса, как CPU или оперативная память.

### PersistentVolumeClaim

Мы ранее создали ресурс дискового хранилища, распространенный на весь кластер, в виде PersistentVolume.  
Чтобы выделить приложению часть такого ресурса - нужно создать запрос на выдачу - PersistentVolumeClaim. 
Claim - это именно запрос, а не само хранилище.  
С помощью запроса можно выделить место как из конкретного PersistentVolume (тогда параметры accessModes и StorageClass должны соответствовать, а места должно хватать), так и просто создать отдельный PersistentVolume под конкретный запрос.

## Динамическое выделение Volume'ов

Создав PersistentVolume мы отделили объект "хранилища" от наших Service'ов и Pod'ов. Теперь мы можем его при необходимости переиспользовать.  
Но нам гораздо интереснее создавать хранилища при необходимости и в автоматическом режиме. В этом нам помогут StorageClass’ы. Они описывают где (какой провайдер) и какие хранилища создаются.

### StorageClass

Вывод какие получились PersistentVolume'ы

```shell script
kubectl get persistentvolume -n dev
```
