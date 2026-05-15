# CRUD в Kubernetes (локально)

## Стек
- **kind** — локальный K8s в Docker
- **Go** — REST API (CRUD над таблицей users)
- **PostgreSQL 16** — база данных с PVC

## Структура
```
.
├── app/
│   ├── main.go       # Go REST API
│   ├── go.mod
│   └── Dockerfile
├── k8s/
│   ├── kind-config.yaml   # конфиг кластера
│   ├── namespace.yaml
│   ├── postgres.yaml      # Secret + PVC + Deployment + Service
│   └── app.yaml           # ConfigMap + Deployment + Service
└── helm/
    └── crud-app/
        ├── Chart.yaml
        ├── values.yaml    # все параметры в одном месте
        └── templates/     # те же манифесты, но шаблонизированные
```

Развернуть можно двумя способами: через **kubectl** (раздел 4) или через **Helm** (раздел 4а).

## 1. Установка зависимостей

```bash
# Docker (если нет)
curl -fsSL https://get.docker.com | sh

# kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.23.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

## 2. Создать кластер

```bash
kind create cluster --name demo --config k8s/kind-config.yaml
```

## 3. Собрать и загрузить образ

```bash
cd app
go mod tidy
docker build -t crud-app:latest .
kind load docker-image crud-app:latest --name demo
cd ..
```

## 4. Задеплоить через kubectl

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/app.yaml

# Подождать готовности
kubectl rollout status deployment/postgres -n crud-demo
kubectl rollout status deployment/crud-app -n crud-demo
```

## 4а. Задеплоить через Helm

### Установка Helm (если нет)

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

### Установка чарта

```bash
helm install crud-app ./helm/crud-app

# Подождать готовности
kubectl rollout status deployment/postgres -n crud-demo
kubectl rollout status deployment/crud-app -n crud-demo
```

### Переопределить параметры

```bash
# Через флаг --set (разово)
helm install crud-app ./helm/crud-app --set app.replicas=3

# Через отдельный values-файл (для prod/staging)
helm install crud-app ./helm/crud-app -f values-prod.yaml
```

### Обновить релиз после изменений

```bash
helm upgrade crud-app ./helm/crud-app
```

### Откатить к предыдущей версии

```bash
helm rollback crud-app
```

### Удалить релиз

```bash
helm uninstall crud-app
```

### Полезные команды Helm

```bash
# Посмотреть статус релиза
helm status crud-app

# История версий
helm history crud-app

# Проверить шаблоны без деплоя
helm template crud-app ./helm/crud-app

# Проверить chart на ошибки
helm lint ./helm/crud-app
```

## 5. Тестировать CRUD

```bash
BASE=http://localhost:8080

# CREATE
curl -s -X POST $BASE/users \
  -H 'Content-Type: application/json' \
  -d '{"name":"Alice","email":"alice@example.com"}' | jq

# READ ALL
curl -s $BASE/users | jq

# READ ONE
curl -s $BASE/users/1 | jq

# UPDATE
curl -s -X PUT $BASE/users/1 \
  -H 'Content-Type: application/json' \
  -d '{"name":"Alice Smith","email":"alice@example.com"}' | jq

# DELETE
curl -s -X DELETE $BASE/users/1 -w "%{http_code}\n"
```

## 6. Полезные команды

```bash
# Посмотреть поды
kubectl get pods -n crud-demo

# Логи приложения
kubectl logs -l app=crud-app -n crud-demo -f

# Зайти в PostgreSQL
kubectl exec -it deployment/postgres -n crud-demo -- psql -U postgres -d cruddb

# Масштабировать
kubectl scale deployment/crud-app --replicas=3 -n crud-demo

# Удалить кластер
kind delete cluster --name demo
```
