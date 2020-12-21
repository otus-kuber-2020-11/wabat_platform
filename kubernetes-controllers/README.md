## Kubernetes Controllers

### ReplicaSet

Назначение данного вида контроллеров - следить что бы в данный момент времени стабильно было запущено определенное количество идентичных подов.
Обновление версии приложения в запущенном replicaset возможно при помощи параметра scale, т.е. например, если текущая версия образа в реплике 0.2 а реплик 3, то
```
kubectl scale replicaset <label> --replicas=2

Например, приложение с меткой <label> в файле kubernetes-controllers/<label>-replicaset.yaml
```
это уменьшит текущее количество реплик до 2
затем, меняем версию образа в <label>-replicaset.yaml и применяем манифесит еще раз
```
kubectl apply -f kubernetes-controllers/<label>-replicaset.yaml | kubectl get pods -l app=<label> -w
```
Replicaset создаст еще один под с новой версией приложения, при этом в двух оставшихся подах приложение не изменится.

Это происходит потому, что replicaset не проверяет соответствие запущенных Podов шаблону -> пока скейл не будет уменьшен до 0, все реплики не обновят свою версию.

посмотреть поды из каких образов зарущены можно так
```
kubectl get pods -l app=<label> -o=jsonpath='{.items[0:3].spec.containers[0].image}'
```
а какой образ указан в репликасэт так
```
kubectl get replicaset <label> -o=jsonpath='{.spec.template.spec.containers[0].image}'
```
про форматирование [тут](https://kubernetes.io/docs/reference/kubectl/jsonpath/)

### Deployment
Такой тип контроллера следит за запуском подов, поддерживает автоматичесткие обновления для rs.
Рекомендованный способ запуска Podов (даже если нужна только однареплика).

откат до первой ревизии (версия считается по порядковому номеру деплоя не зависимо от версии указанной в тэге)
kubectl rollout undo deployment paymentservice --to-revision=1 | kubectl get rs -l app=paymentservice -w

посмотреть версии
```
kubectl rollout history deployment paymentservice
```
Дефолтная стратегия обновления - rolling update
при помощи параметров maxSurge и maxUnavailable можно менять количество и скорость замены подов 
[Max Surge](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-surge)
[Max Unavailable](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#max-unavailable)

Механизмы проверки состояния приложения [Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) 

```
kubectl rollout status deployment/name
```

могут быть полезны при организации автоматизации с использованием CI
например:
```
#Gitlab CI
deploy_job:
  stage:deploy
  script:    
    - kubectl apply -f frontend-deployment.yaml    
    - kubectl rollout statusdeployment/frontend --timeout=60s

rollback_deploy_job:
  stage: rollback  
  script:
    - kubectl rollout undo deployment/frontend  
      when: on_failure
```

### DaemonSet
Особенность данного вида контроллера состоит в том, что экземпляр пода создается на каждой ноде.
Типичные случаи применения: сетевые плагины, экспортеры метрик, утиличты для сбора и отправки метрик

Настройка экспортера:
Манифесты взяты [тут](https://github.com/prometheus-operator/prometheus-operator)

Запуск в следующем порядке:
```
kubectl create ns monitoring
namespace/monitoring created

kubectl apply -f exporter_deps/_node-exporter-serviceAccount.yaml
serviceaccount/node-exporter created

kubectl apply -f exporter_deps/node-exporter-clusterRole.yaml
clusterrole.rbac.authorization.k8s.io/node-exporter created

kubectl apply -f exporter_deps/node-exporter-clusterRoleBinding.yaml
clusterrolebinding.rbac.authorization.k8s.io/node-exporter created

kubectl apply -f node-exporter-daemonset.yaml
daemonset.apps/node-exporter created

kubectl apply -f exporter_deps/node-exporter-service.yaml
service/node-exporter created
```
