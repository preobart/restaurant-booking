# RabbitMQ в Yandex Managed Kubernetes

Этот каталог описывает учебный production-like кластер RabbitMQ. Он создаёт три RabbitMQ-реплики без публичного доступа, три постоянных SSD-диска Yandex Cloud и защиту кворума при добровольном отключении одного узла.

Микросервисы на этом шаге не разворачиваются и не изменяются.

## Что уже подготовлено

- Kubernetes-кластер `lab-k8s`, Kubernetes `1.35`, три worker-узла;
- namespace `platform` с квотами;
- External Secrets Operator `v2.5.0`, Helm chart `2.5.0-2`, namespace `external-secrets`;
- Kubernetes Secret `external-secrets/sa-creds` с авторизованным ключом сервисного аккаунта;
- Lockbox secret `lab-rabbitmq-admin`, ID `e6qtimi50pl3hc33d1v4`, ключи `username` и `password`;
- роль `lockbox.payloadViewer` выдана сервисному аккаунту только на этот секрет.

Не добавляйте в Git пароль RabbitMQ, содержимое `sa-creds` или файл авторизованного ключа.

## Версии и совместимость

- cert-manager `v1.21.1`;
- RabbitMQ Cluster Operator `v2.22.3`;
- RabbitMQ `4.3.1-management` с зафиксированным OCI digest;
- External Secrets Operator chart `2.5.0-2`.

cert-manager 1.21 поддерживает и тестирует Kubernetes 1.35. RabbitMQ Cluster Operator 2.22.3 требует Kubernetes 1.31 или новее, но в его опубликованной документации тестирование указано только до Kubernetes 1.32. Для этого учебного стенда установка разрешена только после успешной проверки CRD, deployment и admission webhook. Для настоящего production необходимо использовать сочетание версий из официально протестированной матрицы либо получить поддержку производителя.

## Состав каталога

- `00-external-secret.yaml` — подключает External Secrets Operator к Yandex Lockbox и создаёт `platform/lab-rabbitmq-default-user`;
- `10-rabbitmq-cluster.yaml` — создаёт три RabbitMQ-реплики и три PVC;
- `20-pod-disruption-budget.yaml` — требует, чтобы при добровольных disruption оставались доступны минимум две реплики.

## 1. Проверить существующие предпосылки

Работайте с компьютера, с которого уже выполняется `kubectl get nodes`:

```bash
kubectl get nodes
kubectl get storageclass yc-network-ssd
helm status external-secrets -n external-secrets
kubectl get secret sa-creds -n external-secrets
kubectl get crd clustersecretstores.external-secrets.io externalsecrets.external-secrets.io
```

Не выводите `sa-creds` с `-o yaml` или `-o json`.

## 2. Установить cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.21.1/cert-manager.yaml
```

Дождитесь готовности трёх deployment:

```bash
kubectl rollout status deployment/cert-manager -n cert-manager --timeout=5m
kubectl rollout status deployment/cert-manager-cainjector -n cert-manager --timeout=5m
kubectl rollout status deployment/cert-manager-webhook -n cert-manager --timeout=5m
```

## 3. Установить RabbitMQ Cluster Operator

```bash
kubectl apply -f https://github.com/rabbitmq/cluster-operator/releases/download/v2.22.3/cluster-operator.yml
```

Проверьте CRD и controller:

```bash
kubectl wait --for=condition=Established crd/rabbitmqclusters.rabbitmq.com --timeout=2m
kubectl rollout status deployment/rabbitmq-cluster-operator -n rabbitmq-system --timeout=5m
kubectl get pods -n rabbitmq-system
```

Если CRD не получает `Established`, deployment не становится готовым или в логах webhook/controller есть ошибки совместимости с Kubernetes 1.35, остановитесь и не применяйте `10-rabbitmq-cluster.yaml`.

## 4. Синхронизировать учётные данные из Lockbox

```bash
kubectl apply -f deploy/k8s/platform/rabbitmq/00-external-secret.yaml
```

Проверьте хранилище и синхронизацию:

```bash
kubectl wait --for=condition=Ready clustersecretstore/cluster-secret-store --timeout=2m
kubectl wait --for=condition=Ready externalsecret/lab-rabbitmq-default-user -n platform --timeout=2m
kubectl get clustersecretstore cluster-secret-store
kubectl get externalsecret lab-rabbitmq-default-user -n platform
kubectl get secret lab-rabbitmq-default-user -n platform
```

Последняя команда проверяет только существование Secret. Не добавляйте `-o yaml`, `-o json` или jsonpath для полей `username` и `password`.

## 5. Выполнить серверный dry-run RabbitMQ-ресурсов

После установки CRD и появления внешнего Secret:

```bash
kubectl apply --dry-run=server -f deploy/k8s/platform/rabbitmq/10-rabbitmq-cluster.yaml
kubectl apply --dry-run=server -f deploy/k8s/platform/rabbitmq/20-pod-disruption-budget.yaml
```

Обе команды должны завершиться без ошибок admission webhook.

## 6. Создать RabbitMQ и PDB

```bash
kubectl apply -f deploy/k8s/platform/rabbitmq/10-rabbitmq-cluster.yaml
kubectl apply -f deploy/k8s/platform/rabbitmq/20-pod-disruption-budget.yaml
```

Дождитесь трёх готовых pod:

```bash
kubectl rollout status statefulset/lab-rabbitmq-server -n platform --timeout=10m
kubectl get pods -n platform -l app.kubernetes.io/name=lab-rabbitmq -o wide
```

Все три pod должны иметь `READY 1/1`, а столбец `NODE` должен содержать три разных worker-узла.

## 7. Проверить диски, сервисы и кворум

```bash
kubectl get pvc -n platform
kubectl get service -n platform -l app.kubernetes.io/name=lab-rabbitmq -o wide
kubectl get poddisruptionbudget lab-rabbitmq -n platform
kubectl exec -n platform lab-rabbitmq-server-0 -- rabbitmq-diagnostics -q cluster_status
```

Ожидаемый результат:

- PVC `persistence-lab-rabbitmq-server-0`, `persistence-lab-rabbitmq-server-1` и `persistence-lab-rabbitmq-server-2` находятся в состоянии `Bound`, каждый размером `5Gi`;
- основной Service имеет тип `ClusterIP`, headless Service имеет `CLUSTER-IP: None`;
- ни один Service не имеет `EXTERNAL-IP`;
- PDB требует две доступные реплики;
- `cluster_status` показывает три RabbitMQ-узла.

## 8. Временно открыть management UI

Публичный Service или Ingress не создаётся. Для учебного доступа используйте локальный port-forward:

```bash
kubectl port-forward -n platform service/lab-rabbitmq 15672:15672
```

Откройте `http://127.0.0.1:15672`. Логин — значение `username` из Lockbox, пароль — сохранённое значение `password`. После проверки остановите port-forward сочетанием `Ctrl+C`.

## 9. Проверить восстановление одного pod

```bash
kubectl delete pod lab-rabbitmq-server-0 -n platform
kubectl get pods -n platform -l app.kubernetes.io/name=lab-rabbitmq -o wide --watch
```

Дождитесь возвращения `lab-rabbitmq-server-0` в `Running 1/1`, затем остановите watch сочетанием `Ctrl+C` и повторите:

```bash
kubectl exec -n platform lab-rabbitmq-server-0 -- rabbitmq-diagnostics -q cluster_status
kubectl get pvc -n platform
```

Удаляйте только один RabbitMQ pod за раз. Одновременная потеря двух из трёх узлов лишает quorum queue большинства.

## Короткая пауза

Если Kubernetes остаётся включённым, RabbitMQ можно временно масштабировать в ноль, сохранив PVC:

```bash
kubectl patch rabbitmqcluster lab-rabbitmq -n platform --type=merge -p '{"spec":{"replicas":0}}'
```

Вернуть три реплики:

```bash
kubectl patch rabbitmqcluster lab-rabbitmq -n platform --type=merge -p '{"spec":{"replicas":3}}'
```

Scale-to-zero не удаляет три сетевых SSD и не останавливает worker-узлы, поэтому диски и Kubernetes продолжают тарифицироваться.

## Полное удаление RabbitMQ

Сначала удалите PDB и RabbitmqCluster:

```bash
kubectl delete -f deploy/k8s/platform/rabbitmq/20-pod-disruption-budget.yaml
kubectl delete -f deploy/k8s/platform/rabbitmq/10-rabbitmq-cluster.yaml
```

Проверьте оставшиеся PVC и удалите только три PVC этого кластера:

```bash
kubectl get pvc -n platform
kubectl delete pvc -n platform \
  persistence-lab-rabbitmq-server-0 \
  persistence-lab-rabbitmq-server-1 \
  persistence-lab-rabbitmq-server-2
```

Затем удалите project-owned синхронизацию секрета:

```bash
kubectl delete -f deploy/k8s/platform/rabbitmq/00-external-secret.yaml
```

В интерфейсе Compute Cloud убедитесь, что три диска RabbitMQ исчезли. StorageClass `yc-network-ssd` использует reclaim policy `Delete`, но проверка отсутствия дисков обязательна.

External Secrets Operator и cert-manager оставьте установленными, если они нужны следующим платформенным компонентам. RabbitMQ Cluster Operator удаляйте только после удаления всех `RabbitmqCluster`:

```bash
kubectl delete -f https://github.com/rabbitmq/cluster-operator/releases/download/v2.22.3/cluster-operator.yml
```

При полном завершении лаборатории также удалите авторизованный ключ `lab-external-secrets`, Lockbox secret `lab-rabbitmq-admin` и сервисный аккаунт `lab-external-secrets` через Yandex Cloud UI.
