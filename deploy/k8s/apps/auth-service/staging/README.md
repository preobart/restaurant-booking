# Auth service в staging

Этот каталог разворачивает первую прикладную нагрузку в учебном кластере `lab-k8s`:

```text
Yandex Lockbox
  -> External Secrets Operator
  -> staging/auth-service

auth-service
  -> TLS 6432 -> Managed PostgreSQL / auth_staging
  -> AMQP 5672 -> platform/lab-rabbitmq
```

Микросервис использует immutable-образ из Yandex Container Registry. Публичный `Ingress` или `LoadBalancer` на этом этапе не создаётся.

## Что создаётся

- отдельный `ServiceAccount` без автоматического Kubernetes API token;
- `ExternalSecret`, собирающий PostgreSQL DSN, JWT и RabbitMQ URL из трёх Lockbox-секретов;
- ConfigMap с несекретными параметрами приложения;
- три минимальные NetworkPolicy;
- одноразовый Job миграций;
- один staging Pod `auth-service`;
- внутренний `ClusterIP Service` на порту `8081`.

## Зафиксированные параметры

- образ: `cr.yandex/crpfrtdcq05e9r2l43ro/auth-service@sha256:01025f039ba3897b6ff2b02bbdced4fc312f5ce4533f757367523c06b49cacbf`;
- PostgreSQL: `c-c9q3gaf98a21tn8dvb68.rw.mdb.yandexcloud.net:6432`;
- база и пользователь: `auth_staging`;
- PostgreSQL Lockbox secret: `e6qrgpbvtvlhtga4ir8m`, property `postgresql_password`;
- staging JWT Lockbox secret: `e6q1defjt7a1d78bg8kd`, property `jwt_secret`;
- RabbitMQ Lockbox secret: `e6qtimi50pl3hc33d1v4`, properties `username` и `password`;
- RabbitMQ endpoint: `lab-rabbitmq.platform.svc.cluster.local:5672`.

Значения паролей и JWT в репозитории отсутствуют.

## 1. Проверить предпосылки в Yandex Cloud UI

В группе безопасности `lab-postgresql` должно быть входящее правило:

```text
TCP 6432
Источник: группа безопасности lab-k8s-nodes
```

Сервисному аккаунту `lab-external-secrets` должна быть выдана роль `lockbox.payloadViewer` отдельно на:

```text
e6qrgpbvtvlhtga4ir8m
e6q1defjt7a1d78bg8kd
e6qtimi50pl3hc33d1v4
```

В PostgreSQL-секрете проверьте только имя свойства. Если оно отличается от `postgresql_password`, исправьте `remoteRef.property` в `10-external-secret.yaml`. Значение свойства не копируйте и не выводите.

## 2. Проверить инфраструктуру Kubernetes

```bash
kubectl get nodes
kubectl get clustersecretstore cluster-secret-store
kubectl get pods -n external-secrets
kubectl get pods -n platform -l app.kubernetes.io/name=lab-rabbitmq
```

Ожидается:

- три worker-узла `Ready`;
- `ClusterSecretStore` имеет `Ready=True`;
- три Pod External Secrets Operator имеют `Running 1/1`;
- три RabbitMQ Pod имеют `Running 1/1`.

## 3. Выполнить server-side dry-run

Dry-run проверит объекты с учётом Pod Security, LimitRange, квот и установленных CRD, но ничего не создаст:

```bash
kubectl apply --dry-run=server \
  -f deploy/k8s/environments/staging/10-yandex-internal-root-ca.yaml \
  -f deploy/k8s/apps/auth-service/staging/00-service-account.yaml \
  -f deploy/k8s/apps/auth-service/staging/10-external-secret.yaml \
  -f deploy/k8s/apps/auth-service/staging/20-configmap.yaml \
  -f deploy/k8s/apps/auth-service/staging/30-network-policies.yaml \
  -f deploy/k8s/apps/auth-service/staging/40-migration-job.yaml \
  -f deploy/k8s/apps/auth-service/staging/50-deployment.yaml \
  -f deploy/k8s/apps/auth-service/staging/60-service.yaml
```

Dry-run проверяет Kubernetes-схему, но не читает payload Lockbox. Точное имя свойства `postgresql_password` подтверждается в UI Lockbox, а затем фактической синхронизацией `ExternalSecret` на следующем шаге.

## 4. Создать настройки и синхронизировать секреты

```bash
kubectl apply \
  -f deploy/k8s/environments/staging/10-yandex-internal-root-ca.yaml \
  -f deploy/k8s/apps/auth-service/staging/00-service-account.yaml \
  -f deploy/k8s/apps/auth-service/staging/10-external-secret.yaml \
  -f deploy/k8s/apps/auth-service/staging/20-configmap.yaml \
  -f deploy/k8s/apps/auth-service/staging/30-network-policies.yaml
```

Дождитесь синхронизации:

```bash
kubectl wait \
  --for=condition=Ready \
  externalsecret/auth-service \
  -n staging \
  --timeout=2m

kubectl get externalsecret auth-service -n staging
kubectl get secret auth-service -n staging
```

Проверяйте только наличие Secret и количество ключей. Не используйте для него `-o yaml`, `-o json` или jsonpath, выводящий данные.

Если ExternalSecret не становится готовым:

```bash
kubectl describe externalsecret auth-service -n staging
```

Проверьте ID, property и роль `lockbox.payloadViewer` на каждом из трёх исходных секретов.

## 5. Выполнить миграцию auth_staging

Сначала удалите только старый завершённый Job с тем же именем, если он остался от прошлого запуска:

```bash
kubectl delete job auth-service-migrations \
  -n staging \
  --ignore-not-found

kubectl apply \
  -f deploy/k8s/apps/auth-service/staging/40-migration-job.yaml

kubectl wait \
  --for=condition=Complete \
  job/auth-service-migrations \
  -n staging \
  --timeout=10m
```

Проверьте журнал миграции:

```bash
kubectl logs job/auth-service-migrations -n staging
```

Не переходите к Deployment, если Job получил `Failed` или не завершился. Диагностика:

```bash
kubectl describe job auth-service-migrations -n staging
kubectl get pods -n staging -l app.kubernetes.io/component=migration
kubectl get events -n staging --sort-by=.lastTimestamp
```

## 6. Развернуть auth-service

Только после успешной миграции:

```bash
kubectl apply \
  -f deploy/k8s/apps/auth-service/staging/50-deployment.yaml \
  -f deploy/k8s/apps/auth-service/staging/60-service.yaml

kubectl rollout status \
  deployment/auth-service \
  -n staging \
  --timeout=5m
```

Проверьте объекты:

```bash
kubectl get deployment -n staging -o wide
kubectl get pods -n staging -o wide
kubectl get service -n staging -o wide
kubectl get networkpolicy -n staging
```

Ожидается:

- Deployment `1/1`;
- один Pod `Running 1/1`;
- Service имеет тип `ClusterIP`;
- `EXTERNAL-IP` отсутствует;
- Pod находится на узле с меткой `workload=general`.

## 7. Временно проверить HTTP без публикации сервиса

Пока используются TCP startup/readiness/liveness probes. Они подтверждают, что приложение принимает соединения на `8081`, но не проверяют бизнес-зависимости.

Откройте локальный port-forward:

```bash
kubectl port-forward \
  -n staging \
  service/auth-service \
  8081:8081
```

В другом окне выполните:

```bash
curl -i http://127.0.0.1:8081/live
curl -i http://127.0.0.1:8081/ready
curl -i http://127.0.0.1:8081/metrics
```

Если путь существует, ожидается HTTP `200`. Ответ `404` означает, что endpoint не реализован или называется иначе; Deployment при этом останется работоспособным благодаря TCP-пробам. Не подменяйте `/ready` случайным бизнес-endpoint: readiness должен отражать готовность Pod принимать трафик.

После проверки остановите port-forward сочетанием `Ctrl+C`.

## 8. Что именно ограничивает сеть

В namespace уже действует общий `default-deny-ingress` и `default-deny-egress`.

Новые политики разрешают Pod auth-service:

- DNS через существующую общую политику;
- PostgreSQL только в `10.20.2.0/24` и `10.20.3.0/24` на TCP `6432`;
- RabbitMQ только в namespace `platform`, только к RabbitMQ Pod и только на TCP `5672`;
- входящий HTTP только от staging Pod с меткой `networking.restaurant-booking/access-auth-service=true`.

Публичного ingress на этом этапе нет. Локальный `kubectl port-forward` используется только для учебной проверки.

## 9. Почему пока нет HPA и PDB

Staging начинается с одной реплики. PDB для единственного Pod мог бы мешать обслуживанию и drain узла, а HPA усложнил бы первичную диагностику.

После успешного запуска:

1. подтвердим настоящие `/live`, `/ready` и `/metrics`;
2. создадим production overlay с двумя репликами;
3. добавим PDB `minAvailable: 1`;
4. добавим HPA `2-6`;
5. проверим rolling update и удаление одного Pod;
6. затем подключим Application Load Balancer.

## 10. Как ускорить следующие сервисы

Не нужно заново проектировать инфраструктуру для каждой базы. Для `catalog-service` и `booking-service` повторяется один шаблон:

1. взять immutable image digest;
2. найти ID автоматически созданного PostgreSQL Lockbox secret;
3. выдать `lab-external-secrets` роль только на этот secret;
4. создать ExternalSecret, Job миграции, Deployment и Service из того же шаблона;
5. добавить только нужные сетевые направления.

Один JWT secret используется всеми тремя сервисами одного окружения:

```text
staging: e6q1defjt7a1d78bg8kd
production: будет отдельный общий production JWT secret
```

RabbitMQ secret также переиспользуется. Уникальным для каждого сервиса и окружения остаётся только PostgreSQL secret.

Сохранять одного общего PostgreSQL-пользователя для всех баз не следует: отдельные пользователи ограничивают последствия утечки и позволяют независимо менять пароли.

## Откат только auth-service staging

```bash
kubectl delete \
  -f deploy/k8s/apps/auth-service/staging/60-service.yaml \
  -f deploy/k8s/apps/auth-service/staging/50-deployment.yaml

kubectl delete job auth-service-migrations \
  -n staging \
  --ignore-not-found

kubectl delete \
  -f deploy/k8s/apps/auth-service/staging/30-network-policies.yaml \
  -f deploy/k8s/apps/auth-service/staging/20-configmap.yaml \
  -f deploy/k8s/apps/auth-service/staging/10-external-secret.yaml \
  -f deploy/k8s/apps/auth-service/staging/00-service-account.yaml
```

Общий `staging/yandex-internal-root-ca` оставьте для следующих микросервисов. Не удаляйте `ClusterSecretStore`, External Secrets Operator, PostgreSQL, RabbitMQ или namespace `staging`.
