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

## Запуск в кластере Kubernetes используя Minikube

- Установите [kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/).
- Установите и запустите [minikube](https://minikube.sigs.k8s.io/docs/):
- Создайте образ Django-приложения в кластере с помощью команды:

```bash
minikube image build -t django_app backend_main_django/
```

- Заполните переменные окружения разделе `data` файла конфигурации `kubernetes/django-app-config.yml`:
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-app-config
data:
  ALLOWED_HOSTS: 127.0.0.1, localhost, 123.456.78.9
  DATABASE_URL: postgres://username:password@host:5432/dbname
  DEBUG: 'False'
  SECRET_KEY: secret-key
```

- Создайте `ConfigMap` из этого файла с помощью команды:
  
```bash
kubectl apply --filename kubernetes/django-app-config.yml
```

- Запустите деплой:

```bash
kubectl apply -f kubernetes/django-app-deployment.yml
```

## Настройка Ingress
Установите расширение для `minikube`:

```bash
minikube addons enable ingress
```

Запустите манифест:

```bash
kubectl apply -f kubernetes/ingress.yml
```

Добавьте в файл /etc/host строчку:

```bash
11.22.33.44 star-burger.test
```
Где 11.22.33.44 - ip-адрес кластера.

## Настройка удаления сессий

Задать регулярность удаления пользовательских сессий можно в файле `django-app-clearsessions-cronjob.yml`:

```yaml
schedule: "0 0 1 * *"
```

После настройки времени необходимо запустить команду в работу:

```bash
kubectl apply --filename kubernetes/django-app-clear-sessions-cronjob.yml
```

Для проверки:

```bash
kubectl create job --from=cronjob/django-app-clearsessions-cronjob test-clearsession-job
```