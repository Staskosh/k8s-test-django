# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Запуск в кластере minikube

Установить:

[Kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)
[minikube](https://minikube.sigs.k8s.io/docs/)
[VirtualBox](https://www.virtualbox.org) - для Linux\Windows\MacOS на процессорах Интел.
[DockerDesktop](https://www.docker.com) - MacOS для Mac на M1.

Поднимите Postgres:

Установите [helm](https://helm.sh/docs/intro/install/). 
Выполните следующие команды для добавления необходимого чарта и его установки:
```shell-session
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm install <NAME> bitnami/postgresql
```

Подключитесь к поду Potgresql:
```shell-session
kubectl exec --stdin --tty <НАЗВАНИЕ ПОДА>> -- sh
```

Создайте базу данных в поде Postgresql:
```
CREATE DATABASE {DB_NAME};
CREATE USER {DB_USER} WITH ENCRYPTED PASSWORD {PASSWORD};
GRANT ALL PRIVILEGES ON DATABASE {DB_NAME} TO {DB_USER};
```

В папке kubernetes создайте файл configmap.yaml, заполнив как в примере ниже:
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-app
data:
  SECRET_KEY: 'Ваш секрет тут'
  DEBUG: 'False'
  DATABASE_URL: 'postgres://DB_USER:PASSWORD@<NAME>-postgresql:5432/DB_NAME'
  ALLOWED_HOSTS: '127.0.0.1,minkube.ip'
```

Запустить манифесты:
`kubectl -n django-app apply -f kubernetes/ `

Проверить запущенные поды:
`kubectl get pods`

 Подключитесь к поду `django-app` и создайте суперпользователя Django:
 ```shell-session
kubectl exec --stdin --tty <НАЗВАНИЕ ПОДА>> -- sh
```

```python
./manage.py createsuperuser
```