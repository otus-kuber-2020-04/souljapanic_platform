# souljapanic_platform
souljapanic Platform repository

[![Build Status](https://travis-ci.com/otus-kuber-2020-04/souljapanic_platform.svg?branch=master)](https://travis-ci.com/otus-kuber-2020-04/souljapanic_platform)

# kubernetes-intro

## Настройка рабочего окружения:

* Установка minikube с driver virtualbox:

```
minikube start --driver=virtualbox
```

* Проверка статуса minikube:

```
minikube status
```

* Подключение к виртуальной машине minikube:

```
minikube ssh
```

* Проверка статуса k8s:

```
kubectl cluster-info
kubectl get cs
```

* Проверка статуса POD'ов проекта kube-system:

```
kubectl get pods -n kube-system
```

## Проверка отказоустойчивости:

Для проверки отказоустойчивости удалим все контейнере на виртуальной машине и проверим? будет выполнен их повторный запуск или нет.

Выполняем следующие команды:

* minikube ssh

* docker rm -f $(docker ps -a -q)

Выполняем проверку:

* kubectl get pods -n kube-system

Наблюдаем, что POD'ы запустились повторно. методы запуска POD'ов:

```
kubectl get deployment.apps/coredns -o yaml -n kube-system

Используется ReplicaSet
```

```
kubectl get daemonset.apps/kube-proxy -o yaml -n kube-system
Используется Daemonset
```

```
Все остальные POD'ы используют Static Pods.
Конфигурация хранится на виртуальной машине minikube в диреткории /etc/kubernetes/manifests/
```

## Создание POD'а web:

* Сборка и публикация образа:

```
docker build --no-cache --rm --force-rm -t souljapanic/web:0.3 -f web/Dockerfile ./web/.
docker push souljapanic/web:0.3
```

* kubectl apply -f web-pod.yaml -n default

* kubectl get pods -n default

* Проверка UID:

```
kubectl exec -it pod/web-static -- id
uid=1001(nginx) gid=101(nginx) groups=101(nginx)
```

* Проверка работоспособности контейнера:

```
kubectl port-forward --address 0.0.0.0 pod/web-static 8000:8000 -n default
curl localhost:8000
```

## Создание POD'а frontend:

* kubectl run frontend --image souljapanic/frontend:0.1 --restart=Never

* Создание manifest файла:

```
kubectl run frontend --image souljapanic/frontend:0.1 --restart=Never --dry-run=client -o yaml
```

* Отладка POD'а:

```
kubectl logs frontend -n default
{"message":"Tracing enabled.","severity":"info","timestamp":"2020-05-11T13:12:28.479289246Z"}
{"message":"Profiling enabled.","severity":"info","timestamp":"2020-05-11T13:12:28.479562918Z"}
panic: environment variable "PRODUCT_CATALOG_SERVICE_ADDR" not set

Ошибка связана с отсутствующими переменными
```

* kubectl apply -f frontend-pod-healthy.yaml -n default

* Просмотр переменных:

```
kubectl set env pod/frontend --list
# Pod frontend, container frontend
PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550
CURRENCY_SERVICE_ADDR=currencyservice:7000
CART_SERVICE_ADDR=cartservice:7070
RECOMMENDATION_SERVICE_ADDR=recommendationservice:8080
SHIPPING_SERVICE_ADDR=shippingservice:50051
CHECKOUT_SERVICE_ADDR=checkoutservice:5050
AD_SERVICE_ADDR=adservice:9555
```

# kubernetes-controllers

## Настройка окружения:

* Создание кластера:

```
kind create cluster --config kind-config.yaml
```

* Список кластеров:

```
kind get clusters
```

* Информация о кластере:

```
kubectl cluster-info --context kind-kind
```

## ReplicaSet

* Проверка работы ReplicaSet, matchLabels выполняется по app=frontend:

```
kubectl apply -f frontend-replicaset.yaml -n default
```

* Увеличение количества replica:

```
kubectl scale replicaset frontend --replicas=3
```

* Проверка образа указанного в ReplicaSet:

```
kubectl get replicaset frontend -o=jsonpath='{.spec.template.spec.containers[0].image}' -n default
```

* Проверка образа из которого запущен контейнер:

```
kubectl get pods -l app=frontend -o=jsonpath='{.items[0].spec.containers[0].image}' -n default
```

### Описание:

```
ReplicaSet сделит за тем, сколько POD'ов должно быть запущено, значение указано в .spec.replicas
```

## Deployment

* Проверка работы Deployment:

```
kubectl apply -f paymentservice-deployment.yam -n default
```

* История Deployment:

```
kubectl rollout history deployment paymentservice -n default
```

* Просмотр всех элементов namespace:

```
kubectl get all -o name -n default

pod/paymentservice-7c8468cf9-hcwts
pod/paymentservice-7c8468cf9-kkf6k
pod/paymentservice-7c8468cf9-xvqbx
service/kubernetes
deployment.apps/paymentservice
replicaset.apps/paymentservice-68c65b7974
replicaset.apps/paymentservice-7c8468cf9
```

* Возврат к предыдущей версии:

```
kubectl rollout undo deployment paymentservice --to-revision=1 -n default
```

* Проверка работы сценария развёртывания blue-green:

```
kubectl apply -f paymentservice-deployment-bg.yaml -n default
```

* Проверка работы сценария развёртывания Reverse Rolling Update:

```
kubectl apply -f paymentservice-deployment-reverse.yaml -n default
```

## Probes

* Проверка работы readinessProbe:

```
kubectl apply -f frontend-deployment.yaml -n default
```

* Просмотр состояние POD'ов:

```
kubectl describe pod -n default
```

* Проверка статуса обновления:

```
kubectl rollout status deployment/frontend --timeout=60s -n default
```

* Отмена развёртывания с некорректным readinessProbe:

```
kubectl rollout undo deployment/frontend -n default
```

