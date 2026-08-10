# Managed Valkey в Yandex Managed Kubernetes

Этот каталог подключает namespace `production` к существующему кластеру Yandex Managed Service for Valkey через приватную сеть и TLS. Пароль поступает из Yandex Lockbox и не хранится в Git.

Микросервисы на этом шаге не разворачиваются и не изменяются. Сначала мы проверяем только инфраструктурный путь:

```text
Lockbox -> External Secrets Operator -> Kubernetes Secret
Pod -> DNS -> TLS 6380 -> Managed Valkey
```

## Используемые ресурсы

- кластер Valkey: `c9q09s84iqj5vgc6gkte`;
- writable-FQDN: `c-c9q09s84iqj5vgc6gkte.rw.mdb.yandexcloud.net`;
- TLS-порт: `6380`;
- Lockbox secret ID: `e6qa6q4v5gu33fvi3rkl`;
- свойство Lockbox: `redis_password`;
- Kubernetes Secret: `production/lab-valkey-credentials`;
- подсеть Valkey: `10.20.1.0/24`;
- группа безопасности Valkey: `lab-valkey`;
- группа безопасности worker-узлов: `lab-k8s-nodes`.

## Состав каталога

- `00-external-secret.yaml` — синхронизирует пароль из Lockbox в `production/lab-valkey-credentials`;
- `10-connection-config.yaml` — создаёт параметры подключения и ConfigMap с публичным CA Yandex Cloud;
- `20-network-policy.yaml` — разрешает исходящий TCP-трафик только в `10.20.1.0/24` на порт `6380`;
- `yandex-internal-root-ca.pem` — официальный CA bundle из `https://storage.yandexcloud.net/cloud-certs/CA.pem`.

Общий `ClusterSecretStore/cluster-secret-store` находится в `deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml`. Его также использует RabbitMQ.

## 1. Проверить доступ External Secrets Operator к Lockbox

В консоли Yandex Cloud откройте:

```text
Lockbox -> секрет e6qa6q4v5gu33fvi3rkl -> Права доступа
```

У сервисного аккаунта `lab-external-secrets` должна быть роль `lockbox.payloadViewer` на этом конкретном секрете.

Не выдавайте роль на весь каталог, если она нужна только для этого секрета. Не копируйте пароль, авторизованный ключ или содержимое `external-secrets/sa-creds` в репозиторий.

## 2. Проверить группу безопасности Valkey

Откройте:

```text
Virtual Private Cloud -> Группы безопасности -> lab-valkey
```

Проверьте входящее правило:

| Поле | Значение |
|---|---|
| Направление | Входящий трафик |
| Протокол | TCP |
| Порт | `6380` |
| Тип источника | Группа безопасности |
| Источник | `lab-k8s-nodes` |

Публичный доступ к Valkey не требуется. IP Cloud Shell и домашний публичный IP не нужны: соединение устанавливают приватные worker-узлы Kubernetes.

## 3. Выполнить server-side dry-run

Команды проверят манифесты через API Kubernetes, но не создадут ресурсы:

```bash
kubectl apply --dry-run=server \
  -f deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml

kubectl apply --dry-run=server \
  -f deploy/k8s/platform/valkey/00-external-secret.yaml \
  -f deploy/k8s/platform/valkey/10-connection-config.yaml \
  -f deploy/k8s/platform/valkey/20-network-policy.yaml
```

Если dry-run сообщает, что `ExternalSecret` или `ClusterSecretStore` неизвестен, сначала проверьте установку External Secrets Operator:

```bash
kubectl get pods -n external-secrets
kubectl get crd clustersecretstores.external-secrets.io externalsecrets.external-secrets.io
```

## 4. Применить общий ClusterSecretStore

```bash
kubectl apply \
  -f deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml

kubectl wait \
  --for=condition=Ready \
  clustersecretstore/cluster-secret-store \
  --timeout=2m
```

Ресурс уже может существовать после настройки RabbitMQ. В этом случае `kubectl apply` обновит его без пересоздания.

## 5. Синхронизировать пароль Valkey

```bash
kubectl apply \
  -f deploy/k8s/platform/valkey/00-external-secret.yaml

kubectl wait \
  --for=condition=Ready \
  externalsecret/lab-valkey-credentials \
  -n production \
  --timeout=2m
```

Проверьте только состояние и наличие Secret:

```bash
kubectl get clustersecretstore cluster-secret-store
kubectl get externalsecret lab-valkey-credentials -n production
kubectl get secret lab-valkey-credentials -n production
```

Не добавляйте к команде Secret параметры `-o yaml`, `-o json` или jsonpath для поля `password`.

## 6. Добавить параметры подключения, CA и сетевое разрешение

```bash
kubectl apply \
  -f deploy/k8s/platform/valkey/10-connection-config.yaml \
  -f deploy/k8s/platform/valkey/20-network-policy.yaml
```

Проверьте созданные объекты:

```bash
kubectl get configmap \
  lab-valkey-connection \
  yandex-internal-root-ca \
  -n production

kubectl get networkpolicy allow-valkey-egress -n production
```

NetworkPolicy дополняет существующий `default-deny-egress`. Она разрешает всем Pod в `production` только TCP `6380` в подсеть `10.20.1.0/24`. Когда появятся Deployment микросервисов с окончательными метками, политику нужно сузить до сервисов, которым действительно нужен Valkey.

CA bundle получен из официального хранилища Yandex Cloud 10 августа 2026 года. Его SHA-256:

```text
6d148f85b5213445b23ad22ff45e47e1aa2be968f183f9bd6ff39de54d47a8ef
```

Первый сертификат bundle действует до 20 июня 2027 года. До этой даты обновите файл из официального источника и повторно примените ConfigMap.

## 7. Создать временный клиент Valkey

Этот Pod нужен только для проверки инфраструктуры. Он не является частью production deployment и будет удалён после четырёх команд.

```bash
kubectl apply -f - <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: lab-valkey-smoke-test
  namespace: production
  labels:
    app.kubernetes.io/part-of: restaurant-booking
    app.kubernetes.io/component: valkey-smoke-test
    environment: production
spec:
  automountServiceAccountToken: false
  restartPolicy: Never
  securityContext:
    runAsNonRoot: true
    runAsUser: 999
    runAsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: redis-cli
      image: redis:7.4.2-alpine
      imagePullPolicy: IfNotPresent
      command:
        - /bin/sh
        - -c
      args:
        - sleep 3600
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      resources:
        requests:
          cpu: 25m
          memory: 32Mi
        limits:
          cpu: 100m
          memory: 128Mi
      env:
        - name: REDISCLI_AUTH
          valueFrom:
            secretKeyRef:
              name: lab-valkey-credentials
              key: password
        - name: VALKEY_HOST
          valueFrom:
            configMapKeyRef:
              name: lab-valkey-connection
              key: host
        - name: VALKEY_PORT
          valueFrom:
            configMapKeyRef:
              name: lab-valkey-connection
              key: port
        - name: VALKEY_USERNAME
          valueFrom:
            configMapKeyRef:
              name: lab-valkey-connection
              key: username
      volumeMounts:
        - name: yandex-internal-root-ca
          mountPath: /etc/yandex-ca
          readOnly: true
  volumes:
    - name: yandex-internal-root-ca
      configMap:
        name: yandex-internal-root-ca
        items:
          - key: ca.crt
            path: ca.crt
YAML

kubectl wait \
  --for=condition=Ready \
  pod/lab-valkey-smoke-test \
  -n production \
  --timeout=3m
```

Если Pod не становится `Ready`, выполните:

```bash
kubectl describe pod lab-valkey-smoke-test -n production
kubectl get events -n production --sort-by=.lastTimestamp
```

## 8. Выполнить PING, SET, GET и удалить тестовые данные

Пароль передаётся клиенту через переменную `REDISCLI_AUTH`. Он не появляется в аргументах процесса и истории shell.

```bash
kubectl exec -n production lab-valkey-smoke-test -- sh -c \
  'redis-cli --host "$VALKEY_HOST" --port "$VALKEY_PORT" --user "$VALKEY_USERNAME" --tls --cacert /etc/yandex-ca/ca.crt PING'

kubectl exec -n production lab-valkey-smoke-test -- sh -c \
  'redis-cli --host "$VALKEY_HOST" --port "$VALKEY_PORT" --user "$VALKEY_USERNAME" --tls --cacert /etc/yandex-ca/ca.crt SET restaurant-booking:smoke-test ok EX 60'

kubectl exec -n production lab-valkey-smoke-test -- sh -c \
  'redis-cli --host "$VALKEY_HOST" --port "$VALKEY_PORT" --user "$VALKEY_USERNAME" --tls --cacert /etc/yandex-ca/ca.crt GET restaurant-booking:smoke-test'

kubectl exec -n production lab-valkey-smoke-test -- sh -c \
  'redis-cli --host "$VALKEY_HOST" --port "$VALKEY_PORT" --user "$VALKEY_USERNAME" --tls --cacert /etc/yandex-ca/ca.crt DEL restaurant-booking:smoke-test'
```

Ожидаемые ответы:

```text
PONG
OK
ok
1
```

Удалите временный Pod:

```bash
kubectl delete pod lab-valkey-smoke-test -n production
```

Тестовый ключ дополнительно имеет TTL 60 секунд, поэтому он исчезнет автоматически, даже если команда `DEL` не выполнится.

## 9. Диагностика

### ExternalSecret не становится Ready

```bash
kubectl describe externalsecret lab-valkey-credentials -n production
kubectl get clustersecretstore cluster-secret-store
```

Проверьте:

- роль `lockbox.payloadViewer` у `lab-external-secrets` на secret `e6qa6q4v5gu33fvi3rkl`;
- точное имя свойства `redis_password`;
- наличие `external-secrets/sa-creds`;
- готовность Pod External Secrets Operator.

Не выводите содержимое целевого Kubernetes Secret.

### Соединение завершается по timeout

Проверьте оба независимых уровня:

1. в `lab-valkey` разрешён входящий TCP `6380` из группы `lab-k8s-nodes`;
2. в `production` существует `NetworkPolicy/allow-valkey-egress` с `10.20.1.0/24` и TCP `6380`.

### Ошибка TLS

Проверьте:

- FQDN `c-c9q09s84iqj5vgc6gkte.rw.mdb.yandexcloud.net`;
- порт `6380`;
- наличие `/etc/yandex-ca/ca.crt` в тестовом Pod;
- системное время Pod и worker-узла.

Не добавляйте `--insecure` и не переключайтесь на незашифрованный порт `6379`.

### Имя хоста не разрешается

Проверьте CoreDNS и уже созданную политику `allow-dns-egress`:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get networkpolicy allow-dns-egress -n production
```

Не включайте публичный доступ Valkey для исправления внутреннего DNS.

## 10. Как оставить всё после успешной проверки

Kubernetes Secret, ConfigMap и NetworkPolicy почти не потребляют вычислительные ресурсы. Их можно оставить: они понадобятся будущим Deployment.

Чтобы кластер Valkey не расходовал вычислительные ресурсы между занятиями:

1. откройте `Managed Service for Valkey`;
2. найдите кластер `c9q09s84iqj5vgc6gkte`;
3. откройте меню `⋮`;
4. выберите **Остановить** и подтвердите действие;
5. дождитесь статуса `Stopped`.

Для остановленного кластера вычислительные ресурсы не тарифицируются, но продолжают тарифицироваться хранилище и резервные копии. Перед следующей проверкой запустите кластер из того же меню и дождитесь `Running`.

Документация: [остановка и запуск Managed Valkey](https://yandex.cloud/ru/docs/managed-valkey/operations/cluster-stop).

## 11. Удалить только Kubernetes-интеграцию

Если Valkey больше не нужен будущим Deployment, удалите объекты в таком порядке:

```bash
kubectl delete pod lab-valkey-smoke-test -n production --ignore-not-found
kubectl delete -f deploy/k8s/platform/valkey/20-network-policy.yaml
kubectl delete -f deploy/k8s/platform/valkey/10-connection-config.yaml
kubectl delete -f deploy/k8s/platform/valkey/00-external-secret.yaml
```

Не удаляйте `ClusterSecretStore/cluster-secret-store`, пока его использует RabbitMQ.

Удаление этих Kubernetes-объектов не удаляет Managed Valkey и Lockbox secret. Для полного прекращения расходов удалите сам кластер Valkey через Yandex Cloud UI после проверки, что данные и резервные копии больше не нужны.

## Что дальше

После успешного ответа `PONG` интеграцию Valkey считаем завершённой. Следующий полезный этап — сгенерировать первый production Deployment и Service для микросервиса с probes, двумя репликами, requests/limits, PDB, HPA и подключением уже созданных PostgreSQL, RabbitMQ и Valkey.
