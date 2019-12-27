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
