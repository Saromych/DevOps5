# Observability в Kubernetes — Вариант 1

## Описание проекта

CRUD приложение на Go с PostgreSQL. Добавлен эндпоинт `/metrics` для Prometheus, развернуты Prometheus и Grafana в Kubernetes (Kind) для мониторинга.

## Требования задания

- [x] Приложение отдаёт метрики на `/metrics`
- [x] Prometheus развернут через raw-манифесты
- [x] Grafana развернута через raw-манифесты
- [x] Prometheus подключен как datasource в Grafana
- [x] Дашборд с 3+ панелями (RPS, латентность, ошибки)
- [x] Экспортирован JSON дашборда в репозиторий

## 1. Установка зависимостей

```bash
# Docker (если нет)
curl -fsSL https://get.docker.com | sh

# kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
```

## 2. Создать кластер

```bash
kind create cluster --name crud-cluster --config k8s/kind-config.yaml
```
## 3. Собрать и загрузить образ приложения

```bash
cd app
go mod tidy
docker build -t crud-app:latest .
kind load docker-image crud-app:latest --name crud-cluster
cd ..
```

## 4. Задеплоить приложение + PostgreSQL

```bash
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/app.yaml

kubectl rollout status deployment/postgres -n crud-demo
kubectl rollout status deployment/crud-app -n crud-demo
```

## 5. Задеплоить Prometheus и Grafana

```bash
kubectl apply -f k8s/monitoring/prometheus-config.yaml
kubectl apply -f k8s/monitoring/prometheus.yaml
kubectl apply -f k8s/monitoring/grafana.yaml

# Проверить что поды запустились
kubectl get pods -n monitoring -w
```

## 6. Пробросить порты для доступа

```bash
# Prometheus UI
kubectl port-forward -n monitoring svc/prometheus 9090:9090 &

# Grafana UI
kubectl port-forward -n monitoring svc/grafana 3000:3000 &

# Приложение (для генерации нагрузки)
kubectl port-forward -n crud-demo svc/crud-app 8080:8080 &
```
