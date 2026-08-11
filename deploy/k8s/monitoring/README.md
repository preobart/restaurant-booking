# Метрики Kubernetes и staging-приложений

Prometheus Operator установлен через Yandex Cloud Marketplace в namespace `monitoring` и пишет метрики в Managed Prometheus workspace:

```text
mond7en8jco98a3qtrg6
```

Этот каталог добавляет сбор `/metrics` у трёх внутренних сервисов:

```text
auth-service.staging.svc.cluster.local:8081/metrics
catalog-service.staging.svc.cluster.local:8082/metrics
booking-service.staging.svc.cluster.local:8083/metrics
```

Service не публикуются наружу. Prometheus получает доступ к их HTTP-портам через отдельную NetworkPolicy.

## 1. Проверить установку Marketplace

```bash
kubectl get pods -n monitoring -o wide
kubectl get crd servicemonitors.monitoring.coreos.com
kubectl get pods \
  -n monitoring \
  -l app.kubernetes.io/name=prometheus \
  --show-labels
```

Ожидается:

- компоненты monitoring имеют состояние `Running`;
- CRD `servicemonitors.monitoring.coreos.com` существует;
- команда с селектором `app.kubernetes.io/name=prometheus` находит Prometheus Pod.

Если последняя команда не находит Pod, NetworkPolicy пока не применяйте: сначала уточните реальные метки Prometheus.

## 2. Проверить манифесты на API-сервере

```bash
kubectl apply --dry-run=server \
  -f deploy/k8s/monitoring/00-prometheus-scrape-network-policy.yaml \
  -f deploy/k8s/monitoring/10-staging-service-monitor.yaml
```

Dry-run должен распознать `ServiceMonitor`. Ошибка `no matches for kind ServiceMonitor` означает, что установка Operator ещё не завершилась или его CRD отсутствует.

## 3. Разрешить scrape и создать ServiceMonitor

```bash
kubectl apply \
  -f deploy/k8s/monitoring/00-prometheus-scrape-network-policy.yaml \
  -f deploy/k8s/monitoring/10-staging-service-monitor.yaml
```

Проверьте созданные объекты:

```bash
kubectl get networkpolicy allow-prometheus-scrape -n staging
kubectl get servicemonitor restaurant-booking-staging -n monitoring
```

ServiceMonitor выбирает все Service с тремя метками:

```text
app.kubernetes.io/part-of=restaurant-booking
app.kubernetes.io/component=api
environment=staging
```

Все три Service используют имя порта `http`, поэтому одного ServiceMonitor достаточно.

## 4. Проверить метрики в Yandex Monitoring

Подождите 1–2 минуты после применения.

Откройте:

```text
Monitoring -> Метрики
Источник данных: Prometheus
Workspace: mond7en8jco98a3qtrg6
```

Выполните:

```promql
up{namespace="staging"}
```

Ожидаются три временных ряда со значением `1`. Если приложение экспортирует дополнительные метрики, их имена можно найти запросом:

```promql
count by (__name__) ({namespace="staging"})
```

Для проверки Kubernetes-метрик:

```promql
count(kube_pod_info{namespace="staging"})
```

## 5. Открыть Grafana только локально

Сначала найдите Service и Secret Grafana:

```bash
kubectl get service \
  -n monitoring \
  -l app.kubernetes.io/name=grafana

kubectl get secret -n monitoring | grep grafana
```

При названии релиза `prometheus` обычно используются Service и Secret `prometheus-grafana`.

Получите логин и пароль локально. Не отправляйте их в чат и не сохраняйте в репозитории:

```bash
kubectl get secret prometheus-grafana \
  -n monitoring \
  -o jsonpath='{.data.admin-user}' | base64 --decode

kubectl get secret prometheus-grafana \
  -n monitoring \
  -o jsonpath='{.data.admin-password}' | base64 --decode
```

Откройте локальный туннель:

```bash
kubectl port-forward \
  -n monitoring \
  service/prometheus-grafana \
  3000:80
```

Пока команда выполняется, Grafana доступна только на локальном компьютере:

```text
http://127.0.0.1:3000
```

Публичный LoadBalancer для Grafana не создаётся.

В Grafana откройте `Explore`, выберите datasource Prometheus и выполните:

```promql
up{namespace="staging"}
```

## Что будет дальше

После появления трёх application targets добавляется первый `PrometheusRule`: недоступность Pod и тестовый alert. Затем создаётся небольшой Grafana dashboard для staging, после чего можно переходить к Application Load Balancer.
