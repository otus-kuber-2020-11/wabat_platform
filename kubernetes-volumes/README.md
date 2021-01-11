# Kubernetes volumes
---------------------
- Volume - абстракция реального хранилища
- Volume создается и удаляется вместе с подом
- Один и тот же Volume может использоваться одновременнонесколькими контейнерами в поде


## Типы volumes
### [emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir)
- Используется для обмена файлами между разными контейнерами одного pod
- Создается вместе с подом и удаляется вместе с ним (падение контейнера в поде не влияет на данные)
- Все контейнеры пода имеют rw на одни и те же файлы
- Может быть смонтирована в разные точки контейнеров
- Данные могут храниться в tmpfs (emptyDir.medium: "Memory")

### [hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)
- монтирование файла или директории с хоста
- используется часто с daemonsets
- файл или директория должны существовать на хосте, если не указана опция, которая создаст пустой файл или каталог при его отсутствии
В том числе блочные и chardev

### cloud volume
- aws
- gcp
- etc

### 'служебные' типы
- СonfigMap
- Secret

Например,
создать секрет
```
kubectl create secret generic 1-minio-secret --from-literal=accesskey=minio --from-literal=secretkey=minio123

```
```
apiVersion: v1
kind: Secret
metadata:
  name: 1-minio-secret
type: Opaque
data:
  accesskey: bWluaW8=
  secretkey: bWluaW8xMjM=
```
```
kubectl create secret generic <name>-minio-secret --from-literal=accesskey=$(head -c 512 /dev/urandom | LC_CTYPE=C tr -cd 'a-zA-Z0-9' | head -c 20) --from-literal=secretkey=$(head -c 512 /dev/urandom | LC_CTYPE=C tr -cd 'a-zA-Z0-9' | head -c 64)
```

### [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes)
PV похожи на обычные Volume, но имеют отдельный от сервисов жизненный цикл
[типы PV](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#types-of-persistent-volumes)

### [PersistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes)
- Запрос на хранилище от пользователя
- PVC могут требовать определенный объем хранилища и правдоступа
- Создаются на уровне namespace

### [Local persistent volumes](https://kubernetes.io/blog/2019/04/04/kubernetes-1.14-local-persistent-volumes-ga/)

### [Режимы доступа (Access Modes)]
-    ReadWriteOnce -- the volume can be mounted as read-write by a single node
-    ReadOnlyMany -- the volume can be mounted read-only by many nodes
-    ReadWriteMany -- the volume can be mounted as read-write by many nodes

[таблица совметсимости разных типов fs c этими режимами](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#access-modes)


## StatefullSet
- Каждый под имеет уникальное состояние (имя, сетевой адрес,volumes)
- Для каждого создается отдельный PVC
- для обеспечения сетевой связности должен использоваться HeadlessService
- Удаление/масштабирование подов не удаляет тома, связанные сними
- У каждого pod свой pvc и свой pv, поэтому надо пользоватьсяvolume claim template

