# Цель
- настроить автоматическое масштабирование тестового приложения через Kubernetes
  - по изменению памяти
  - по RPS через Prometheus

## Тестовое приложение:
- docker-image: `ghcr.io/yandex-practicum/scaletestapp:sha256-eff20ae3ae2d596375f9ed6d612a78d149a35a66cd2907ea90d7175ca918c993.sig`
- порт: `8080`
- ручки:
    - GET / — возвращает идентификатор pod
    - GET /metrics — Prometheus-метрики, внутри есть `http_requests_total`


## Подготовка
- установка kubernetes 
```
// windows Powershell
winget install Kubernetes.kubectl
kubectl version --client
```

- установка minikube
```
// windows Powershell
winget install Kubernetes.minikube
minikube version
```

- установка locust
```
pip install locust
```


## Описание файлов в директории Task2/app
- locustfile.py - задание для locust
- k8s/deployment.yaml - Deployment приложения (1 реплика, memory limit 30Mi)
- k8s/service.yaml - Service для доступа к приложению
- k8s/hpa-memory.yaml - HPA по памяти (по заданию: target 80%, maxReplicas 10)
- results/ - результаты

### Запуск
#### Часть 1. Динамическая маршрутизация на основании показателей утилизации памяти
- запуск minikube
```
minikube start
```
- активация metrics-server
```
minikube addons enable metrics-server

// чек запуска
kubectl get pods -n metrics-server // metrics-server должен быть Running
-----------------------
NAME       CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)   
minikube   160m         0%       674Mi           4%

```

- запуск тестового приложения через манифесты
```
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl apply -f k8s/hpa-memory.yaml

// чек запуска
kubectl top pods

-----------------------
NAME                           CPU(cores)   MEMORY(bytes)   
scaletestapp-9778c76df-nkv5j   1m           1Mi

```

- получить URL тестового приложения
```
minikube service scaletestapp --url

------------
http://127.0.0.1:52922/

```

- запускаем locust
```
/Task2/app/locust  - запуститься атвоматом так как название скрипта locustfile.py

// запускается на поту 8089
добавляем http://127.0.0.1:52922/
```


- тестируем нагрузку
  - запускаем на одно пользователе
  - просмотр через UI мникуба
    - minikube dashboard
  - просмотр логов kubernetes
    - kubectl get hpa -w
  - предварительные результаты
    - нагрузка медленная, один под (тас 3 сета пробные были, далее удалили, они и не были запущены)
      - ![minikube-locust промежуточное 1 pod, запросов около 400.png](app/results/minikube-locust%20%D0%BF%D1%80%D0%BE%D0%BC%D0%B5%D0%B6%D1%83%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D0%B5%201%20pod%2C%20%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%BE%D0%B2%20%D0%BE%D0%BA%D0%BE%D0%BB%D0%BE%20400.png)
      - ![kubernetes-locust промежуточное 1 pod, запросов около 800.png.png](app/results/kubernetes-locust%20%D0%BF%D1%80%D0%BE%D0%BC%D0%B5%D0%B6%D1%83%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D0%B5%201%20pod%2C%20%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%BE%D0%B2%20%D0%BE%D0%BA%D0%BE%D0%BB%D0%BE%20800.png.png)
  - добавил 10 пользователей, ибо долго растет память 
    - предварительное на 10 пользователей, запросов около 10000
      - ![kubernetes-locust промежуточное 1 pod, 10 юзеров запросов около 10000.png.png.png](app/results/kubernetes-locust%20%D0%BF%D1%80%D0%BE%D0%BC%D0%B5%D0%B6%D1%83%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D0%B5%201%20pod%2C%2010%20%D1%8E%D0%B7%D0%B5%D1%80%D0%BE%D0%B2%20%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%BE%D0%B2%20%D0%BE%D0%BA%D0%BE%D0%BB%D0%BE%2010000.png.png.png)
      - ![minikube-locust промежуточное 1 pod, 10 юзеров запросов около 10000.png](app/results/minikube-locust%20%D0%BF%D1%80%D0%BE%D0%BC%D0%B5%D0%B6%D1%83%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D0%B5%201%20pod%2C%2010%20%D1%8E%D0%B7%D0%B5%D1%80%D0%BE%D0%B2%20%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%BE%D0%B2%20%D0%BE%D0%BA%D0%BE%D0%BB%D0%BE%2010000.png)
  - на 41-42 минуте по логам память, то падает, то растет медленно
    - добавил пользователей до 100 
      - ![kubernetes-locust промежуточное 1 pod, 100 юзеров запросов около 68000.png.png.png](app/results/kubernetes-locust%20%D0%BF%D1%80%D0%BE%D0%BC%D0%B5%D0%B6%D1%83%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D0%B5%201%20pod%2C%20100%20%D1%8E%D0%B7%D0%B5%D1%80%D0%BE%D0%B2%20%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%BE%D0%B2%20%D0%BE%D0%BA%D0%BE%D0%BB%D0%BE%2068000.png.png.png)
      - память по тихоньку растет до 68%
    - устал, добавил до 1000 пользователей, но в locust  увеличивается медленно показания около 269 ползьвоателей
        - ![minikube-locust промежуточное 2 pod, 269 юзеров запросов около 240000.png](app/results/minikube-locust%20%D0%BF%D1%80%D0%BE%D0%BC%D0%B5%D0%B6%D1%83%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D0%B5%202%20pod%2C%20269%20%D1%8E%D0%B7%D0%B5%D1%80%D0%BE%D0%B2%20%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%BE%D0%B2%20%D0%BE%D0%BA%D0%BE%D0%BB%D0%BE%20240000.png)
      - ура, запустился еще один под
        - ![рождение пода.png](app/results/%D1%80%D0%BE%D0%B6%D0%B4%D0%B5%D0%BD%D0%B8%D0%B5%20%D0%BF%D0%BE%D0%B4%D0%B0.png)
        - ![kubernetes-locust промежуточное 2 pod, 361 юзеров запросов около 300 000.png](app/results/kubernetes-locust%20%D0%BF%D1%80%D0%BE%D0%BC%D0%B5%D0%B6%D1%83%D1%82%D0%BE%D1%87%D0%BD%D0%BE%D0%B5%202%20pod%2C%20361%20%D1%8E%D0%B7%D0%B5%D1%80%D0%BE%D0%B2%20%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%BE%D0%B2%20%D0%BE%D0%BA%D0%BE%D0%BB%D0%BE%20300%20000.png)

## Задание 2 Динамическая маршрутизация на основании показателей количества запросов в секунду


### Подготовка
- установка helm
```
winget install Helm.Helm

// чек установки
helm version


```

- установка Prometheus
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install prom prometheus-community/prometheus

// чекаем
kubectl get pods

проброс портов для прометеуса
kubectl port-forward svc/prom-prometheus-server 9090:80

```

- делаем отдельный service-metric.yaml
  - [service-metric.yaml](app/k8s/service-metric.yaml)
```
// конфигурим 
kubectl apply -f k8s/service-metric.yaml

```

### Готовим адаптер Prometheus
- файл маниффест Task2/app/k8s/adapter/values.yaml  -  Prometheus Adapter (правило для RPS)
  - [values.yaml](app/k8s/adapter/values.yaml)
- подтянуть манифест из Task2/app
```
helm install adapter prometheus-community/prometheus-adapter -f adapter/values.yaml

// чекаем
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1"

```

- создаем hpa-rps.yaml - HPA по RPS (метрики подов)
  - [hpa-rps.yaml](app/k8s/hpa-rps.yaml)
  - подтягиваем
```
kubectl apply -f hpa-rps.yaml
```

- где-то пока настраивал
  - надо удалить предыдущий запуск для памяти
```
kubectl delete hpa scaletestapp-hpa-memory
```

- проверяем живых
  - Custom Metrics API жив
```

 kubectl get apiservices | findstr metrics
v1beta1.custom.metrics.k8s.io     default/adapter-prometheus-adapter   True        26m
v1beta1.metrics.k8s.io            kube-system/metrics-server           True        164m

```
- Метрика реально отдаётся
```
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/services/scaletestapp/http_requests_per_second"
{"kind":"MetricValueList","apiVersion":"custom.metrics.k8s.io/v1beta1","metadata":{},"items":[{"describedObject":{"kind":"Service","namespace":"default","name":"scaletestapp","apiVersion":"/v1"},"metricName":"http_requests_per_second","timestamp":"2026-01-25T18:12:53Z","value":"0","selector":null}]}

```

- HPA видит метрику
```
kubectl get hpa
NAME                   REFERENCE                 TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
scaletestapp-hpa-rps   Deployment/scaletestapp   0/10      1         10        1          10m


kubectl describe hpa scaletestapp-hpa-rps

PS D:\Edication\cource_architector\8\Task2\app> kubectl describe hpa scaletestapp-hpa-rps
Name:                                                                 scaletestapp-hpa-rps
Namespace:                                                            default
Labels:                                                               <none>
Annotations:                                                          <none>
CreationTimestamp:                                                    Sun, 25 Jan 2026 21:02:30 +0300
Reference:                                                            Deployment/scaletestapp
Metrics:                                                              ( current / target )
  "http_requests_per_second" on Service/scaletestapp (target value):  0 / 10
Min replicas:                                                         1
Max replicas:                                                         10
Deployment pods:                                                      1 current / 1 desired
Conditions:
  Type            Status  Reason            Message
  ----            ------  ------            -------
  AbleToScale     True    ReadyForNewScale  recommended size matches current size
  ScalingActive   True    ValidMetricFound  the HPA was able to successfully calculate a replica count from Service metric http_requests_per_second
  ScalingLimited  True    TooFewReplicas    the desired replica count is less than the minimum replica count

```
- Prometheus Targets
  - ![check request  http_requests_total.png](app/results/check%20request%20%20http_requests_total.png)

- Prometheus запрос
  - ![sum(rate(http_requests_total[5m])).png](app/results/sum%28rate%28http_requests_total%5B5m%5D%29%29.png)
  

- теперь нагружаем locust
  - теперь запрос в sum(rate(http_requests_total[5m])) by (namespace, service) меняется и растет
    - ![тест-locust rps при запросе в протеусе sum(rate(http_requests_total[5m]))  .png](app/results/%D1%82%D0%B5%D1%81%D1%82-locust%20rps%20%D0%BF%D1%80%D0%B8%20%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%B5%20%D0%B2%20%D0%BF%D1%80%D0%BE%D1%82%D0%B5%D1%83%D1%81%D0%B5%20sum%28rate%28http_requests_total%5B5m%5D%29%29%20%20.png)
    - ![тест-locust rps при запросе в протеусе sum(rate(http_requests_total[5m])) растет .png](app/results/%D1%82%D0%B5%D1%81%D1%82-locust%20rps%20%D0%BF%D1%80%D0%B8%20%D0%B7%D0%B0%D0%BF%D1%80%D0%BE%D1%81%D0%B5%20%D0%B2%20%D0%BF%D1%80%D0%BE%D1%82%D0%B5%D1%83%D1%81%D0%B5%20sum%28rate%28http_requests_total%5B5m%5D%29%29%20%D1%80%D0%B0%D1%81%D1%82%D0%B5%D1%82%20.png)
  - смотрим за kubectl get hpa -w
    - изначально target=0, далее растет на дрожжах  до 60000 значение метрики sum(rate(http_requests_total[5m])) by (namespace, service)
      - ![тест-locust rps  логи get hpa -w .png](app/results/%D1%82%D0%B5%D1%81%D1%82-locust%20rps%20%20%D0%BB%D0%BE%D0%B3%D0%B8%20get%20hpa%20-w%20.png)
    - так же видит новенькие поды, рождаются 
      - ![тест-locust rps  логи get pods -w .png](app/results/%D1%82%D0%B5%D1%81%D1%82-locust%20rps%20%20%D0%BB%D0%BE%D0%B3%D0%B8%20get%20pods%20-w%20.png)
    - так же из логов видно ограничение на количество подов, больше 10 не создается
      - Horizontal Pod Autoscaler сравнил текущее значение метрики с заданным порогом и вычислил необходимость увеличения количества реплик.