# Managed Valkey Production Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Подготовить воспроизводимые Kubernetes-манифесты и русскоязычную инструкцию для безопасного TLS-подключения namespace `production` к существующему Yandex Managed Service for Valkey.

**Architecture:** Существующий External Secrets Operator синхронизирует один пароль из Yandex Lockbox в Kubernetes Secret. Несекретные параметры и официальный Yandex CA хранятся в ConfigMap, а отдельная NetworkPolicy разрешает production Pod только TCP `6380` в подсеть Valkey `10.20.1.0/24`. Микросервисы, Yandex Cloud и работающий Kubernetes-кластер в ходе генерации файлов не изменяются.

**Tech Stack:** Kubernetes `1.35`, External Secrets Operator API `external-secrets.io/v1`, Yandex Lockbox, Yandex Managed Service for Valkey, Kubernetes NetworkPolicy, YAML, OpenSSL.

## Global Constraints

- Целевой namespace — только `production`.
- Lockbox secret ID — `e6qa73r663u29svikf0m`; property — `redis_password`.
- Writable-FQDN — `c-c9q09s84iqj5vgc6gkte.rw.mdb.yandexcloud.net`.
- Использовать только TLS-порт `6380`; не использовать `6379` и insecure-режим.
- Egress разрешать только в `10.20.1.0/24` на TCP `6380`.
- Не добавлять пароль, содержимое `sa-creds` или URL с паролем в Git и команды проверки.
- Не применять манифесты к кластеру и не изменять ресурсы Yandex Cloud: пользователь выполнит это вручную.
- Не изменять код и Deployment микросервисов.
- Общий `ClusterSecretStore` должен быть определён в репозитории ровно один раз.

---

### Task 1: Вынести общий ClusterSecretStore из каталога RabbitMQ

**Files:**
- Create: `deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml`
- Modify: `deploy/k8s/platform/rabbitmq/00-external-secret.yaml`
- Modify: `deploy/k8s/platform/rabbitmq/README.md`

**Interfaces:**
- Consumes: Kubernetes Secret `external-secrets/sa-creds`, уже созданный Helm-релиз External Secrets Operator.
- Produces: единственный общий `ClusterSecretStore/cluster-secret-store`, который используют RabbitMQ и Valkey.

- [ ] **Step 1: Зафиксировать проверку текущего дублирования**

Run:

```bash
rg -n '^kind: ClusterSecretStore$' deploy/k8s
```

Expected: одна строка в `deploy/k8s/platform/rabbitmq/00-external-secret.yaml`; её расположение не соответствует общему владению ресурсом.

- [ ] **Step 2: Создать общий манифест ClusterSecretStore**

Create `deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml`:

```yaml
apiVersion: external-secrets.io/v1
kind: ClusterSecretStore
metadata:
  name: cluster-secret-store
  labels:
    app.kubernetes.io/part-of: restaurant-booking
    app.kubernetes.io/component: secrets
spec:
  provider:
    yandexlockbox:
      auth:
        authorizedKeySecretRef:
          name: sa-creds
          namespace: external-secrets
          key: key
```

- [ ] **Step 3: Оставить в RabbitMQ-файле только ExternalSecret**

Modify `deploy/k8s/platform/rabbitmq/00-external-secret.yaml` so its first line is the existing:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: lab-rabbitmq-default-user
```

Delete the former `ClusterSecretStore` document and its following `---`. Do not change the RabbitMQ `ExternalSecret` body.

- [ ] **Step 4: Обновить порядок применения RabbitMQ**

In `deploy/k8s/platform/rabbitmq/README.md`, add this command before applying the RabbitMQ ExternalSecret:

```bash
kubectl apply -f deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml
kubectl wait --for=condition=Ready clustersecretstore/cluster-secret-store --timeout=2m
```

Change the RabbitMQ teardown text so it deletes only `deploy/k8s/platform/rabbitmq/00-external-secret.yaml` and explicitly retains the shared `ClusterSecretStore` while Valkey or any other ExternalSecret uses it.

- [ ] **Step 5: Проверить единственное определение и корректность YAML**

Run:

```bash
test "$(rg -l '^kind: ClusterSecretStore$' deploy/k8s | wc -l | tr -d ' ')" = "1"
ruby -e 'require "yaml"; ARGV.each { |f| YAML.load_stream(File.read(f)) }' \
  deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml \
  deploy/k8s/platform/rabbitmq/00-external-secret.yaml
git diff --check
```

Expected: all commands exit with code `0`; the only definition is in `platform/external-secrets`.

- [ ] **Step 6: Commit**

```bash
git add \
  deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml \
  deploy/k8s/platform/rabbitmq/00-external-secret.yaml \
  deploy/k8s/platform/rabbitmq/README.md
git commit -m "refactor: share external secrets store"
```

---

### Task 2: Добавить Secret, параметры подключения, CA и NetworkPolicy Valkey

**Files:**
- Create: `deploy/k8s/platform/valkey/00-external-secret.yaml`
- Create: `deploy/k8s/platform/valkey/10-connection-config.yaml`
- Create: `deploy/k8s/platform/valkey/20-network-policy.yaml`
- Create: `deploy/k8s/platform/valkey/yandex-internal-root-ca.pem`

**Interfaces:**
- Consumes: `ClusterSecretStore/cluster-secret-store`, Lockbox secret `e6qa73r663u29svikf0m`, namespace `production` and existing DNS-egress policy.
- Produces: Secret `production/lab-valkey-credentials`, ConfigMap `production/lab-valkey-connection`, ConfigMap `production/yandex-internal-root-ca`, NetworkPolicy `production/allow-valkey-egress`.

- [ ] **Step 1: Сохранить официальный CA bundle Yandex Cloud**

Use the exact two-certificate PEM bundle from `https://storage.yandexcloud.net/cloud-certs/CA.pem` as `deploy/k8s/platform/valkey/yandex-internal-root-ca.pem`. Verify the downloaded source before adding it with `apply_patch`:

```bash
curl --fail --silent --show-error https://storage.yandexcloud.net/cloud-certs/CA.pem | shasum -a 256
```

Expected on `2026-08-10`:

```text
6d148f85b5213445b23ad22ff45e47e1aa2be968f183f9bd6ff39de54d47a8ef  -
```

After adding the file, verify:

```bash
test "$(rg -c 'BEGIN CERTIFICATE' deploy/k8s/platform/valkey/yandex-internal-root-ca.pem)" = "2"
openssl crl2pkcs7 -nocrl \
  -certfile deploy/k8s/platform/valkey/yandex-internal-root-ca.pem \
  | openssl pkcs7 -print_certs -noout
```

Expected subjects include `CN=YandexCLCA` and `CN=YandexInternalRootCA`. If the official bundle checksum changes after `2026-08-10`, inspect both certificates and update the committed bundle and README provenance instead of accepting the change silently.

- [ ] **Step 2: Создать ExternalSecret Valkey**

Create `deploy/k8s/platform/valkey/00-external-secret.yaml`:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: lab-valkey-credentials
  namespace: production
  labels:
    app.kubernetes.io/part-of: restaurant-booking
    app.kubernetes.io/component: cache
    environment: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: cluster-secret-store
    kind: ClusterSecretStore
  target:
    name: lab-valkey-credentials
    creationPolicy: Owner
    template:
      type: Opaque
      engineVersion: v2
      metadata:
        labels:
          app.kubernetes.io/part-of: restaurant-booking
          app.kubernetes.io/component: cache
          environment: production
  data:
    - secretKey: password
      remoteRef:
        key: e6qa73r663u29svikf0m
        property: redis_password
```

- [ ] **Step 3: Создать ConfigMap параметров и CA**

Create `deploy/k8s/platform/valkey/10-connection-config.yaml` with two YAML documents. The first document must be exactly:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: lab-valkey-connection
  namespace: production
  labels:
    app.kubernetes.io/part-of: restaurant-booking
    app.kubernetes.io/component: cache
    environment: production
data:
  host: c-c9q09s84iqj5vgc6gkte.rw.mdb.yandexcloud.net
  port: "6380"
  username: default
  tls: "true"
```

The second document must be a ConfigMap with the exact contents of `yandex-internal-root-ca.pem` under `data.ca.crt`:

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: yandex-internal-root-ca
  namespace: production
  labels:
    app.kubernetes.io/part-of: restaurant-booking
    app.kubernetes.io/component: trust-store
    environment: production
data:
  ca.crt: |
    -----BEGIN CERTIFICATE-----
    MIIE3TCCAsWgAwIBAgIKPxb5sAAAAAAAFzANBgkqhkiG9w0BAQ0FADAfMR0wGwYD
    VQQDExRZYW5kZXhJbnRlcm5hbFJvb3RDQTAeFw0xNzA2MjAxNjQ0MzdaFw0yNzA2
    MjAxNjU0MzdaMFUxEjAQBgoJkiaJk/IsZAEZFgJydTEWMBQGCgmSJomT8ixkARkW
    BnlhbmRleDESMBAGCgmSJomT8ixkARkWAmxkMRMwEQYDVQQDEwpZYW5kZXhDTENB
    MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAqgNnjk0JKPcbsk1+KG2t
    eM1AfMnEe5RkAJuBBuwVV49snhcvO1jhKBx/pCnjr6biICc1/oAFDVgU8yVYYPwp
    WZ2vH3ZtscjJ/RAT/NS9OKKG7kKknhFhVYxua5xhoIQmm6usBNYYiTcWoFm1eHC8
    I9oddOLSscZYbh3unVRvt+3V+drVmUx9oSUKpqMgfysiv1MN6zB3vq9TFkbhz53E
    k0tEcV+W2NnDaeFhLKy284FDKLvOdTDj1EDsSAihxl7sNEKpupNuhgyy2siOqUb+
    d5mO/CRfaAKGg3E6hDM3pEi48E506dJdjPXWfHKSvuguMLRlb2RWdVocRZuyWxOh
    0QIDAQABo4HkMIHhMBAGCSsGAQQBgjcVAQQDAgEAMB0GA1UdDgQWBBRMU5uItjx+
    TOicX1+ovC1Xq2PSnzAZBgkrBgEEAYI3FAIEDB4KAFMAdQBiAEMAQTALBgNVHQ8E
    BAMCAYYwDwYDVR0TAQH/BAUwAwEB/zAfBgNVHSMEGDAWgBSrucX/oe/mUx0zOSKE
    0XbUN04tajBUBgNVHR8ETTBLMEmgR6BFhkNodHRwOi8vY3Jscy55YW5kZXgucnUv
    WWFuZGV4SW50ZXJuYWxSb290Q0EvWWFuZGV4SW50ZXJuYWxSb290Q0EuY3JsMA0G
    CSqGSIb3DQEBDQUAA4ICAQAsR5Lb4Pv2FD0Kk+4oc1GEOnehxKLsQtdV81nrU+IV
    l9pr2oNMdi8lwIolvHZRllLM4Ba5AcRH6YJ5fe7AjKm+5EdSkhqVWo2UOllRCbtS
    wmL50+erOAkxstSlRkO6b8x1L0MOBKv54E5YcQ/Wwt27ldSb6RkEmJBGvmxObAaf
    5zc51pqSqao9tnldYaCblEQ/Zmy43FliIpa2eUJoh8DqK8bVo2gcI3wbQ32tWs9u
    wvKk8fo4lAdhCwhv+QHuqau1VAY9hPU106bsFIDUmijTMxjAobKBi6CkIX6EbNHU
    Jv4DzYVLlDd2y0CADdn2F6I70xpCBn5cquSGuvFbqZjQDmIHwb7WQSxadkiGRWfc
    zVTnmiHjJONJJIpE2t+FOV3hc+8o98OzOtNaH2QQ9j6dnKvtIGKGFeNSDp0vXPOi
    QhHiIyuB7eWx+g2whktQ74UCpGDSXYnEW3s8w5wezVWIEmouq7q4rCEkTNvJ7Ico
    43AgUdPzAFS2zYktw1C+cbUALM8smvXbXrXOBzMmscjIhtXvLMrpPeh23VfdJfQB
    0rN2BmRCLUE8JOV+o0k98XMm83oN+lGkL1l+hyoj3ok1uI3JrsWOcDyjOds3ptcN
    KimJLm27ndjcxDNo/iA6gefMJuCxFRaqI+eF4P0jSkMgnnQqZkvLGFuHCw8eRDhm
    bw==
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    MIIFGTCCAwGgAwIBAgIQJMM7ZIy2SYxCBgK7WcFwnjANBgkqhkiG9w0BAQ0FADAf
    MR0wGwYDVQQDExRZYW5kZXhJbnRlcm5hbFJvb3RDQTAeFw0xMzAyMTExMzQxNDNa
    Fw0zMzAyMTExMzUxNDJaMB8xHTAbBgNVBAMTFFlhbmRleEludGVybmFsUm9vdENB
    MIICIjANBgkqhkiG9w0BAQEFAAOCAg8AMIICCgKCAgEAgb4xoQjBQ7oEFk8EHVGy
    1pDEmPWw0Wgw5nX9RM7LL2xQWyUuEq+Lf9Dgh+O725aZ9+SO2oEs47DHHt81/fne
    5N6xOftRrCpy8hGtUR/A3bvjnQgjs+zdXvcO9cTuuzzPTFSts/iZATZsAruiepMx
    SGj9S1fGwvYws/yiXWNoNBz4Tu1Tlp0g+5fp/ADjnxc6DqNk6w01mJRDbx+6rlBO
    aIH2tQmJXDVoFdrhmBK9qOfjxWlIYGy83TnrvdXwi5mKTMtpEREMgyNLX75UjpvO
    NkZgBvEXPQq+g91wBGsWIE2sYlguXiBniQgAJOyRuSdTxcJoG8tZkLDPRi5RouWY
    gxXr13edn1TRDGco2hkdtSUBlajBMSvAq+H0hkslzWD/R+BXkn9dh0/DFnxVt4XU
    5JbFyd/sKV/rF4Vygfw9ssh1ZIWdqkfZ2QXOZ2gH4AEeoN/9vEfUPwqPVzL0XEZK
    r4s2WjU9mE5tHrVsQOZ80wnvYHYi2JHbl0hr5ghs4RIyJwx6LEEnj2tzMFec4f7o
    dQeSsZpgRJmpvpAfRTxhIRjZBrKxnMytedAkUPguBQwjVCn7+EaKiJfpu42JG8Mm
    +/dHi+Q9Tc+0tX5pKOIpQMlMxMHw8MfPmUjC3AAd9lsmCtuybYoeN2IRdbzzchJ8
    l1ZuoI3gH7pcIeElfVSqSBkCAwEAAaNRME8wCwYDVR0PBAQDAgGGMA8GA1UdEwEB
    /wQFMAMBAf8wHQYDVR0OBBYEFKu5xf+h7+ZTHTM5IoTRdtQ3Ti1qMBAGCSsGAQQB
    gjcVAQQDAgEAMA0GCSqGSIb3DQEBDQUAA4ICAQAVpyJ1qLjqRLC34F1UXkC3vxpO
    nV6WgzpzA+DUNog4Y6RhTnh0Bsir+I+FTl0zFCm7JpT/3NP9VjfEitMkHehmHhQK
    c7cIBZSF62K477OTvLz+9ku2O/bGTtYv9fAvR4BmzFfyPDoAKOjJSghD1p/7El+1
    eSjvcUBzLnBUtxO/iYXRNo7B3+1qo4F5Hz7rPRLI0UWW/0UAfVCO2fFtyF6C1iEY
    /q0Ldbf3YIaMkf2WgGhnX9yH/8OiIij2r0LVNHS811apyycjep8y/NkG4q1Z9jEi
    VEX3P6NEL8dWtXQlvlNGMcfDT3lmB+tS32CPEUwce/Ble646rukbERRwFfxXojpf
    C6ium+LtJc7qnK6ygnYF4D6mz4H+3WaxJd1S1hGQxOb/3WVw63tZFnN62F6/nc5g
    6T44Yb7ND6y3nVcygLpbQsws6HsjX65CoSjrrPn0YhKxNBscF7M7tLTW/5LK9uhk
    yjRCkJ0YagpeLxfV1l1ZJZaTPZvY9+ylHnWHhzlq0FzcrooSSsp4i44DB2K7O2ID
    87leymZkKUY6PMDa4GkDJx0dG4UXDhRETMf+NkYgtLJ+UIzMNskwVDcxO4kVL+Hi
    Pj78bnC5yCw8P5YylR45LdxLzLO68unoXOyFz1etGXzszw8lJI9LNubYxk77mK8H
    LpuQKbSbIERsmR+QqQ==
    -----END CERTIFICATE-----
```

Confirm byte-equivalence after parsing with this exact command:

```bash
ruby -ryaml -e '
docs = YAML.load_stream(File.read("deploy/k8s/platform/valkey/10-connection-config.yaml"))
embedded = docs.fetch(1).fetch("data").fetch("ca.crt")
source = File.read("deploy/k8s/platform/valkey/yandex-internal-root-ca.pem")
abort "embedded CA differs" unless embedded == source
'
```

Expected: exit code `0` and no output.

- [ ] **Step 4: Создать минимальную egress NetworkPolicy**

Create `deploy/k8s/platform/valkey/20-network-policy.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-valkey-egress
  namespace: production
  labels:
    app.kubernetes.io/part-of: restaurant-booking
    app.kubernetes.io/component: cache
    environment: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - ipBlock:
            cidr: 10.20.1.0/24
      ports:
        - protocol: TCP
          port: 6380
```

- [ ] **Step 5: Выполнить локальные проверки безопасности и структуры**

Run:

```bash
ruby -e 'require "yaml"; ARGV.each { |f| YAML.load_stream(File.read(f)) }' \
  deploy/k8s/platform/valkey/00-external-secret.yaml \
  deploy/k8s/platform/valkey/10-connection-config.yaml \
  deploy/k8s/platform/valkey/20-network-policy.yaml
rg -n 'e6qa73r663u29svikf0m|redis_password|c-c9q09s84iqj5vgc6gkte|10\.20\.1\.0/24|6380' \
  deploy/k8s/platform/valkey
test "$(rg -c 'BEGIN CERTIFICATE' deploy/k8s/platform/valkey/yandex-internal-root-ca.pem)" = "2"
if rg -n '(?i)(redis_password\s*[:=]\s*[^r]|password\s*:\s*[[:alnum:]]{12,}|private key|authorized_key)' deploy/k8s/platform/valkey; then
  echo 'possible secret material found'
  exit 1
fi
git diff --check
```

Expected: YAML parsing and assertions exit `0`; `rg` finds only IDs, property names and connection metadata, never an actual password or authorized key.

- [ ] **Step 6: Commit**

```bash
git add \
  deploy/k8s/platform/valkey/00-external-secret.yaml \
  deploy/k8s/platform/valkey/10-connection-config.yaml \
  deploy/k8s/platform/valkey/20-network-policy.yaml \
  deploy/k8s/platform/valkey/yandex-internal-root-ca.pem
git commit -m "feat: add managed valkey connection resources"
```

---

### Task 3: Добавить ручную инструкцию применения и безопасной проверки

**Files:**
- Create: `deploy/k8s/platform/valkey/README.md`

**Interfaces:**
- Consumes: все ресурсы из Tasks 1–2 и ручной доступ пользователя к Yandex Cloud UI и `kubectl`.
- Produces: последовательность UI-проверок, server-side dry-run, применение, TLS smoke test, диагностика и удаление без показа пароля.

- [ ] **Step 1: Описать две UI-предпосылки**

Create `deploy/k8s/platform/valkey/README.md` in Russian. Start with the exact resource inventory and instruct the user to verify:

1. `Lockbox → e6qa73r663u29svikf0m → Права доступа`: service account `lab-external-secrets` has `lockbox.payloadViewer` on this secret.
2. `Virtual Private Cloud → Группы безопасности → lab-valkey`: inbound TCP `6380`, source security group `lab-k8s-nodes`.

Explicitly state that public access, the Cloud Shell IP, and the user's home IP are not required.

- [ ] **Step 2: Описать dry-run и порядок применения**

Include these commands exactly:

```bash
kubectl apply --dry-run=server \
  -f deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml

kubectl apply --dry-run=server \
  -f deploy/k8s/platform/valkey/00-external-secret.yaml \
  -f deploy/k8s/platform/valkey/10-connection-config.yaml \
  -f deploy/k8s/platform/valkey/20-network-policy.yaml

kubectl apply \
  -f deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml

kubectl wait \
  --for=condition=Ready \
  clustersecretstore/cluster-secret-store \
  --timeout=2m

kubectl apply \
  -f deploy/k8s/platform/valkey/00-external-secret.yaml

kubectl wait \
  --for=condition=Ready \
  externalsecret/lab-valkey-credentials \
  -n production \
  --timeout=2m

kubectl apply \
  -f deploy/k8s/platform/valkey/10-connection-config.yaml \
  -f deploy/k8s/platform/valkey/20-network-policy.yaml
```

Then verify metadata only:

```bash
kubectl get clustersecretstore cluster-secret-store
kubectl get externalsecret lab-valkey-credentials -n production
kubectl get secret lab-valkey-credentials -n production
kubectl get configmap lab-valkey-connection yandex-internal-root-ca -n production
kubectl get networkpolicy allow-valkey-egress -n production
```

Warn not to add `-o yaml`, `-o json` or a password jsonpath to the Secret command.

- [ ] **Step 3: Добавить временный TLS smoke test**

Add an inline Pod manifest named `lab-valkey-smoke-test`. It must:

- use namespace `production`;
- set `automountServiceAccountToken: false` and `restartPolicy: Never`;
- run as UID/GID `999`, non-root, with `RuntimeDefault`, dropped capabilities, no privilege escalation and read-only root filesystem;
- use image `redis:7.4.2-alpine` only as a temporary client;
- read `REDISCLI_AUTH` from Secret key `lab-valkey-credentials/password`;
- read host, port and username from ConfigMap `lab-valkey-connection`;
- mount `yandex-internal-root-ca/ca.crt` read-only at `/etc/yandex-ca/ca.crt`;
- request `25m` CPU and `32Mi` memory and limit `100m` CPU and `128Mi` memory;
- execute `sleep 3600` until the user finishes the test.

Use this exact manifest in the README command block:

```yaml
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
```

After creating the Pod, include:

```bash
kubectl wait \
  --for=condition=Ready \
  pod/lab-valkey-smoke-test \
  -n production \
  --timeout=3m

kubectl exec -n production lab-valkey-smoke-test -- sh -c \
  'redis-cli --host "$VALKEY_HOST" --port "$VALKEY_PORT" --user "$VALKEY_USERNAME" --tls --cacert /etc/yandex-ca/ca.crt PING'

kubectl exec -n production lab-valkey-smoke-test -- sh -c \
  'redis-cli --host "$VALKEY_HOST" --port "$VALKEY_PORT" --user "$VALKEY_USERNAME" --tls --cacert /etc/yandex-ca/ca.crt SET restaurant-booking:smoke-test ok EX 60'

kubectl exec -n production lab-valkey-smoke-test -- sh -c \
  'redis-cli --host "$VALKEY_HOST" --port "$VALKEY_PORT" --user "$VALKEY_USERNAME" --tls --cacert /etc/yandex-ca/ca.crt GET restaurant-booking:smoke-test'

kubectl exec -n production lab-valkey-smoke-test -- sh -c \
  'redis-cli --host "$VALKEY_HOST" --port "$VALKEY_PORT" --user "$VALKEY_USERNAME" --tls --cacert /etc/yandex-ca/ca.crt DEL restaurant-booking:smoke-test'

kubectl delete pod lab-valkey-smoke-test -n production
```

Expected output in order: `PONG`, `OK`, `ok`, integer `1`. Explain that `REDISCLI_AUTH` prevents the password from appearing in the process arguments or shell history.

- [ ] **Step 4: Добавить диагностику и удаление**

Document:

- `ExternalSecret Ready=False`: inspect `kubectl describe externalsecret lab-valkey-credentials -n production`, IAM role and exact property `redis_password` without printing Secret payload.
- timeout: inspect NetworkPolicy and the `lab-valkey` SG rule from `lab-k8s-nodes`.
- TLS failure: verify FQDN, `6380`, CA mount and Pod time; never add `--insecure`.
- DNS failure: inspect CoreDNS and existing `allow-dns-egress`.

Include teardown in this order:

```bash
kubectl delete pod lab-valkey-smoke-test -n production --ignore-not-found
kubectl delete -f deploy/k8s/platform/valkey/20-network-policy.yaml
kubectl delete -f deploy/k8s/platform/valkey/10-connection-config.yaml
kubectl delete -f deploy/k8s/platform/valkey/00-external-secret.yaml
```

State that the shared `ClusterSecretStore` stays because RabbitMQ uses it, and that Kubernetes resource deletion does not stop or delete Managed Valkey or its Lockbox secret.

- [ ] **Step 5: Проверить инструкцию и весь набор файлов**

Run:

```bash
rg -n 'kubectl apply|kubectl wait|PING|smoke-test|--tls|--cacert|kubectl delete' \
  deploy/k8s/platform/valkey/README.md
rg -n -- '--insecure|6379|LoadBalancer|NodePort' deploy/k8s/platform/valkey
ruby -e 'require "yaml"; ARGV.each { |f| YAML.load_stream(File.read(f)) }' \
  deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml \
  deploy/k8s/platform/rabbitmq/00-external-secret.yaml \
  deploy/k8s/platform/valkey/00-external-secret.yaml \
  deploy/k8s/platform/valkey/10-connection-config.yaml \
  deploy/k8s/platform/valkey/20-network-policy.yaml
test "$(rg -l '^kind: ClusterSecretStore$' deploy/k8s | wc -l | tr -d ' ')" = "1"
git diff --check
git status --short
```

Expected: required instructions are present; `6379` and public exposure kinds appear only in explanatory warnings, all YAML parses, one ClusterSecretStore exists and `git diff --check` is clean.

- [ ] **Step 6: Commit**

```bash
git add deploy/k8s/platform/valkey/README.md
git commit -m "docs: add managed valkey operating guide"
```

---

### Task 4: Финальная локальная проверка и ручная передача

**Files:**
- Verify only: `deploy/k8s/platform/external-secrets/00-cluster-secret-store.yaml`
- Verify only: `deploy/k8s/platform/rabbitmq/00-external-secret.yaml`
- Verify only: `deploy/k8s/platform/rabbitmq/README.md`
- Verify only: `deploy/k8s/platform/valkey/*`

**Interfaces:**
- Consumes: завершённые Tasks 1–3.
- Produces: проверенный локальный набор файлов и короткий порядок действий для пользователя.

- [ ] **Step 1: Выполнить итоговую проверку без обращения к кластеру**

Run all commands from Task 3 Step 5 plus:

```bash
git grep -n 'e6qa73r663u29svikf0m'
git grep -n 'c-c9q09s84iqj5vgc6gkte.rw.mdb.yandexcloud.net'
git grep -n '10.20.1.0/24'
openssl verify \
  -CAfile deploy/k8s/platform/valkey/yandex-internal-root-ca.pem \
  deploy/k8s/platform/valkey/yandex-internal-root-ca.pem || true
git log -4 --oneline
```

For the CA bundle, do not treat a self-verification failure of the first intermediate certificate as a file failure. The authoritative checks are the two-certificate count, parsed subjects, official-source checksum and byte equality between the PEM file and ConfigMap.

- [ ] **Step 2: Передать пользователю только ручные действия**

Report:

1. which files were generated;
2. that no cloud or Kubernetes commands were executed;
3. the two Yandex Cloud UI checks;
4. the exact `kubectl` application order from the README;
5. the expected smoke-test outputs;
6. the next platform topic after Valkey: delivery of the first production Deployment and Service, then ALB/TLS, then observability.

Do not claim that Valkey connectivity works until the user runs the smoke test and provides the result.
