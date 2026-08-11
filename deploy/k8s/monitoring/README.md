# Метрики Kubernetes и staging-приложений

Prometheus Operator установлен через Yandex Cloud Marketplace в namespace `monitoring` и пишет метрики в Managed Prometheus workspace:

```text
mond7en8jco98a3qtrg6
```

Prometheus уже собирает Kubernetes-, node- и container-метрики. Дополнительно этот каталог содержит опциональную настройку сбора `/metrics` у трёх внутренних сервисов:

```text
auth-service.staging.svc.cluster.local:8081/metrics
catalog-service.staging.svc.cluster.local:8082/metrics
booking-service.staging.svc.cluster.local:8083/metrics
```

Service не публикуются наружу. Prometheus получает доступ к их HTTP-портам через отдельную NetworkPolicy.

## Текущее состояние application metrics

Все три Service отвечают `404 Not Found` на `/metrics`. Это подтверждает, что ServiceMonitor обнаружил цели и сетевой доступ работает, но приложения пока не экспортируют Prometheus metrics.

До добавления `/metrics` удалите опциональные объекты, чтобы не хранить постоянные ложные targets со значением `up = 0`:

```bash
kubectl delete servicemonitor restaurant-booking-staging \
  -n monitoring \
  --ignore-not-found

kubectl delete networkpolicy allow-prometheus-scrape \
  -n staging \
  --ignore-not-found
```

Файлы `00-prometheus-scrape-network-policy.yaml` и `10-staging-service-monitor.yaml` остаются шаблоном на будущее. Не применяйте их снова, пока `/metrics` не начнёт возвращать HTTP `200`.

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

## 2. Проверить опциональные манифесты на API-сервере

Этот и следующие два раздела выполняются только после реализации `/metrics` в приложениях.

```bash
kubectl apply --dry-run=server \
  -f deploy/k8s/monitoring/00-prometheus-scrape-network-policy.yaml \
  -f deploy/k8s/monitoring/10-staging-service-monitor.yaml
```

Dry-run должен распознать `ServiceMonitor`. Ошибка `no matches for kind ServiceMonitor` означает, что установка Operator ещё не завершилась или его CRD отсутствует.

## 3. Разрешить scrape и создать ServiceMonitor после появления `/metrics`

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

## 4. Проверить application metrics в Yandex Monitoring

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
kube_deployment_status_replicas_available{namespace="staging"}
```

## 6. Понять два независимых контура правил

`PrometheusRule` в Kubernetes вычисляет локальный Prometheus. Такой alert виден на локальной странице Prometheus, но сам по себе не передаётся в Managed Alertmanager и не вызывает письмо Yandex Monitoring.

Для уведомлений Yandex Cloud правила загружаются отдельными `.yml`-файлами непосредственно в Managed Prometheus workspace. Именно эти правила вычисляет Yandex Cloud, после чего Managed Alertmanager направляет уведомления в созданный канал.

Если старые локальные правила уже применены, удалите их, чтобы не видеть два одноимённых alert из разных контуров:

```bash
kubectl delete prometheusrule \
  restaurant-booking-monitoring-test \
  restaurant-booking-staging \
  -n monitoring \
  --ignore-not-found
```

Удаление этих двух правил не удаляет Prometheus, метрики, Deployment или Pod.

## 7. Создать канал уведомлений Yandex Monitoring

В каталоге с workspace откройте:

```text
Monitoring -> Каналы уведомлений -> Создать канал
```

Укажите:

```text
Имя: lab-alerts-email
Метод: Email
Получатель: ваш аккаунт Yandex Cloud
```

Получатель должен иметь роль `monitoring.viewer` на каталог и включённые уведомления Monitoring в настройках своего аккаунта.

## 8. Загрузить конфигурацию Managed Alertmanager

Файл `deploy/monitoring/yandex/alertmanager.yaml` маршрутизирует alerts с меткой `severity="warning"` в канал `lab-alerts-email`. Alerts без этой метки никуда не отправляются.

Откройте:

```text
Monitoring -> Prometheus -> workspace mond7en8jco98a3qtrg6
-> Конфигурация Alert manager
```

Нажмите `Загрузить файл конфигурации` и выберите:

```text
deploy/monitoring/yandex/alertmanager.yaml
```

После загрузки убедитесь, что у получателя `lab-email` найден канал `lab-alerts-email`. Получатель `null` без канала нужен как маршрут по умолчанию и не является ошибкой.

## 9. Загрузить тестовое правило в Managed Prometheus

Откройте:

```text
Monitoring -> Поставка и хранение -> Prometheus
-> workspace mond7en8jco98a3qtrg6 -> Правила
```

Нажмите `Добавить файл` и выберите:

```text
deploy/monitoring/yandex/rules/monitoring-test.yml
```

Файл содержит всегда истинное выражение `vector(1)` и не изменяет приложения или кластер. Подождите 1–2 минуты и в этом же workspace откройте вкладку `Алерты`. `MonitoringPipelineTest` должен перейти в `FIRING`, после чего должно прийти письмо в канал `lab-alerts-email`.

Если alert находится в `FIRING`, но письма нет, проверьте:

- что это вкладка `Алерты` именно Managed Prometheus workspace, а не локальная страница `127.0.0.1:9090`;
- что в конфигурации Alertmanager получатель `lab-email` связан с существующим каналом `lab-alerts-email`;
- папки `Спам`, `Рассылки` и задержку доставки в несколько минут;
- что email-получатель канала имеет доступ к каталогу и включён в настройках канала.

После получения письма удалите `monitoring-test.yml` во вкладке `Правила` через меню файла. Уведомление о разрешении alert после удаления ожидаемо.

## 10. Загрузить постоянное правило доступности staging

В ту же вкладку `Правила` добавьте файл:

```text
deploy/monitoring/yandex/rules/staging-alerts.yml
```

Правило использует уже поступающие метрики `kube-state-metrics`, поэтому `/metrics` в приложениях для него не нужен. `StagingDeploymentReplicasUnavailable` переходит в `FIRING`, если любой Deployment в `staging` более двух минут имеет меньше доступных реплик, чем указано в `spec.replicas`.

Постоянный файл `staging-alerts.yml` оставьте в workspace.

## Что будет дальше

После проверки email-уведомления можно переходить к Application Load Balancer. Application metrics остаются отдельным улучшением после реализации `/metrics` в сервисах.
