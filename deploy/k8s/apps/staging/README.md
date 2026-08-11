# Оставшиеся микросервисы в staging

Этот runbook добавляет `catalog-service` и `booking-service` после уже работающего `auth-service`.

Порядок зависимостей:

```text
Yandex Lockbox -> External Secrets Operator -> Kubernetes Secret

catalog-service -> Managed PostgreSQL / catalog_staging
catalog-service -> RabbitMQ / platform namespace

booking-service -> Managed PostgreSQL / booking_staging
booking-service -> catalog-service.staging.svc.cluster.local:8082
```

Оба сервиса остаются внутренними `ClusterIP`. Публичные IP, LoadBalancer и Ingress на этом шаге не создаются.

## Зафиксированные параметры

- PostgreSQL: `c-c9q3gaf98a21tn8dvb68.rw.mdb.yandexcloud.net:6432` с TLS;
- Catalog DB и пользователь: `catalog_staging`;
- Catalog PostgreSQL Lockbox secret: `e6qjjkr57d9v13o74qse`, property `postgresql_password`;
- Booking DB и пользователь: `booking_staging`;
- Booking PostgreSQL Lockbox secret: `e6qfa6qd7k7567srils9`, property `postgresql_password`;
- общий staging JWT secret: `e6q1defjt7a1d78bg8kd`, property `jwt_secret`;
- RabbitMQ secret: `e6qtimi50pl3hc33d1v4`, properties `username` и `password`;
- Catalog image: `cr.yandex/crpfrtdcq05e9r2l43ro/catalog-service@sha256:cac707c2b6fdd4a182c2e85c29f8e7cffb76a741d155e756fb00373785b6770a`;
- Booking image: `cr.yandex/crpfrtdcq05e9r2l43ro/booking-service@sha256:f340d3d0d960ca030ec1496792a6658672b78200ecf58151f3bbc46e9f5a87b9`.

Значения секретов в репозитории отсутствуют.

## 1. Подготовить Yandex Cloud через UI

На каждом из двух PostgreSQL-секретов выдайте сервисному аккаунту `lab-external-secrets` роль `lockbox.payloadViewer`:

```text
e6qjjkr57d9v13o74qse
e6qfa6qd7k7567srils9
```

Роль выдаётся на конкретный секрет, а не на весь каталог. Существующие права на JWT и RabbitMQ повторно выдавать не нужно.

В Managed PostgreSQL откройте базы `catalog_staging` и `booking_staging`. Если `pgcrypto` ещё не включено, добавьте расширение через настройки каждой базы и дождитесь состояния кластера `Running`. Не устанавливайте расширение SQL-командой.

В Lockbox проверьте только имя свойства пароля: `postgresql_password`. Сам пароль не копируйте и не выводите.

## 2. Проверить манифесты на API-сервере

Команда ничего не создаёт:

```bash
kubectl apply --dry-run=server \
  -f deploy/k8s/apps/catalog-service/staging/00-service-account.yaml \
  -f deploy/k8s/apps/catalog-service/staging/10-external-secret.yaml \
  -f deploy/k8s/apps/catalog-service/staging/20-configmap.yaml \
  -f deploy/k8s/apps/catalog-service/staging/30-network-policies.yaml \
  -f deploy/k8s/apps/catalog-service/staging/40-migration-job.yaml \
  -f deploy/k8s/apps/catalog-service/staging/50-deployment.yaml \
  -f deploy/k8s/apps/catalog-service/staging/60-service.yaml \
  -f deploy/k8s/apps/booking-service/staging/00-service-account.yaml \
  -f deploy/k8s/apps/booking-service/staging/10-external-secret.yaml \
  -f deploy/k8s/apps/booking-service/staging/20-configmap.yaml \
  -f deploy/k8s/apps/booking-service/staging/30-network-policies.yaml \
  -f deploy/k8s/apps/booking-service/staging/40-migration-job.yaml \
  -f deploy/k8s/apps/booking-service/staging/50-deployment.yaml \
  -f deploy/k8s/apps/booking-service/staging/60-service.yaml
```

## 3. Создать настройки и синхронизировать секреты

```bash
kubectl apply \
  -f deploy/k8s/apps/catalog-service/staging/00-service-account.yaml \
  -f deploy/k8s/apps/catalog-service/staging/10-external-secret.yaml \
  -f deploy/k8s/apps/catalog-service/staging/20-configmap.yaml \
  -f deploy/k8s/apps/catalog-service/staging/30-network-policies.yaml \
  -f deploy/k8s/apps/booking-service/staging/00-service-account.yaml \
  -f deploy/k8s/apps/booking-service/staging/10-external-secret.yaml \
  -f deploy/k8s/apps/booking-service/staging/20-configmap.yaml \
  -f deploy/k8s/apps/booking-service/staging/30-network-policies.yaml
```

Дождитесь обоих секретов:

```bash
kubectl wait \
  --for=condition=Ready \
  externalsecret/catalog-service \
  -n staging \
  --timeout=2m

kubectl wait \
  --for=condition=Ready \
  externalsecret/booking-service \
  -n staging \
  --timeout=2m
```

Проверяйте только статус и количество ключей, не содержимое:

```bash
kubectl get externalsecret catalog-service -n staging
kubectl get secret catalog-service -n staging
kubectl get externalsecret booking-service -n staging
kubectl get secret booking-service -n staging
```

Catalog Secret должен содержать три ключа, Booking Secret — два.

## 4. Последовательно выполнить миграции

Сначала Catalog:

```bash
kubectl delete job catalog-service-migrations \
  -n staging \
  --ignore-not-found

kubectl apply \
  -f deploy/k8s/apps/catalog-service/staging/40-migration-job.yaml

kubectl wait \
  --for=condition=Complete \
  job/catalog-service-migrations \
  -n staging \
  --timeout=10m

kubectl logs job/catalog-service-migrations -n staging
```

Только после `Complete 1/1` выполните Booking:

```bash
kubectl delete job booking-service-migrations \
  -n staging \
  --ignore-not-found

kubectl apply \
  -f deploy/k8s/apps/booking-service/staging/40-migration-job.yaml

kubectl wait \
  --for=condition=Complete \
  job/booking-service-migrations \
  -n staging \
  --timeout=10m

kubectl logs job/booking-service-migrations -n staging
```

Если любой Job завершится с ошибкой, не повторяйте его и не запускайте `force` самостоятельно. Сначала сохраните журнал и состояние Job:

```bash
kubectl get job catalog-service-migrations -n staging -o wide
kubectl logs -n staging -l job-name=catalog-service-migrations --prefix=true --tail=200

kubectl get job booking-service-migrations -n staging -o wide
kubectl logs -n staging -l job-name=booking-service-migrations --prefix=true --tail=200
```

## 5. Развернуть Catalog, затем Booking

```bash
kubectl apply \
  -f deploy/k8s/apps/catalog-service/staging/50-deployment.yaml \
  -f deploy/k8s/apps/catalog-service/staging/60-service.yaml

kubectl rollout status \
  deployment/catalog-service \
  -n staging \
  --timeout=5m
```

После успешного Catalog:

```bash
kubectl apply \
  -f deploy/k8s/apps/booking-service/staging/50-deployment.yaml \
  -f deploy/k8s/apps/booking-service/staging/60-service.yaml

kubectl rollout status \
  deployment/booking-service \
  -n staging \
  --timeout=5m
```

## 6. Проверить Kubernetes-состояние

```bash
kubectl get deployment catalog-service -n staging
kubectl get deployment booking-service -n staging

kubectl get pods -n staging -l app.kubernetes.io/component=api -o wide

kubectl get service catalog-service -n staging -o wide
kubectl get service booking-service -n staging -o wide

kubectl get endpointslice -n staging -l kubernetes.io/service-name=catalog-service
kubectl get endpointslice -n staging -l kubernetes.io/service-name=booking-service
```

Ожидается:

- три API Deployment в staging имеют `READY 1/1` с учётом уже работающего Auth;
- оба новых Service имеют тип `ClusterIP` и не имеют `EXTERNAL-IP`;
- каждый EndpointSlice содержит один адрес Pod;
- Booking Pod имеет метку `networking.restaurant-booking/access-catalog-service=true`;
- NetworkPolicy разрешает Booking обращаться к Catalog на `8082`, но не открывает Catalog всем Pod namespace.

## Что будет дальше

После трёх работающих сервисов в staging следующий production-like шаг — не копирование в production. Сначала добавляются наблюдаемость и управляемый входящий трафик: application metrics, Managed Prometheus, Grafana и Application Load Balancer. Затем на уже наблюдаемом приложении безопасно проверяются две реплики, PDB и HPA.
