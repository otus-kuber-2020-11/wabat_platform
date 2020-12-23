## Безопасность и управление доступом

### RBAC

RBAC состоит из трех элементов:
1) Субъект - пользователь или процесс

2) Ресурс - единица доступа (apiGroups)

3) Глагол -  (get, watch, create, delete и т.п.) операции CRUD (Create, Read, Update, Delete).

Элементы 1 и 2 соединяются в роли (ресурсы + глаголы). Могут носить глобальный характер (ClusterRole) или существовать внутри namespace. Добавляется привязка к субъекту при помощи RoleBindings (для глобального ClusterRoleBindings). 


----------------
Список доступных apiGroup
```
kubectl  api-resources
```
----------------
ссылки

https://sysdig.com/blog/kubernetes-security-rbac-tls/

https://thenewstack.io/three-realistic-approaches-to-kubernetes-rbac/

https://www.cncf.io/blog/2018/08/01/demystifying-rbac-in-kubernetes/ 

примеры 

https://github.com/terickson/kubernetes-rbac-policies
