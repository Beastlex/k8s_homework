# Задание 2

Создаем манифесты для запуск
```console
$ kubectl create secret generic connection-string --from-literal=DATABASE_URL=postgres://connect --dry-run=client -o yaml > secret.yaml
$ kubectl create configmap user --from-literal=firstname=Alex --from-literal=lastname=Zverev --dry-run=client -o yaml > cm.yaml
```

Запускаем манифесты 
```console
$ kubectl apply -f secret.yaml
$ kubectl apply -f cm.yaml
$ kubectl apply -f pod.yaml
```

Подключаемся к поду и смотрим вывод переменных среды
```console
$ kubectl exec -it nginx -- bash
$ printenv
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_SERVICE_PORT=443
DATABASE_URL=postgres://connect
HOSTNAME=nginx
PWD=/
PKG_RELEASE=1~bullseye
HOME=/root
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
NJS_VERSION=0.7.1
TERM=xterm
SHLVL=1
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
lastname=Zverev
KUBERNETES_SERVICE_HOST=10.96.0.1
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_PORT_443_TCP_PORT=443
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
firstname=Alex
NGINX_VERSION=1.21.5
_=/usr/bin/printenv
```

Запускаем манифесты для deployment и получаем список подов
```console
$ kubectl apply -f nginx-configmap.yaml 
$ k apply -f deployment.yaml
$ kubectl get pods -o wide
NAME                      READY   STATUS    RESTARTS   AGE   IP            NODE       NOMINATED NODE   READINESS GATES
nginx                     1/1     Running   0          35m   172.17.0.6    minikube   <none>           <none>
nginx2-54c7bd9d87-4vmcs   1/1     Running   1          47h   172.17.0.10   minikube   <none>           <none>
nginx2-54c7bd9d87-d6g9x   1/1     Running   1          47h   172.17.0.5    minikube   <none>           <none>
web                       1/1     Running   1          47h   172.17.0.9    minikube   <none>           <none>
web-6745ffd5c8-2lcms      1/1     Running   0          61s   172.17.0.12   minikube   <none>           <none>
web-6745ffd5c8-nlbdj      1/1     Running   0          61s   172.17.0.13   minikube   <none>           <none>
web-6745ffd5c8-vcrhr      1/1     Running   0          61s   172.17.0.14   minikube   <none>           <none>
webreplica-lg572          1/1     Running   1          47h   172.17.0.7    minikube   <none>           <none>

```

Пробуем соединиться с помощью c компьютера
```console
$ curl 172.17.0.13
curl: (7) Failed to connect to 172.17.0.13 port 80: No route to HOSTNAME
```
Так происходит, потом что по умолчанию под доступен только внутри 
кластера по своему внутреннему IP. А никакого Service мы с ним
не связывали, чтобы получить внешний IP

Пробуем соединиться из сессии minikube ssh
```console
$ minikube ssh
$ curl 172.17.0.13
web-6745ffd5c8-nlbdj
```
Так происходит потому что находимся внутри кластера, и внутренние
IP доступны для взаимодействия. Обращене возможно только при
условии использования kubectl proxy или port-forward

Тоже происходит, когда мы пробуем получить доступ с другой ноды
кластера
```console
$ kubectl exec -it $(kubectl get pod |awk '{print $1}'|grep web-|head -n1) bash
$ curl 172.17.0.13
web-6745ffd5c8-nlbdj
```

Создаем Service типа ClusterIP и проверяем его доступность
```console
$ k expose deployment/web --type=ClusterIP --dry-run=client -o yaml > service_template.yaml
$ k apply -f service_template.yaml 
$ k get service
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   2d23h
web          ClusterIP   10.97.237.14   <none>        80/TCP    7s
```

Пробуем соединиться с сервисом с ПК
```console
curl 10.97.237.14 -m 10
curl: (28) Connection timed out after 10000 milliseconds
```
Так происходит, потому что Service типа ClusterIP доступен
только внутри кластера.

Пробуем соединиться из сессии minikube ssh
```console
$ minikube ssh
$ curl 10.97.237.14
web-6745ffd5c8-nlbdddj
$ curl 10.97.237.14
web-6745ffd5c8-2lcms
```
Выводимое значение может меняться при каждом запросе, потому
что мы создали Service который, который связан с разными подами.
Тоже происходит и при доступе с пода в кластере.
```console
$ k exec -it $(kubectl get pod |awk '{print $1}'|grep web-|head -n1) bash
root@web-6745ffd5c8-2lcms:/# curl 10.97.237.14
web-6745ffd5c8-vcrhr
root@web-6745ffd5c8-2lcms:/# curl 10.97.237.14
web-6745ffd5c8-nlbdj
```

Создаем Service типа NodePort и убеждаемся в его доступности
```console
$ k apply -f service-nodeport.yaml
$ k get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        2d23h
web          ClusterIP   10.97.237.14    <none>        80/TCP         30m
web-np       NodePort    10.106.25.248   <none>        80:32752/TCP   0s
```

Проверяем доступность по IP minikube
```console
$ minikube ip
192.168.59.100
$ curl 192.168.59.100:32752
web-6745ffd5c8-vcrhr
```

Создаем сервис, связанный с NodePort и проверяем доменное имя с пода
```console
$ k apply -f service-headless.yaml
$ k exec -it $(kubectl get pod |awk '{print $1}'|grep web-|head -n1) bash
$ cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Создаем под c dnsulils и сравниваем доменные имена сервисов
```console
$ k apply -f https://k8s.io/examples/admin/dns/dnsutils.yaml
$ k exec -i -t dnsutils -- nslookup web
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   web.default.svc.cluster.local
Address: 10.97.237.14

$ k exec -i -t dnsutils -- nslookup web-headless
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   web-headless.default.svc.cluster.local
Address: 172.17.0.12
Name:   web-headless.default.svc.cluster.local
Address: 172.17.0.13
Name:   web-headless.default.svc.cluster.local
Address: 172.17.0.14

```

Добавляем в кластер ingress controller
```console
$ minikube addons enable ingress
$ k get pods -n ingress-nginx
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-r5c6c        0/1     Completed   0          111s
ingress-nginx-admission-patch-994zw         0/1     Completed   1          111s
ingress-nginx-controller-6d5f55986b-tcf92   1/1     Running     0          111s
$ k get pod $(kubectl get pod -n ingress-nginx|grep ingress-nginx-controller|awk '{print $1}') -n ingress-nginx -o yaml > result_ingres.yaml

$ k apply -f ingress.yaml
$ curl $(minikube ip)
web-6745ffd5c8-2lcms
```

# Домашнее задание

Получаем поды и сслыку на ту сущность, которая контролирует их 
жизненый цикл, из пространства имен kube-system
```console
$  k get pod -n kube-system -o jsonpath='{range .items[*]}{@.metadata.name}{" - "}{@.metadata.ownerReferences[*].name}{"\n"}{end}'
coredns-64897985d-fpxcr - coredns-64897985d
etcd-minikube - minikube
kube-apiserver-minikube - minikube
kube-controller-manager-minikube - minikube
kube-proxy-rxxhg - kube-proxy
kube-scheduler-minikube - minikube
metrics-server-68fbbb47dc-lvnll - metrics-server-68fbbb47dc
metrics-server-fc6f95b99-427jr - metrics-server-fc6f95b99
storage-provisioner -
```

Создаем Canary Deployments. В файле [canary-ingress.yaml](canary-ingress.yaml) создаем
Ingress с тем же именем ingress-web, что и Ingress, созданный
ранее. Но в другом namespace - web-canary. Устанавливаем 
правило, чтобы заголовок canary:always перенаправлялся сюда
и устранвливаем процент в 20 для перенаправления.
