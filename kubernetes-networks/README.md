# K8s networks

----------
## Info

Абстракция строится на основе механизма [namespace](https://habr.com/ru/post/458462/)

Если созать для каждого контейнера свой mnt-namespace (m1, m2), pid-namespace (p1, p2), uts-namespace (u1, u2), network namespace (n), ipc namespace (i)
то мы получим такую картину

`container1: n, i, m1, p1, u1`

`container2: n, i, m2, p2, u2`

`containerX: n, i, mX, pX, uX`

Как могут общаться контейнеры внутри Pod? 
- IPC (POSIX message queue, etc)
- localhost доступен
- Внешний интерфейс доступен

Network namespace можно и не создавать, еслиуказать для Pod параметр `hostNetwork: true`

Типы сервисов для доступа к подам
- ClusterIP 
- NodePort
- LoadBalancer
- ExternalName

### DNS
Адресация осуществляется и при помощи DNS
Внутри кластера dnsPolicy в spec для Pod определяет настройки DNS. 
Возможны следующие варианты:
1. Default - копирует resolv.conf c ноды
2. ClusterFirst - резолвер берется из конфигурации кластера а search формируется как
```
  <namespace>.svc.<домен кластера>

  svc.<доменк кластера>

  <доменкластера>
```

Стоит обратить внимание на то, что сервис ссылается по селектору, однако fqdn резолвится по имени сервиса.

```
- service.namespace.svc.<доменкластера>.  FQDN сервиса
- service.namespace                       Обращение к сервису внутри кластера
- service                                 Обращение к сервису внутри namespace
```
3. ClusterFirstWithHostNet-резолвер берется из конфигурации кластера search как в ClusterFirst + в конец дописывается значение search c ноды(рекомендуется при hostNetwork: true)
4. None-настройка производится с помощью блока dnsConfig(всё можно указать самому)

В файле настроек kubelet `/var/lib/kubelet/config.yaml`
можно прописать
1. Адреса резолверов - параметр clusterDNS
2. Домен кластера - параметр clusterDomain

+ доп опции в Pod прописываются при помощи секции dnsConfig

Текущая схема описана [тут](https://github.com/kubernetes/dns/blob/master/docs/specification.md)

DNS-запросы, где меньше 5точек (или не FQDN)обрабатываются так: дописываем домен из списка search и пробуем резолвить если не вышло, берем следующий домен, снова пробуем Обычно,список такой:  
default.svc.cluster.local -> svc.cluster.local -> cluster.local

Проблема с образами alpine:
[краткий разбор](https://github.com/gliderlabs/docker-alpine/blob/master/docs/caveats.md#dns), [полный разбор](https://wiki.musl-libc.org/functional-differences-from-glibc.html#Name-Resolver/DNS), [например](https://pracucci.com/kubernetes-dns-resolution-ndots-options-and-why-it-may-affect-application-performances.html)

Публиковать имена кластера лучше через рекурсор или [ExternalDNS](https://github.com/kubernetes-incubator/external-dns)

## Services
### 1. Cluster IP

Диапазон внутренних адресов берется из параметра `kube-apiserver` `--service-cluster-ip-range=10.96.0.0/12`

Данная лабораторная работа выполняется в minikube c драйвером docker и proxy.mode=ipvs

Запуск minikube с ipvs
``` minikube start --extra-config=kube-proxy.mode=ipvs \
--extra-config=kube-proxy.ipvs.strictARP=true
```

#### IPVS

Межанизм IPVS (IP Virtual Server) строится поверх Netfilter и представляет из себя балансировщик транспортного уровня, являясь частью ядра linux.
IPVS включен в LVS (Linux Virtual Server), и может перенаправлять запрсы на TCP и UDP сервисы, а так же группировать ресурсы кластера за одним виртуальным адресом.

Работа с сервером и правилами ipvs осуществляется утилитами `ipvsadm`, `ipset`
Например, после создания сервиса, публикующего dns `wabat_platform/kubernetes-networks/coredns/publish-dns.yml` через адрес сервис типа loadbalancer по адресу 172.17.255.22 увидим:
```
docker@minikube:~$ sudo ipvsadm -S | grep 172.17.255.22
-A -t 172.17.255.22:domain -s rr
-a -t 172.17.255.22:domain -r 172.17.0.2:domain -m -w 1
-A -u 172.17.255.22:domain -s rr
-a -u 172.17.255.22:domain -r 172.17.0.2:domain -m -w 1

```
По-умолчанию он использует механизм балансировки rr (Round-Robin)

[другие алгоритмы балансировки](https://github.com/kubernetes/kubernetes/blob/1cb3b5807ec37490b4582f22d991c043cc468195/pkg/proxy/apis/config/types.go#L185)
[с описанием](http://www.linuxvirtualserver.org/docs/scheduling.html)


выгрузить текущие правила:
```
ipvsadm -S > ipv.rules
```
[примеры](http://kb.linuxvirtualserver.org/wiki/Ipvsadm)

### 2. NodePort
Параметр kube-apiserver --service-node-port-range (default 30000-32767)
Позволяет получить доступ к сервису снаружи, в том числе в автоматическом режиме при помощи metalllb.
Например:
```
kubectl describe services web-svc-lb 
Name:                     web-svc-lb
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=web
Type:                     LoadBalancer
IP:                       10.96.37.2
LoadBalancer Ingress:     172.17.255.1
Port:                     <unset>  80/TCP
TargetPort:               8000/TCP
NodePort:                 <unset>  31495/TCP
Endpoints:                172.17.0.5:8000
Session Affinity:         None
External Traffic Policy:  Cluster
Events:
  Type    Reason        Age                      From             Message
  ----    ------        ----                     ----             -------
  Normal  nodeAssigned  3m21s (x398 over 2d23h)  metallb-speaker  announcing from node "minikube"

```

### 3. LoadBalancer
#### Установка metalllb
```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.5/manifests/metallb.yaml
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```
Настройка в режиме `l2` существляется при помощи config map

```
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 172.17.255.1-172.17.255.255 
```
Добавляем маршрут
```
sudo ip r add 172.17.255.0/24 via 192.168.49.2 dev br-XXXXXX
```
### 4.ExternalName
cопоставить имя сервиса с любым внешним адресом

### Ingress 

- Единая точка входа вприложения снаружи
- Балансировка трафика
- Терминация SSL
- Набор правил внутри кластера Kubernetes.
- работает на 7 уровне OSI

[виды](https://kubedex.com/ingress/)
[обзор](https://kubernetes.github.io/ingress-nginx/deploy/baremetal/)
https://kubernetes.github.io/ingress-nginx/deploy/baremetal/


#### Ingress с nginx
1.  Установка
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
```

2. Cоздание

сервис типа балансировщик
nginx-lb.yaml
```
kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  externalTrafficPolicy: Local
  type: LoadBalancer
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/instance: ingress-nginx
    app.kubernetes.io/component: controller
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: https
      port: 443
      targetPort: https
```

Список узлов для балансировки заполняется из ресурса Endpoints нужного сервиса (это нужно для "интеллектуальной" балансировки,привязки сессий и т.п.)
Поэтому мы можем использовать headless-сервис для нашего приложения где параметр `clusterIP: None`

После можно создать ingress-прокси к приложению.

#### Информацию и логи для отладки смотрим 
```
kubectl describe ingress/<имя_ingress>
```
```
kubectl -n ingress-nginx logs ingress-nginx-controller-XXXX
```

#### Прикручивание ingress к dashboard
дока по установке борды https://github.com/kubernetes/dashboard
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: dashboard
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
  namespace: kubernetes-dashboard
spec:
  rules:
  - http:
      paths:
      - path: /dashboard(/|$)(.*)
        pathType: Prefix
        backend:
          service: 
            name: kubernetes-dashboard
            port: 
              number: 443
```
```
kubectl describe ingress/dashboard
```

#### canary
В папке wabat_platform/kubernetes-networks/canary лежат конфиги для того что бы развернуть 2 севиса с использованием hostname, что позволит направить часть трафика по заголовкам с указанием хоста и меткой `canary` через kubernetes-networks/canary/canary-inrgess.yaml, остальная часть трафика идет через kubernetes-networks/web-ingress.yaml. Для балнсировки между подами деплоймента используется headless service.

Проверить можно так:

- если устанавливать вес 
```
for i in $(seq 1 10); do curl -s -H "Host: web.app" http://172.17.255.2/web | grep "HOSTNAME=" ; done
```
- с передачей через заголовок, без веса
```
curl -s -H "canary: always" -H "Host: web.app" http://172.17.255.2/web | grep "HOSTNAME"
curl -s -H "canary: never" -H "Host: web.app" http://172.17.255.2/web | grep "HOSTNAME"
```