# Gwin и Application Load Balancer для staging

Следующий production-like слой — управляемый L7-вход в Kubernetes:

```text
локальный компьютер
  -> публичный IP Yandex Application Load Balancer
  -> Gwin
  -> NodePort Service
  -> Pod staging
```

Gwin устанавливается из Yandex Cloud Marketplace и управляет ресурсами Application Load Balancer по Kubernetes-манифестам. На первом этапе создаются сервисный аккаунт, группы безопасности и сам контроллер. Балансировщик появится только после применения отдельного Ingress/Gateway-манифеста.

## Зафиксированные параметры стенда

```text
Каталог: b1gtc4a64q6o5m7neg55
Сеть: lab-vpc
Группа узлов: lab-workers
Группа безопасности узлов: lab-k8s-nodes
Подсеть a: lab-subnet-a, 10.20.1.0/24
Подсеть b: lab-subnet-b, 10.20.2.0/24
Подсеть d: lab-subnet-d, 10.20.3.0/24
Namespace Gwin: gwin-space
Имя Marketplace-приложения: gwin
IAM service account: lab-gwin
Kubernetes service account: gwin
IAM service account ID: ajekao68lsfas66foh5k
IAM federation ID: ajeqroqpgpv7cajnnvn3
ALB security group ID: enpdntuqp8q5v422beh1
```

## 1. Создать IAM service account для Gwin

В консоли Yandex Cloud откройте:

```text
Каталог -> Управление доступом -> Сервисные аккаунты -> Создать
```

Укажите имя `lab-gwin`.

На уровне каталога назначьте ему только базовые роли этого этапа:

```text
alb.editor
vpc.publicAdmin
compute.viewer
k8s.viewer
```

Пока не добавляйте `editor` или `admin`. Роли для Certificate Manager, Cloud Logging и Smart Web Security будут добавляться отдельно, когда соответствующий сервис действительно понадобится.

Сохраните идентификатор сервисного аккаунта. Секретный или авторизованный ключ создавать не нужно.

## 2. Создать IAM-федерацию кластера

Опция федерации уже включена в кластере и группе `lab-workers`, но она только включает OIDC-провайдер Kubernetes. Отдельный объект федерации IAM создаётся вручную один раз.

Откройте обзорную страницу кластера `lab-k8s`. В блоке `Управление идентификацией и доступом` скопируйте два несекретных значения:

```text
URL эмитента
URL набора ключей JWKS
```

Обычно они выглядят так:

```text
Issuer: https://storage.yandexcloud.net/mk8s-oidc/v1/clusters/<ID_КЛАСТЕРА>
JWKS:   https://storage.yandexcloud.net/mk8s-oidc/v1/clusters/<ID_КЛАСТЕРА>/jwks.json
```

Затем откройте:

```text
Identity and Access Management
-> Федерации сервисных аккаунтов
-> Создать федерацию
```

Заполните:

```text
Имя: lab-k8s-wlif
Значение Issuer (iss): <URL эмитента из кластера>
Допустимые значения Audience (aud): <тот же URL эмитента>
Адрес JWKS: <URL набора ключей JWKS из кластера>
```

Создайте федерацию и сохраните её ID. Федерация не содержит секретов и не создаёт долгоживущие ключи.

## 3. Привязать lab-gwin к федерации кластера

Откройте сервисный аккаунт `lab-gwin`:

```text
Identity and Access Management -> Сервисные аккаунты
-> lab-gwin -> Привязать к федерации
```

Укажите:

```text
Федерация сервисных аккаунтов: lab-k8s-wlif
Значение Subject (sub): system:serviceaccount:gwin-space:gwin
```

Это разрешает только Kubernetes ServiceAccount `gwin` из namespace `gwin-space` получать короткоживущие IAM-токены от имени `lab-gwin`. Долгоживущий JSON-ключ в кластере не хранится.

## 4. Создать группу безопасности балансировщика

Откройте:

```text
Virtual Private Cloud -> Группы безопасности -> Создать
```

Укажите:

```text
Имя: lab-alb-staging
Сеть: lab-vpc
```

Добавьте входящие правила:

| Назначение | Протокол | Порт | Источник |
|---|---|---:|---|
| Staging HTTP с рабочего компьютера | TCP | 80 | `<ВАШ_ПУБЛИЧНЫЙ_IP>/32` |
| Проверки узлов самого ALB | TCP | 30080 | `Проверки состояния балансировщика` |

На первом этапе не открывайте `443` и не используйте `0.0.0.0/0`: пока нет домена и TLS, staging будет доступен только с вашего текущего публичного IP.

Добавьте исходящие правила к узлам Kubernetes:

| Назначение | Протокол | Порты | Получатель |
|---|---|---:|---|
| Трафик к NodePort | TCP | 30000–32767 | `10.20.1.0/24`, `10.20.2.0/24`, `10.20.3.0/24` |
| Проверки Kubernetes-бэкендов | TCP | 10501 | `10.20.1.0/24`, `10.20.2.0/24`, `10.20.3.0/24` |

Сохраните идентификатор `lab-alb-staging`.

## 5. Дополнить группу безопасности узлов

Откройте существующую группу `lab-k8s-nodes` и добавьте входящие правила:

| Назначение | Протокол | Порты | Источник |
|---|---|---:|---|
| Трафик ALB к NodePort | TCP | 30000–32767 | группа безопасности `lab-alb-staging` |
| Проверки Kubernetes-бэкендов | TCP | 10501 | группа безопасности `lab-alb-staging` |

Если такие правила уже есть с источником `0.0.0.0/0`, не создавайте дубликаты. После проверки ALB широкое правило можно заменить источником `lab-alb-staging`. Ссылка на группу здесь безопаснее CIDR: узлы будут принимать NodePort-трафик только от сетевых интерфейсов, которым назначена группа балансировщика.

Не удаляйте остальные правила `lab-k8s-nodes`: они нужны мастеру, Pod, Service и исходящему доступу через NAT.

## 6. Установить Gwin через Marketplace

Откройте:

```text
Managed Service for Kubernetes -> lab-k8s
-> Marketplace -> Gwin -> Перейти к установке
```

Укажите:

```text
Пространство имен: создать новое gwin-space
Название приложения: gwin
Идентификатор каталога: b1gtc4a64q6o5m7neg55
Способ аутентификации: федерация сервисных аккаунтов
Сервисный аккаунт, привязанный к WLIF: lab-gwin
```

Нажмите `Установить` и дождитесь состояния `Deployed`.

## 7. Проверить контроллер без создания ALB

```bash
kubectl get pods -n gwin-space -o wide

kubectl get serviceaccount gwin -n gwin-space

kubectl get ingressclass

kubectl get crd | grep gwin.yandex.cloud
```

Ожидается:

- все Pod Gwin имеют состояние `Running` и `READY`;
- существует ServiceAccount `gwin`;
- существует IngressClass `gwin-default`;
- CRD Gwin зарегистрированы;
- в Application Load Balancer ещё нет нового балансировщика, потому что Ingress/Gateway пока не создавался.

Если Pod не запускается, до создания Ingress сохраните диагностику:

```bash
kubectl get pods -n gwin-space -o wide
kubectl get events -n gwin-space --sort-by=.lastTimestamp
kubectl logs -n gwin-space -l app.kubernetes.io/name=gwin --all-containers --tail=200
```

## 8. Подготовленные Kubernetes-ресурсы

После успешной установки Gwin используются файлы:

```text
deploy/k8s/apps/auth-service/staging/60-service.yaml
deploy/k8s/apps/catalog-service/staging/60-service.yaml
deploy/k8s/apps/booking-service/staging/60-service.yaml
deploy/k8s/ingress/gwin/10-staging-network-policies.yaml
deploy/k8s/ingress/gwin/20-staging-ingress.yaml
```

Три Service меняются с `ClusterIP` на `NodePort`:

```text
auth-service:    8081 -> 30081
catalog-service: 8082 -> 30082
booking-service: 8083 -> 30083
```

NodePort не делает Pod публичными сам по себе: у узлов нет внешних IP, а группа `lab-k8s-nodes` принимает эти порты только от `lab-alb-staging`.

Отдельные NetworkPolicy разрешают трафик к API Pod только из трёх подсетей `lab-vpc` и только на порт конкретного сервиса.

Ingress создаёт один динамический публичный IP и маршрутизирует:

```text
/api/auth        -> auth-service:8081
/api/restaurants -> catalog-service:8082
/api/bookings    -> booking-service:8083
```

Аннотация подсетей намеренно отсутствует. По умолчанию Gwin использует подсети Managed Kubernetes-кластера, поэтому их ID в первом манифесте не требуются.

## 9. Проверить манифесты на API-сервере

Эти команды ничего не создают и не изменяют:

```bash
kubectl apply --dry-run=server \
  -f deploy/k8s/apps/auth-service/staging/60-service.yaml \
  -f deploy/k8s/apps/catalog-service/staging/60-service.yaml \
  -f deploy/k8s/apps/booking-service/staging/60-service.yaml \
  -f deploy/k8s/ingress/gwin/10-staging-network-policies.yaml \
  -f deploy/k8s/ingress/gwin/20-staging-ingress.yaml
```

Dry-run проверит в том числе, свободны ли NodePort `30081`, `30082` и `30083`.

## 10. Открыть бэкенды только для ALB

Сначала примените Service и NetworkPolicy без Ingress:

```bash
kubectl apply \
  -f deploy/k8s/apps/auth-service/staging/60-service.yaml \
  -f deploy/k8s/apps/catalog-service/staging/60-service.yaml \
  -f deploy/k8s/apps/booking-service/staging/60-service.yaml \
  -f deploy/k8s/ingress/gwin/10-staging-network-policies.yaml
```

Проверьте:

```bash
kubectl get service \
  auth-service catalog-service booking-service \
  -n staging \
  -o wide

kubectl get networkpolicy \
  -n staging \
  -l app.kubernetes.io/component=ingress
```

Ожидаются `TYPE=NodePort` и порты `8081:30081`, `8082:30082`, `8083:30083`. Поле `EXTERNAL-IP` остаётся пустым.

## 11. Создать Application Load Balancer

Применение следующего файла запускает создание оплачиваемого ALB и динамического публичного IP:

```bash
kubectl apply \
  -f deploy/k8s/ingress/gwin/20-staging-ingress.yaml
```

Следите за состоянием:

```bash
kubectl get ingress restaurant-booking-staging \
  -n staging \
  --watch
```

Создание может занимать несколько минут. Завершите `watch` сочетанием `Ctrl+C`, когда в `ADDRESS` появится IPv4-адрес.

После этого выполните:

```bash
kubectl describe ingress restaurant-booking-staging -n staging

kubectl get events \
  -n staging \
  --sort-by=.lastTimestamp
```

В UI Yandex Cloud откройте `Application Load Balancer`. Должен появиться один балансировщик, созданный Gwin, с группой безопасности `lab-alb-staging`. Не редактируйте его или дочерние ресурсы вручную: источником конфигурации является Kubernetes Ingress.

## 12. Проверить HTTP-вход

Возьмите значение `ADDRESS`:

```bash
kubectl get ingress restaurant-booking-staging -n staging
```

С локального компьютера проверьте маршрут Catalog:

```bash
curl -i http://<ADDRESS>/api/restaurants/
```

Ответ приложения с любым ожидаемым HTTP-кодом подтверждает маршрутизацию. `502` или `503` означает проблему между ALB и Kubernetes-бэкендом; в этом случае сохраните диагностику, не меняя ALB вручную:

```bash
kubectl describe ingress restaurant-booking-staging -n staging
kubectl get service -n staging
kubectl get endpointslice -n staging
kubectl get events -n staging --sort-by=.lastTimestamp
kubectl logs -n gwin-space deployment/gwin --tail=200
```

## Что сообщить для следующего шага

После успешного создания HTTP-входа пришлите:

```text
ADDRESS созданного Ingress
вывод kubectl describe ingress restaurant-booking-staging -n staging
результат curl к /api/restaurants/
```

После проверки HTTP будут добавлены статический публичный IP, Certificate Manager, DNS, HTTPS и автоматический redirect с HTTP на HTTPS.

## Как остановить этот слой

До применения Ingress/Gateway Gwin не создаёт балансировщик. Для временной остановки уже созданного ALB сначала удаляется управляющий Kubernetes-ресурс, а не сам балансировщик в консоли Yandex Cloud:

```bash
kubectl delete ingress restaurant-booking-staging -n staging --ignore-not-found
```

После этого нужно дождаться, пока Gwin удалит ALB и его дочерние ресурсы. Сам Gwin — бесплатное Marketplace-приложение, его Pod используют небольшую часть уже оплачиваемых ресурсов узлов.
