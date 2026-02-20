# Django Site — локальный запуск через Minikube

## 📦 Требования

- Docker
- Minikube
- kubectl

Проверка:

```bash
minikube version
kubectl version --client
docker --version
```

---

## 🚀 Запуск Minikube

```bash
minikube start
```

Проверить:

```bash
kubectl get nodes
```

---

## 🐳 Сборка Docker-образа внутри Minikube

Чтобы образ был доступен кластеру:

```bash
eval $(minikube docker-env)
docker build -t django-site:local .
```

Проверить:

```bash
docker images | grep django-site
```

---

## ☸ Деплой в Kubernetes

```bash
kubectl apply -f kubernetes/
```

Проверить статус:

```bash
kubectl get pods
kubectl get services
```

---

## 🔁 Применение миграций

```bash
kubectl exec -it deploy/django -- \
python /app/src/manage.py migrate
```

---

## 👤 Создание суперпользователя

```bash
kubectl exec -it deploy/django -- \
python /app/src/manage.py createsuperuser
```

---

## 🌐 Доступ к приложению

Через port-forward:

```bash
kubectl port-forward svc/main-nginx 8080:80
```

Открыть:

http://localhost:8080

Админка:

http://localhost:8080/admin

---

## 🧾 Перезапуск приложения

```bash
kubectl rollout restart deployment/django
kubectl rollout restart deployment/main-nginx
```

---

## 📊 Просмотр логов

Django:

```bash
kubectl logs deploy/django
```

Nginx:

```bash
kubectl logs deploy/main-nginx
```

---

## 🧹 Остановка и удаление кластера

```bash
minikube stop
minikube delete
```

---

## 🎯 Архитектура (локально)

Браузер → main-nginx → Django → PostgreSQL

PostgreSQL может быть запущен:
- либо как отдельный pod
- либо локально через docker-compose

---

Локальное окружение предназначено для разработки и тестирования.
Продакшн-развёртывание выполняется в кластере yc-sirius-dev.
