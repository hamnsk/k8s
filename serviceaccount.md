# Создаем пользователя с доступом только в неймспейс (пользователи)

## 1. Переменные среды
Для того чтобы нам было удобно работать с нашими скриптами и командами определим ряд переменных среды:
укажем имя пользователя, неймспейс, и название роли которую будем создавать

```shell script
$ ACCOUNT_NAME=petya
$ NAMESPACE=netology
$ ROLENAME=read-exec-pods-svc-ing
```

Создадим неймспейс

```shell script
$ kubectl create ns $NAMESPACE
```



## 2. Создаем конфиг для подключение к кластеру

Создадим сервисаккаунт

```shell script
$ kubectl create serviceaccount $ACCOUNT_NAME --namespace $NAMESPACE
```
 
Подготовим дополнительные переменные среды, запускаем оманды последовательно

```shell script
$ TOKEN_NAME=$(kubectl get serviceAccounts $ACCOUNT_NAME --namespace $NAMESPACE  -o jsonpath="{.secrets[0].name}")
$ TOKEN=$(kubectl describe secrets $TOKEN_NAME --namespace $NAMESPACE | grep 'token:' | rev | cut -d ' ' -f1 | rev)
$ CERTIFICATE_AUTHORITY_DATA=$(kubectl config view --flatten --minify -o jsonpath="{.clusters[0].cluster.certificate-authority-data}")
$ SERVER_URL=$(kubectl config view --flatten --minify -o jsonpath="{.clusters[0].cluster.server}")
$ CLUSTER_NAME=$(kubectl config view --flatten --minify -o jsonpath="{.clusters[0].name}")
```

Запишем итоговый конфиг в файл
```shell script
$ cat <<EOF > $CLUSTER_NAME-$ACCOUNT_NAME-kube.conf
apiVersion: v1
kind: Config
users:
- name: $ACCOUNT_NAME
  user:
    token: $TOKEN
clusters:
- cluster:
    certificate-authority-data: $CERTIFICATE_AUTHORITY_DATA    
    server: $SERVER_URL
  name: $CLUSTER_NAME
contexts:
- context:
    cluster: $CLUSTER_NAME
    user: $ACCOUNT_NAME
  name: $CLUSTER_NAME-$ACCOUNT_NAME-context
current-context: $CLUSTER_NAME-$ACCOUNT_NAME-context
EOF
```

Проверим появился ли у нас доступ к нашему кластеру

```shell script
$ kubectl --kubeconfig=$CLUSTER_NAME-$ACCOUNT_NAME-kube.conf get po -n $NAMESPACE
```
Мы получим ошибку, потому что у нас нет роли и связки этой роли с сервисаккаунтом

> Error from server (Forbidden): pods is forbidden: User "system:serviceaccount:firstnamespace:test-service-account" cannot list resource "pods" in API group "" in the namespace "firstnamespace"


## 3. Создаем роль

```shell script
$ cat <<EOF > $ROLENAME-role.yaml ; kubectl apply -f $ROLENAME-role.yaml -n $NAMESPACE
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: $NAMESPACE
  name: $ROLENAME
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "services", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "describe"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]
EOF
```

## 4. Создаем РольБиндинг и связываем нашу роль с сервисаккаунтом

```shell script
$ cat <<EOF > $ROLENAME-rolebinding.yaml ; kubectl apply -f $ROLENAME-rolebinding.yaml -n $NAMESPACE
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: $ACCOUNT_NAME-$ROLENAME-rolebinding
  namespace: $NAMESPACE
subjects:
- kind: User
  name: system:serviceaccount:$NAMESPACE:$ACCOUNT_NAME # Name is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: $ROLENAME # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
EOF
```

## 5. Тестируем доступ в кластер

```shell script
$ kubectl --kubeconfig=$CLUSTER_NAME-$ACCOUNT_NAME-kube.conf get po -n $NAMESPACE
```

Готово!

---

# Создаем кластерную роль, достук ко всем неймспейсам (админы)

## 1. Переменные среды

```shell script
$ ACCOUNT_NAME=vasya
$ ROLENAME=read-exec-pods-svc-ing-global
```

## 2. Создаем конфиг для подключение к кластеру

Создадим сервисаккаунт

```shell script
$ kubectl create serviceaccount $ACCOUNT_NAME
```
 
Подготовим дополнительные переменные среды, запускаем оманды последовательно


```shell script
$ TOKEN_NAME=$(kubectl get serviceAccounts $ACCOUNT_NAME  -o jsonpath="{.secrets[0].name}")
$ TOKEN=$(kubectl describe secrets $TOKEN_NAME  | grep 'token:' | rev | cut -d ' ' -f1 | rev)
$ CERTIFICATE_AUTHORITY_DATA=$(kubectl config view --flatten --minify -o jsonpath="{.clusters[0].cluster.certificate-authority-data}")
$ SERVER_URL=$(kubectl config view --flatten --minify -o jsonpath="{.clusters[0].cluster.server}")
$ CLUSTER_NAME=$(kubectl config view --flatten --minify -o jsonpath="{.clusters[0].name}")
```
Запишем итоговый конфиг в файл
```shell script
cat <<EOF > $CLUSTER_NAME-$ACCOUNT_NAME-kube.conf
apiVersion: v1
kind: Config
users:
- name: $ACCOUNT_NAME
  user:
    token: $TOKEN
clusters:
- cluster:
    certificate-authority-data: $CERTIFICATE_AUTHORITY_DATA    
    server: $SERVER_URL
  name: $CLUSTER_NAME
contexts:
- context:
    cluster: $CLUSTER_NAME
    user: $ACCOUNT_NAME
  name: $CLUSTER_NAME-$ACCOUNT_NAME-context
current-context: $CLUSTER_NAME-$ACCOUNT_NAME-context
EOF
```

## 3. Создаем кластерную роль

```shell script
cat <<EOF > $ROLENAME-role.yaml ; kubectl apply -f $ROLENAME-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  namespace: $NAMESPACE
  name: $ROLENAME
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log", "services", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "describe"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]
EOF
```

## 4. Создаем КластерРольБиндинг и связываем нашу кластерную роль с сервисаккаунтом

```shell script
$ cat <<EOF > $ROLENAME-rolebinding.yaml ; kubectl apply -f $ROLENAME-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: $ACCOUNT_NAME-$ROLENAME-rolebinding
subjects:
- kind: User
  name: system:serviceaccount:default:$ACCOUNT_NAME # default it is namaspase
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole #this must be Role or ClusterRole
  name: $ROLENAME # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
EOF
```


## 5. Тестируем доступ в кластер

```shell script
$ kubectl --kubeconfig=$CLUSTER_NAME-$ACCOUNT_NAME-kube.conf get po -A
```
