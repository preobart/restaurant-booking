# Базовая конфигурация Kubernetes

В этой директории находится базовая конфигурация пространств имён и политик для учебного кластера `lab-k8s`.

## Почему используется Cloud Shell

Консоль управления Yandex Cloud позволяет создавать отдельные типы объектов Kubernetes, но не предоставляет универсального редактора для всех объектов из этого набора. В частности, для точного применения `ResourceQuota`, `LimitRange`, `NetworkPolicy` и меток Pod Security удобнее использовать Cloud Shell.

Cloud Shell работает прямо в браузере. Устанавливать `yc`, `kubectl`, Helm или OpenTofu на локальный компьютер для этого этапа не требуется.

## Подготовка

1. Запустите `lab-k8s` и дождитесь состояний `Running` и `Healthy`.
2. Убедитесь, что все три рабочих узла находятся в состоянии `Ready`.
3. В левом меню консоли Yandex Cloud откройте **Cloud Shell**.
4. Узнайте публичный IPv4-адрес текущей сессии Cloud Shell:

   ```bash
   curl -4 https://ifconfig.me
   ```

5. В группе безопасности `lab-k8s-master` временно разрешите входящий TCP-трафик с полученного адреса `/32` на порты `443` и `6443`.

После завершения работы удалите эти два временных правила.

## Подключение к кластеру

В Cloud Shell выполните:

```bash
yc config set folder-id b1gtc4a64q6o5m7neg55
yc managed-kubernetes cluster get-credentials lab-k8s --external
kubectl cluster-info
kubectl get nodes
```

Перед продолжением должны отображаться три узла со статусом `Ready`.

## Применение файлов

Применяйте файлы строго в указанном порядке:

1. `00-namespaces.yaml` — пространства имён и Pod Security labels.
2. `10-resource-policies.yaml` — квоты и стандартные requests/limits.
3. `20-network-policies.yaml` — запрет трафика по умолчанию и разрешение DNS.

Чтобы применить файл без его загрузки в Cloud Shell:

1. Выполните команду:

   ```bash
   kubectl apply -f -
   ```

2. Вставьте полное содержимое нужного YAML-файла.
3. Нажмите `Ctrl+D`.
4. Проверьте результат перед переходом к следующему файлу.

## Проверка

```bash
kubectl get namespaces platform staging production --show-labels
kubectl get resourcequota,limitrange -A
kubectl get networkpolicy -n staging
kubectl get networkpolicy -n production
```

Ожидаемый результат:

- пространства имён `platform`, `staging` и `production` имеют статус `Active`;
- в каждом пространстве имён есть `namespace-resources` и `container-resources`;
- в `staging` и `production` есть политики `default-deny-ingress`, `default-deny-egress` и `allow-dns-egress`;
- прикладных подов на этом этапе нет;
- кластер остаётся в состоянии `Healthy`, а три рабочих узла — в состоянии `Ready`.

Политики default-deny намеренно блокируют прикладной трафик. Разрешающие политики будут добавляться одновременно с соответствующими сервисами.

## Завершение учебной сессии

1. Удалите из `lab-k8s-master` временные правила для IP-адреса Cloud Shell.
2. Остановите кластер `lab-k8s` через консоль Yandex Cloud.

Созданные Kubernetes-объекты сохранятся после повторного запуска кластера.

## Откат, пока пространства имён пустые

1. Удалите шесть объектов `NetworkPolicy`.
2. Удалите три объекта `LimitRange` с именем `container-resources`.
3. Удалите три объекта `ResourceQuota` с именем `namespace-resources`.
4. Удалите `platform`, `staging` и `production`, только если в них нет нужных рабочих нагрузок.

Удаление пространства имён также удаляет все находящиеся в нём объекты. Не используйте удаление пространства имён как обычный способ отката после развёртывания приложений.
