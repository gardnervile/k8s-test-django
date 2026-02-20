# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Kubernetes secrets (local)

Create `kubernetes/secret.yaml` (do not commit it):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secrets
type: Opaque
stringData:
  SECRET_KEY: "empty"
  DATABASE_URL: "postgres://postgres:postgres@host.docker.internal:5432/webapp"
  ```
Apply it to the cluster:
```
kubectl apply -f kubernetes/secret.yaml
kubectl apply -f kubernetes/
```
## 🚀 Запуск проекта в Minikube (с Ingress)

### 1️⃣ Запуск Minikube

```bash
minikube start --driver=docker
minikube addons enable ingress
```

---

### 2️⃣ Создание Secret (НЕ коммитить в репозиторий)

```bash
kubectl create secret generic django-secrets \
  --from-literal=SECRET_KEY='ваш-секретный-ключ' \
  --from-literal=DATABASE_URL='postgres://postgres:postgres@<IP_ХОСТА>:5432/webapp'
```

Замените `<IP_ХОСТА>` на IP-адрес вашей машины (например: `192.168.1.34`).

---

### 3️⃣ Применение манифестов Kubernetes

```bash
kubectl apply -f kubernetes/
```

Дождитесь запуска Deployment:

```bash
kubectl rollout status deployment/django
```

---

### 4️⃣ Привязка локального домена

Узнайте IP Minikube:

```bash
minikube ip
```

Добавьте его в файл `/etc/hosts`:

```bash
sudo sh -c 'echo "<MINIKUBE_IP> star-burger.test" >> /etc/hosts'
```

Пример:

```bash
sudo sh -c 'echo "192.168.49.2 star-burger.test" >> /etc/hosts'
```

---

### 5️⃣ Открыть сайт в браузере

```
http://star-burger.test
```

Админка:

```
http://star-burger.test/admin/
```

---

### 🔄 Пересборка образа после изменений кода

```bash
docker build -t django_unit_app:latest -f backend_main_django/Dockerfile.unit.k8s backend_main_django
minikube image load django_unit_app:latest
kubectl rollout restart deployment/django
```
## 🧹 Автоматическая очистка сессий

CronJob запускается ежедневно в 03:00.

Проверка:

```bash
kubectl get cronjobs
```

Принудительный запуск:

```bash
kubectl create job --from=cronjob/django-clearsessions django-clearsessions-once
```

## YC-Sirius dev: подготовка SSL-сертификата для PostgreSQL (root.crt)

NS=edu-evgenij-kondratev

 1) Создать secret pg-root-crt из уже выданного секрета postgres
 ```
kubectl -n "$NS" get secret postgres -o jsonpath='{.data.root\.crt}' | base64 -d > root.crt
kubectl -n "$NS" delete secret pg-root-crt --ignore-not-found
kubectl -n "$NS" create secret generic pg-root-crt --from-file=root.crt=root.crt
rm -f root.crt
```

 2) Применить pod psql-test (манифест монтирует root.crt в /root/.postgresql/root.crt)
 ```
kubectl -n "$NS" delete pod psql-test --ignore-not-found
kubectl apply -f deploy/yc-sirius-dev/psql-test.yaml
kubectl -n "$NS" wait --for=condition=Ready pod/psql-test --timeout=120s
```

 3) Проверить подключение (SSL verify-full)
 ```
kubectl -n "$NS" exec -it psql-test -- sh -lc 'psql "sslmode=verify-full" -c "\conninfo"'
kubectl -n "$NS" exec -it psql-test -- sh -lc 'psql "sslmode=verify-full" -c "\dt"'
```
# Как собрать и опубликовать Docker-образ

 1. Перейти в директорию backend_main_django
```
cd backend_main_django
```
 2. Получить хэш текущего коммита
```
TAG=$(git rev-parse --short HEAD)
```
 3. Собрать образ
```
docker buildx build \
  --platform=linux/amd64 \
  -t gardnervile/django-site:$TAG \
  -f Dockerfile.unit.k8s \
  --load .
```
 4. Опубликовать образ
```
docker push gardnervile/django-site:$TAG
```
# Django Site — деплой в yc-sirius-dev

## Архитектура

В dev-окружении используется следующая схема обработки HTTP-запросов:

Браузер → Application Load Balancer → main-nginx → Django

- ALB принимает HTTPS-трафик по доменному имени
- main-nginx работает как reverse proxy
- Django запущен в отдельном Deployment
- PostgreSQL — Managed Service в Яндекс Облаке (вне Kubernetes)

---

## Необходимые переменные окружения

Django использует следующие переменные:

- SECRET_KEY
- DATABASE_URL
- DEBUG
- ALLOWED_HOSTS

Переменные передаются через Kubernetes Secret `django-secrets`.

---

## Сборка и публикация Docker-образа

```bash
docker build -t gardnervile/django-site:<commit_hash> .
docker push gardnervile/django-site:<commit_hash>
```

---

## Обновление версии в кластере

```bash
kubectl -n edu-evgenij-kondratev set image deployment/django \
django=gardnervile/django-site:<commit_hash>

kubectl -n edu-evgenij-kondratev rollout status deployment/django
```

---

## Применение миграций

```bash
kubectl -n edu-evgenij-kondratev exec -it deploy/django -- \
python /app/src/manage.py migrate
```

---

## Создание суперпользователя

```bash
kubectl -n edu-evgenij-kondratev exec -it deploy/django -- \
python /app/src/manage.py createsuperuser
```

---

## Проверка доступности

Через port-forward:

```bash
kubectl -n edu-evgenij-kondratev port-forward svc/main-nginx 8080:80
```

Открыть в браузере:

http://localhost:8080

Через домен (через ALB):

https://edu-evgenij-kondratev.yc-sirius-dev.pelid.team

---

## Где смотреть ошибки

Логи Django:

```bash
kubectl -n edu-evgenij-kondratev logs deploy/django
```

Логи Nginx:

```bash
kubectl -n edu-evgenij-kondratev logs deploy/main-nginx
```

---

## Что требуется для работы приложения

- Kubernetes Cluster (yc-sirius-dev)
- Managed PostgreSQL
- Docker Registry (Docker Hub)
- Application Load Balancer
- TLS-сертификат

---

## Как проверить успешность деплоя

- Pod `django` находится в статусе Running
- Pod `main-nginx` находится в статусе Running
- `kubectl rollout status` завершён успешно
- Сайт доступен через браузер
- Админка доступна по `/admin`