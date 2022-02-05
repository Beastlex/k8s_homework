# Задание 4

Проверяем свой уровень доступа
```console
$ kubectl auth can-i create deployments --namespace kube-system
yes
```

Создаем самоподписной сертифика
```console
$ openssl genrsa -out k8s_user.key 2048
$ openssl req -new -key k8s_user.key \
  -out k8s_user.csr \
  -subj "/CN=k8s_user"
$ openssl x509 -req -in k8s_user.csr \
  -CA ~/.minikube/ca.crt \
  -CAkey ~/.minikube/ca.key \
  -CAcreateserial \
  -out k8s_user.crt -days 500
```

Создаем польльзователя и задаем контекст
```console
$ kubectl config set-credentials k8s_user \
  --client-certificate=k8s_user.crt \
  --client-key=k8s_user.key
$ kubectl config set-context k8s_user \
  --cluster=minikube --user=k8s_user
```

Редактируем файл ~/.kube/config по образцу, переключаемся на контекст
и проверяем доступ
```console
$ kubectl config use-context k8s_user
Switched to context "k8s_user".
$ kubectl get node
Error from server (Forbidden): nodes is forbidden: User "k8s_user" cannot list resource "nodes" in API group "" at the cluster scope
$ kubectl get pod
Error from server (Forbidden): pods is forbidden: User "k8s_user" cannot list resource "pods" in API group "" in the namespace "default"
```

Переключаемся на контекст по умолчанию, применяем role binding и
проеряем результат
```console
$ kubectl config use-context minikube
Switched to context "minikube".

$ kubectl apply -f binding.yaml
rolebinding.rbac.authorization.k8s.io/k8s_user created

$ kubectl get pod
NAME                      READY   STATUS    RESTARTS        AGE
dnsutils                  1/1     Running   123 (27m ago)   11d
httpd-6f77ffd85c-7ctvf    1/1     Running   1               10d
minio-94fd47554-flzfd     1/1     Running   1               10d
minio-state-0             1/1     Running   1               10d
nginx                     1/1     Running   1               12d
...
```

# Задание
Создаем serviceaccount
```console
$ kubectl create serviceaccount deploy-view
serviceaccount/deploy-view created

$ kubectl create serviceaccount deploy-edit
serviceaccount/deploy-edit created
```

Создаем cluster role и связываем аккаунты с ролями
```console 
$ kubectl create clusterrole deploy-view --verb=get --verb=list --verb=watch --resource=deployments,pods
clusterrole.rbac.authorization.k8s.io/deploy-view created

$ kubectl create clusterrole deploy-edit --verb=get,list,watch,create,update,patch,delete --resource=deployments,pods
clusterrole.rbac.authorization.k8s.io/deploy-edit created

$ kubectl create clusterrolebinding deploy-view --serviceaccount=default:deploy-view --clusterrole=deploy-view
$ kubectl create clusterrolebinding deploy-edit --serviceaccount=default:deploy-edit --clusterrole=deploy-edit
```

Создаем контексты и привязываем их к пользователям
```
$ TOKEN=$(kubectl describe secrets "$(kubectl describe serviceaccount deploy-view | grep -i Tokens | awk '{print $2}')" | grep token: | awk '{print $2}')
$ kubectl config set-credentials deploy_view --token=$TOKEN
User "deploy_view" set.

$ TOKEN=$(kubectl describe secrets "$(kubectl describe serviceaccount deploy-edit | grep -i Tokens | awk '{print $2}')" | grep token: | awk '{print $2}')
$ kubectl config set-credentials deploy_edit --token=$TOKEN
User "deploy_edit" set.
```

Проверяем контексты для пользователя
```console
$ kubectl config set-context deploy_view --cluster=minikube --user=deploy_view
$ kubectl config use-context deploy_view
$ kubectl get pods
NAME                      READY   STATUS    RESTARTS        AGE
dnsutils                  1/1     Running   124 (19m ago)   12d
httpd-6f77ffd85c-7ctvf    1/1     Running   1               10d
minio-94fd47554-flzfd     1/1     Running   1               10d
...
```

Создаем serviceaccount и namespace
```console
$ kubectl create namespace prod
namespace/prod created
$ kubectl create serviceaccount prod-admin
serviceaccount/prod-admin created
$ kubectl create serviceaccount prod-view
serviceaccount/prod-view created
```

Создаем rolebinding с ролями по умолчанию
```console
$ kubectl create clusterrolebinding prod-admin --serviceaccount=default:prod-admin --clusterrole=admin --namespace prod
clusterrolebinding.rbac.authorization.k8s.io/prod-admin created
$ kubectl create clusterrolebinding prod-view --serviceaccount=default:prod-view --clusterrole=view --namespace prod
clusterrolebinding.rbac.authorization.k8s.io/prod-view created
```

Аналогично привязываем контекст к пользователям и проверяем права

Создаем serviceaccount sa-namespace-admin
```console
$ kubectl create serviceaccount sa-namespace-admin
serviceaccount/sa-namespace-admin created
```

Создаем привязку к роли по умолчанию admin
```console
$ kubectl create clusterrolebinding sa-namespace --serviceaccount=default:sa-namespace-admin --clusterrole=admin --namespace default
clusterrolebinding.rbac.authorization.k8s.io/sa-namespace created
```

Создаем контект и провреяем доступы
```console
$ TOKEN=$(kubectl describe secrets "$(kubectl describe serviceaccount sa-namespace-admin | grep -i Tokens | awk '{print $2}')" | grep token: | awk '{print $2}')
$ kubectl config set-credentials sa-namespace-admin --token=$TOKEN
$ kubectl config set-context sa-namespace-admin --cluster=minikube --user=sa-namespace-admin
$ kubectl config use-context sa-namespace-admin
```

Проверяем права доступа
```console
$ kubectl auth can-i create pods
yes
$ kubectl auth can-i create deploy
yes
```

