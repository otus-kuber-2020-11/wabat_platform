# Описание контроллера

Kонтроллер будет обрабатывать два типа событий:

1) При создании объекта типа `kind: MySQL`:
```
* Cоздавать PersistentVolume, PersistentVolumeClaim, Deployment, Service для mysql
* Создавать PersistentVolume, PersistentVolumeClaim для бэкапов базы данных, если их еще нет.
* Пытаться восстановиться из бэкапа
```

2) При удалении объекта типа `kind: MySQL`:
```
* Удалять все успешно завершенные backup-job и restore-job
* Удалять PersistentVolume, PersistentVolumeClaim, Deployment, Service для mysql
```
