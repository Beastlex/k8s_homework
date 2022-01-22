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

Запускаем прокси. Вывод сохраняем в файл output1_1.json
```
$ kubectl proxy
$ curl http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/ > output1_1.json
```

Скриншот из браузера сохраняем в screen_1_1.PNG

![screen_1_1]*(./screen_1_1.PNG)

# Задание 1.2

