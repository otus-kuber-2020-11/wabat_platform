# Мониторинг кластера kubernetes с prometheus-operator

## Установка kube-prometheus (quick start)

Один из самых простых способов начать использовать prometheus-operator - установка как часть [kube-prometheus](https://prometheus-operator.dev/docs/prologue/quick-start/#next-steps).

```bash
git clone https://github.com/prometheus-operator/kube-prometheus.git
```
```bash
# Deploy kube-prometheus
# Create the namespace and CRDs, and then wait for them to be availble before creating the remaining resources
kubectl create -f manifests/setup
until kubectl get servicemonitors --all-namespaces ; do date; sleep 1; echo ""; done
kubectl create -f manifests/
```
После установки
```bash
$ kubectl get deployments.apps -n monitoring
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
blackbox-exporter     1/1     1            1           2d20h
grafana               1/1     1            1           2d20h
kube-state-metrics    1/1     1            1           2d20h
prometheus-adapter    2/2     2            2           2d20h
prometheus-operator   1/1     1            1           2d20h

```
```bash
$ kubectl get statefulsets.apps -n monitoring 
NAME                READY   AGE
alertmanager-main   3/3     2d20h
prometheus-k8s      2/2     2d20h
```

---
Operator имеет следующие [ресурсы](https://prometheus-operator.dev/docs/operator/design/) (CRD), которые описывают желаемое состояние кластера

- Prometheus 

Этот CRD предоставляет опции для конфигурации replication, persistent storage, и Alertmanagers в которые он шлет уведомления. 
Для каждого такого ресурса Operator разворачвает сконфигурированный StatefulSet. В каждый Pod Prometheus монтируется секрет, содержащий конфигурацию Prometheus.
Можно конфигурировать этот секрет вручную или автоматически при помощи ServiceMonitor.


- ServiceMonitor

ServiceMonitor CRD описывает список целей для мониторинга.
Для мониторинга любого приложения должен существовать объект Endpoints, который представляет из себя список ip адресов. 

1) Обычно, Endpoints формируется автоматически из объекта типа Service (включая Service с несколькими портами). Service в свою очередь находит Pod по метке. 
Пример deployment с nginx + nginx exporter
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx
  namespace: default
data:
  nginx.conf: |
    worker_processes auto;
    events {
      worker_connections 1024;
    }
    http {
      # Hide nginx version information.
      server_tokens off;
      gzip on;
      # Compression level (1-9).
      # 5 is a perfect compromise between size and CPU usage, offering about
      # 75% reduction for most ASCII files (almost identical to level 9).
      gzip_comp_level    5;
      # Don't compress anything that's already small and unlikely to shrink much
      # if at all (the default is 20 bytes, which is bad as that usually leads to
      # larger files after gzipping).
      # Default: 20
      gzip_min_length    256;
      # Tell proxies to cache both the gzipped and regular version of a resource
      # whenever the client's Accept-Encoding capabilities header varies;
      # Avoids the issue where a non-gzip capable client (which is extremely rare
      # today) would display gibberish if their proxy gave them the gzipped version.
      # Default: off
      gzip_vary          on;
      server {
        listen 80 default_server;
        listen [::]:80 default_server;
        root /usr/share/nginx/html/;
        location / {
          access_log off;
          try_files $uri $uri/ =404;
        }
      }
      server {
        server_name metrics;
        listen 8080;
        listen [::]:8080;
        location /probes/health {
          access_log off;
          return 200 "healthy";
        }
        location /nginx_status {
            access_log off;
            stub_status;
        }
      }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-status-deployment
  labels:
    app: nginx-status-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-status-deployment
  template:
    metadata:
      labels:
        app: nginx-status-deployment
    spec:
      volumes:
      - name: config
        configMap:
          name: nginx
      containers:
      - name: nginx
        image: nginx:1.19.9-alpine
        ports:
        - containerPort: 80
        volumeMounts:           
        - name: config
          mountPath: /etc/nginx
        readinessProbe:
          httpGet:
            path: /probes/health
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 5
          failureThreshold: 3
          successThreshold: 1
        livenessProbe:
          httpGet:
            path: /probes/health
            port: 8080
          initialDelaySeconds: 5
          timeoutSeconds: 5
          periodSeconds: 15
          failureThreshold: 3
          successThreshold: 1
      - name: metrics
        image: nginx/nginx-prometheus-exporter:0.9.0
        imagePullPolicy: IfNotPresent
        ports:
          - containerPort: 9113
        args: [
          "-nginx.scrape-uri", "http://localhost:8080/nginx_status"
          ]
---
kind: Service
apiVersion: v1
metadata:
  name: nginx-status-deployment
  labels:
    app: nginx-status-deployment
spec:
  selector:
    app: nginx-status-deployment
  ports:
  - name: nginx-status-deployment
    port: 9113
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nginx-status-deployment
spec:
  selector:
    matchLabels:
      app: nginx-status-deployment
  endpoints:
  - port: nginx-status-deployment

```

2) Для мониторинга ресурса находящегося вне кластера и отдающего свои метрики по пути http://10.10.0.10/metrics: и объект Endpoints, и Service нужно будет создать,
```
apiVersion: v1
kind: Endpoints
metadata:
  name: pxc-proxy
subsets:
- addresses:
  # list all external ips for this service
  - ip: 10.10.0.10
  ports:
  - name: pxc-proxy
    port: 80
    protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: pxc-proxy
  labels:
    app: pxc-proxy
spec:
  ports:
  - name: pxc-proxy
    port: 80
    protocol: TCP
    targetPort: 80
  clusterIP: None
  type: ClusterIP
```
ServiceMonitor для этого сервиса будет выглядеть так
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: pxc-proxy
spec:
  selector:
    matchLabels:
      app: pxc-proxy
  endpoints:
  - port: pxc-proxy
```

- Alertmanager
- ThanosRuler
- PodMonitor
- Probe
- PrometheusRule
- AlertmanagerConfig


---
 
## Установка с кастомизацией, создание проекта на основе библиотеки kube-prometheus
Альтернативный способ установки и обновления мониторинга с prometheus operator - использовать кастомизацию всей библиотеки kube-prometheus.
Для компиляции манифестов требуется gojsontoyaml и jsonnet.

```
# создание директории под проекта
mkdir my-kube-prometheus-next; cd my-kube-prometheus-next
```
```
# initial/empty `jsonnetfile.json`
jb init
```
```
# установка зависимотей, создает директорию vendor/ 
jb install github.com/prometheus-operator/kube-prometheus/jsonnet/kube-prometheus@release-0.7
```
```
# пример example.jsonnet
$ wget https://raw.githubusercontent.com/prometheus-operator/kube-prometheus/release-0.7/example.jsonnet -O example.jsonnet
```
```
# скрипт компиляции манифестов
$ wget https://raw.githubusercontent.com/prometheus-operator/kube-prometheus/release-0.7/build.sh -O build.sh
```

```
# создает директорию vendor/ 
jb install github.com/prometheus-operator/kube-prometheus/jsonnet/kube-prometheus@release-0.7
```
например, настроить мониторинг namespace отличных от default и kube-system
файл с следующим содержимым передаем аргументом в build.sh

```
# file other-namespaces.jsonnet
local kp = (import 'kube-prometheus/kube-prometheus.libsonnet') + {
  _config+:: {
    namespace: 'monitoring',

    prometheus+:: {
      namespaces: ["default", "kube-system", "production"],
    },
  },
};
 
{ ['00namespace-' + name]: kp.kubePrometheus[name] for name in std.objectFields(kp.kubePrometheus) } +
{ ['0prometheus-operator-' + name]: kp.prometheusOperator[name] for name in std.objectFields(kp.prometheusOperator) } +
{ ['node-exporter-' + name]: kp.nodeExporter[name] for name in std.objectFields(kp.nodeExporter) } +
{ ['kube-state-metrics-' + name]: kp.kubeStateMetrics[name] for name in std.objectFields(kp.kubeStateMetrics) } +
{ ['alertmanager-' + name]: kp.alertmanager[name] for name in std.objectFields(kp.alertmanager) } +
{ ['prometheus-' + name]: kp.prometheus[name] for name in std.objectFields(kp.prometheus) } +
{ ['grafana-' + name]: kp.grafana[name] for name in std.objectFields(kp.grafana) }

```
далее 
```
./build.sh other-namespaces.jsonnet
```
на выходе имеем набор манифестов в диерктории manifests/
