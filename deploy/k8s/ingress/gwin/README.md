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

## 2. Привязать lab-gwin к существующей федерации кластера

Кластер и группа `lab-workers` уже поддерживают федерацию сервисных аккаунтов. Откройте существующую федерацию этого Kubernetes-кластера:

```text
Каталог -> Управление доступом -> Федерации сервисных аккаунтов
-> федерация кластера -> Привязки -> Создать привязку
```

Укажите:

```text
Сервисный аккаунт: lab-gwin
Внешний субъект: system:serviceaccount:gwin-space:gwin
```

Это разрешает только Kubernetes ServiceAccount `gwin` из namespace `gwin-space` получать короткоживущие IAM-токены от имени `lab-gwin`. Долгоживущий JSON-ключ в кластере не хранится.

## 3. Создать группу безопасности балансировщика

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

## 4. Дополнить группу безопасности узлов

Откройте существующую группу `lab-k8s-nodes` и добавьте входящие правила:

| Назначение | Протокол | Порты | Источник |
|---|---|---:|---|
| Трафик ALB к NodePort | TCP | 30000–32767 | `10.20.1.0/24`, `10.20.2.0/24`, `10.20.3.0/24` |
| Проверки Kubernetes-бэкендов | TCP | 10501 | `10.20.1.0/24`, `10.20.2.0/24`, `10.20.3.0/24` |

Если такие правила уже есть с источником `0.0.0.0/0`, не создавайте дубликаты. После проверки ALB широкое правило можно заменить перечисленными CIDR подсетей.

Не удаляйте остальные правила `lab-k8s-nodes`: они нужны мастеру, Pod, Service и исходящему доступу через NAT.

## 5. Установить Gwin через Marketplace

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

## 6. Проверить контроллер без создания ALB

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

## Что сообщить для следующего шага

После успешной установки понадобятся только несекретные идентификаторы:

```text
ID IAM service account lab-gwin
ID группы безопасности lab-alb-staging
ID lab-subnet-a
ID lab-subnet-b
ID lab-subnet-d
```

На их основе будет сгенерирован готовый манифест входа с маршрутами:

```text
/api/auth        -> auth-service:8081
/api/restaurants -> catalog-service:8082
/api/bookings    -> booking-service:8083
```

Сначала вход будет HTTP и ограничен вашим `/32`. После проверки добавятся Certificate Manager, DNS, HTTPS и автоматический redirect с HTTP на HTTPS.

## Как остановить этот слой

До применения Ingress/Gateway Gwin не создаёт балансировщик. Для временной остановки уже созданного ALB сначала удаляется управляющий Kubernetes-ресурс, а не сам балансировщик в консоли Yandex Cloud:

```bash
kubectl delete ingress restaurant-booking-staging -n staging --ignore-not-found
```

После этого нужно дождаться, пока Gwin удалит ALB и его дочерние ресурсы. Сам Gwin — бесплатное Marketplace-приложение, его Pod используют небольшую часть уже оплачиваемых ресурсов узлов.
