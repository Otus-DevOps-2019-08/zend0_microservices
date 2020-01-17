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
