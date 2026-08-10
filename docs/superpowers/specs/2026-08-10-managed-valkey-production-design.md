# Managed Valkey Production Integration Design

## Цель

Завершить подключение существующего Yandex Managed Service for Valkey к namespace `production` в кластере `lab-k8s`. Пароль должен поступать из Yandex Lockbox, соединение должно использовать TLS, а действующие default-deny NetworkPolicy не должны открываться шире необходимого.

Этот этап настраивает инфраструктурный контракт подключения и проверяет доступность Valkey из Kubernetes. Он не разворачивает и не изменяет микросервисы.

## Исходные данные

- кластер Valkey: `c9q09s84iqj5vgc6gkte`;
- writable-FQDN: `c-c9q09s84iqj5vgc6gkte.rw.mdb.yandexcloud.net`;
- диагностический FQDN текущего хоста: `rc1a-oi3754em06envmj0.mdb.yandexcloud.net`;
- TLS-порт: `6380`;
- пользователь Valkey: `default`;
- группа безопасности Valkey: `lab-valkey`;
- группа безопасности узлов Kubernetes: `lab-k8s-nodes`;
- подсеть хоста Valkey `lab-subnet-a`: `10.20.1.0/24`;
- Lockbox secret ID: `e6qa73r663u29svikf0m`;
- имя свойства в Lockbox: `redis_password`;
- сервисный аккаунт External Secrets Operator: `lab-external-secrets`;
- существующий `ClusterSecretStore`: `cluster-secret-store`;
- целевой namespace: только `production`.

## Выбранный подход

Использовать существующий External Secrets Operator и общий `ClusterSecretStore`. Создать в `production`:

1. `ExternalSecret`, который читает только `redis_password` из Lockbox и создаёт Kubernetes Secret `lab-valkey-credentials` с ключом `password`;
2. ConfigMap `lab-valkey-connection` с несекретными параметрами `host`, `port`, `username` и `tls`;
3. ConfigMap `yandex-internal-root-ca` с публичным корневым сертификатом Yandex Cloud;
4. NetworkPolicy `allow-valkey-egress`, разрешающую исходящий TCP-трафик только в `10.20.1.0/24` на порт `6380`.

Этот вариант выбран вместо хранения единой строки подключения в Secret. Пароль остаётся отдельным секретом, а адрес, порт и TLS-настройки можно менять без ротации Lockbox.

Staging на этом этапе не подключается. Позже для него следует создать отдельного пользователя Valkey и отдельный Lockbox secret, чтобы не делить production-учётные данные.

## Общий ClusterSecretStore

Сейчас `cluster-secret-store` объявлен в RabbitMQ-манифесте. После появления второго потребителя это создаёт неверное владение: удаление RabbitMQ-манифеста может удалить хранилище секретов Valkey.

Ресурс будет перенесён в отдельный файл:

```text
deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml
```

RabbitMQ-файл сохранит только свой `ExternalSecret`. Имя и спецификация уже применённого `ClusterSecretStore` не изменятся, поэтому перенос не пересоздаёт ресурс и не прерывает синхронизацию RabbitMQ.

## Поток данных и секретов

Поток пароля:

```text
Yandex Lockbox
  -> External Secrets Operator
  -> production/lab-valkey-credentials
  -> будущий Pod микросервиса
```

Поток подключения:

```text
будущий Pod
  -> DNS writable-FQDN
  -> TLS с проверкой Yandex Internal Root CA
  -> 10.20.1.0/24:6380
  -> Managed Valkey
```

В Git не попадают пароль, содержимое `external-secrets/sa-creds` и готовый URL с паролем. Публичный корневой сертификат Yandex Cloud не является секретом и хранится в репозитории для воспроизводимого запуска без загрузки из интернета во время старта Pod.

## IAM и группа безопасности

Перед применением Kubernetes-манифестов пользователь вручную выполняет в Yandex Cloud UI два действия:

1. выдаёт сервисному аккаунту `lab-external-secrets` роль `lockbox.payloadViewer` только на secret `e6qa73r663u29svikf0m`;
2. проверяет в `lab-valkey` входящее правило TCP `6380`, где источник — группа безопасности `lab-k8s-nodes`.

Публичный доступ к Valkey не включается. CIDR Cloud Shell и домашний публичный IP для этого подключения не требуются: к Valkey обращаются приватные worker-узлы Kubernetes.

## Сетевая политика

В `production` уже действуют `default-deny-egress` и отдельное разрешение DNS. Новая политика добавит только TCP `6380` к `ipBlock.cidr: 10.20.1.0/24`.

На первом учебном этапе политика применяется ко всем Pod в `production`, потому что deployment-метки микросервисов ещё не зафиксированы в Kubernetes-манифестах. Ограничение остаётся узким по подсети и порту. После генерации production Deployment политику следует сузить `podSelector` до сервисов, которым действительно нужен Valkey.

Kubernetes NetworkPolicy не заменяет Security Group: для успешного соединения одновременно нужны egress-политика namespace и ingress-правило `lab-valkey`.

## TLS и адрес подключения

Приложения должны использовать `c-c9q09s84iqj5vgc6gkte.rw.mdb.yandexcloud.net`, а не FQDN конкретного хоста. Специальный writable-FQDN сохраняет контракт подключения при последующем добавлении реплики или смене мастера.

Подключение выполняется на `6380` с проверкой сертификата через Yandex Internal Root CA. Режим `--insecure` и отключение проверки TLS не используются даже в проверочном Pod.

## Структура файлов

```text
deploy/k8s/platform/external-secrets/
└── 00-cluster-secret-store.yaml

deploy/k8s/platform/valkey/
├── 00-external-secret.yaml
├── 10-connection-config.yaml
├── 20-network-policy.yaml
├── yandex-internal-root-ca.pem
└── README.md
```

- `00-external-secret.yaml` создаёт только `production/lab-valkey-credentials`;
- `10-connection-config.yaml` создаёт ConfigMap параметров подключения и ConfigMap корневого CA;
- `20-network-policy.yaml` добавляет минимальный egress к Valkey;
- `yandex-internal-root-ca.pem` содержит публичный CA из официального хранилища Yandex Cloud;
- `README.md` описывает UI-предпосылки, применение, безопасную проверку и удаление.

## Порядок применения

1. Проверить роль `lockbox.payloadViewer` на конкретном Lockbox secret.
2. Проверить ingress TCP `6380` из `lab-k8s-nodes` в SG `lab-valkey`.
3. Применить общий `ClusterSecretStore` из нового каталога.
4. Применить `ExternalSecret` Valkey и дождаться `Ready=True`.
5. Применить ConfigMap подключения и CA.
6. Применить NetworkPolicy.
7. Запустить временный клиентский Pod в `production`, выполнить TLS `PING`, тестовые `SET` и `GET`, затем удалить Pod и тестовый ключ.

Секрет проверяется только по наличию и состоянию ExternalSecret. Инструкции не выводят значение пароля в терминал.

## Проверка

Локальная проверка файлов должна подтвердить:

- все YAML разбираются корректно;
- `ExternalSecret` ссылается на `cluster-secret-store`, secret ID `e6qa73r663u29svikf0m` и property `redis_password`;
- ни один файл не содержит пароль или авторизованный ключ;
- целевой Secret создаётся только в `production`;
- host равен writable-FQDN, port равен `6380`, TLS включён;
- NetworkPolicy разрешает только `10.20.1.0/24` и TCP `6380`;
- общий `ClusterSecretStore` определён в репозитории ровно один раз.

После ручного применения пользователь проверяет:

- `ClusterSecretStore` и `ExternalSecret` имеют состояние `Ready=True`;
- Secret `production/lab-valkey-credentials` существует;
- временный клиент получает `PONG` по TLS;
- запись и чтение тестового ключа успешны;
- клиент не создаёт LoadBalancer, Ingress или публичный IP;
- после проверки временный Pod и тестовый ключ удалены.

Проверка бизнес-логики микросервисов не входит в этот этап.

## Ошибки и диагностика

- `ExternalSecret Ready=False`: проверить роль на конкретном Lockbox secret, точное имя `redis_password` и состояние `cluster-secret-store`.
- timeout на `6380`: проверить оба сетевых уровня — ingress SG `lab-valkey` и egress NetworkPolicy `production`.
- ошибка TLS: проверить writable-FQDN, порт `6380`, монтирование CA и системное время Pod; не обходить ошибку через insecure-режим.
- DNS не разрешается: проверить существующую `allow-dns-egress` и CoreDNS, не добавлять публичный доступ Valkey.

## Удаление и остановка

Для завершения только Kubernetes-интеграции удалить временный клиент, NetworkPolicy, обе ConfigMap и `ExternalSecret`. Общий `ClusterSecretStore` и External Secrets Operator оставить, пока их использует RabbitMQ.

Удаление Kubernetes Secret, созданного `ExternalSecret`, не удаляет исходный Lockbox secret или Managed Valkey. Для прекращения тарификации самого Valkey использовать остановку или удаление кластера в Yandex Cloud UI после проверки зависимостей. Остановка не является частью автоматического применения этих манифестов.
