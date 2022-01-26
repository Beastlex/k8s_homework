# Задание 3

Создаем и проверям PersistentVolume
```console
$ k apply -f pv.yaml
$ k get pv
NAME                  CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
minio-deployment-pv   5Gi        RWO            Retain           Available                                   48s
```

Создаем и проверяем PersistentVolumeClaim
```console
$ k apply -f pvc.yaml
$ k get pv
NAME                  CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                            STORAGECLASS   REASON   AGE
minio-deployment-pv   5Gi        RWO            Retain           Bound    default/minio-deployment-claim                           2m59s

$ k get pvc
NAME                     STATUS   VOLUME                CAPACITY   ACCESS MODES   STORAGECLASS   AGE
minio-deployment-claim   Bound    minio-deployment-pv   5Gi        RWO                           2m7s
```

Создаем Deployment minio и связываем с ним Service типа NodePort
```console
$ k apply -f deployment.yaml
$ k apply -f minio-nodeport.yaml
$ curl 192.168.59.100:30008 > output.html
```

Создаем StatefulSet и связываем с ним Service 
```console
$ k apply -f statefulset.yaml
$ k get pod
NAME                      READY   STATUS    RESTARTS       AGE
dnsutils                  1/1     Running   45 (31m ago)   45h
minio-94fd47554-flzfd     1/1     Running   0              29m
minio-state-0             1/1     Running   0              28s
...

$ k get sts
NAME          READY   AGE
minio-state   1/1     93s
```

# Задачи

Создаем Ingress и правила для разных путей в файле
[ingress.yaml](ingress.yaml)
Проверяем, что получаем данные от разных Service
```console
curl 192.168.59.100 > out-home-root.html
curl 192.168.59.100/web/ > out-home-web.html
```
Результат:
[out-home-root.html](out-home-root.html)
[out-home-web.html](out-home-web.html)

Создаем манифест [httpd.yaml](httpd.yaml) для Apache с 
монтированием EmptyDir. Подключаемся к поду и создаем файл
в монтированной директории 
```console
$ k apply -f httpd.yaml
$ k exec -ti httpd-6f77ffd85c-t7fhg -- bash
$ cd /usr/local/apache2/htdocs/
$ cat '<h1>Hello</h1>' > index.html
```

Удаляем под, заходим на вновь созданный и проверяем содержимое
директории
```console
$ k delete pod httpd-6f77ffd85c-t7fhg
$ k exec -ti httpd-6f77ffd85c-7ctvf -- bash
$ ls /usr/local/apache2/htdocs/
```
Выдает пустой каталог. Время жизни mount с типом EmptyDir совпадает
с временем жизни пода, поэтому создается новая сущность, когда
создается новый под

