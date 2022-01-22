# Задание 1.1

Проверяем версию kubectl 
```console
$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"23", GitVersion:"v1.23.2", GitCommit:"9d142434e3af351a628bffee3939e64c681afa4d", GitTreeState:"clean", BuildDate:"2022-01-19T17:35:46Z", GoVersion:"go1.17.5", Compiler:"gc", Platform:"linux/amd64"}
```
Запускаем minikube
```console
$ source <(kubectl completion zsh)
$ minikube start --driver=virtualbox
minikube v1.25.1 on Ubuntu 20.04
... 
* Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
```

Получаем информацию о нодах
```console
$ kubectl get nodes
NAME       STATUS   ROLES                  AGE     VERSION
minikube   Ready    control-plane,master   4m10s   v1.23.1
```

Устанавливаем Kubernetes Dashboard
```console
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.3.1/aio/deploy/recommended.yaml
$ kubectl get pod -n kubernetes-dashboard
NAME                                         READY   STATUS    RESTARTS   AGE
dashboard-metrics-scraper-577dc49767-kzjml   1/1     Running   0          53s
kubernetes-dashboard-78f9d9744f-h8g6l        1/1     Running   0          53s
```

Устанавливаем Metrics Server и редактируем Deployment
```console
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
$ kubectl edit -n kube-system deployment metrics-server
deployment.apps/metrics-server edited
```

Присоединяемся к Dashboard. Проверяем токен
```console
$ kubectl describe sa -n kube-system default
...
Tokens:              default-token-4rttv
...
$ kubectl get secrets -n kube-system default-token-4rttv -o yaml
...
type: kubernetes.io/service-account-token
$ export A="token form prev step"
$ echo -n $A | base64 -d
... копируем токен в буфер
$ kubectl proxy
```

Запускаем прокси. Вывод сохраняем в файл output11.json
```
$ kubectl proxy
$ curl http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/ > output11.json
```

Скриншот из браузера сохраняем в [screen11.png](https://raw.githubusercontent.com/Beastlex/k8s_homework/main/task01/screen11.png)


# Задание 1.2

Запускаем под
```console
$ kubectl run web --image=nginx:latest
```

Смотрим запущенные поды
```console
$ kubectl get pods
NAME   READY   STATUS    RESTARTS   AGE
web    1/1     Running   0          88s
```

Сотрим в дашборде
[screen12-1.png](https://raw.githubusercontent.com/Beastlex/k8s_homework/main/task01/screen12-1.png)

Заходим в ноду в кластере и смотрим контейнеры запущенные
```console
$ minikube ssh
$ docker container ls
CONTAINER ID   IMAGE                    COMMAND                  CREATED          STATUS          PORTS     NAMES
ccb142c9558f   nginx                    "/docker-entrypoint.…"   14 minutes ago   Up 14 minutes             k8s_web_web_default_9b0c638e-1c16-47e8-ad5a-3769556f048b_0
...
```

Запускаем манифесты и смотрим поды в кластере
```console
$ kubectl apply -f pod.yaml
$ kubectl apply -f rs.yaml

$ kubectl get pod
NAME               READY   STATUS    RESTARTS   AGE
nginx              1/1     Running   0          110s
web                1/1     Running   0          21m
webreplica-lg572   1/1     Running   0          103s
```
Создаем манифест, запустив dry-run 
```console
$ kubectl create deployment nginx2 --image=nginx:latest --replicas=2 --dry-run=client -o yaml > nginx2.yaml
```

Запускаем манифест и смотрим поды
```console
$ kubectl apply -f nginx2.yaml
$ kubectl get pod
NAME                      READY   STATUS    RESTARTS   AGE
nginx                     1/1     Running   0          13m
nginx2-54c7bd9d87-4tn76   1/1     Running   0          14s
nginx2-54c7bd9d87-d6g9x   1/1     Running   0          14s
web                       1/1     Running   0          33m
webreplica-lg572          1/1     Running   0          13m
```

Удаляем один под и видим, что создается новый
```console
$ kubectl delete pod nginx2-54c7bd9d87-4tn76
$ kubectl get pod
NAME                      READY   STATUS    RESTARTS   AGE
nginx                     1/1     Running   0          15m
nginx2-54c7bd9d87-4vmcs   1/1     Running   0          5s
nginx2-54c7bd9d87-d6g9x   1/1     Running   0          2m6s
web                       1/1     Running   0          35m
webreplica-lg572          1/1     Running   0          15m
```
