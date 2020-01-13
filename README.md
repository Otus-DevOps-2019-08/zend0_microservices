# zend0_microservices
zend0 microservices repository

# Docker
## Docker Machine
[Install Docker Machine](https://docs.docker.com/machine/install-machine/)

Для работы с GCP
```shell script
export GOOGLE_PROJECT=<идентификатор проекта>
```
Создаём удаленных хост для управления им через docker-machine
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
eval $(docker-machine env docker-host)
```
Поодключение к машине по SSH
```shell script
docker-machine ssh docker-host
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
