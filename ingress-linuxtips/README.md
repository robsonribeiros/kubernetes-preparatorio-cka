# Ingress

## O que é o Ingress

Normalmente quando executamos um Pod no Kubernetes, todo o tráfego é roteado somente pela rede do cluster, e todo tráfego externo acaba sendo descartado ou encaminhado para outro local. Um ingress é um conjunto de regras para permitir que as conexões externas de entrada atinjam os serviços dentro do cluster

Vamos criar nosso primeiro Ingress, mas primeiro vamos gerar dois deployments e dois services:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - image: dockersamples/static-site
        name: app1
        env:
        - name: AUTHOR
          value: GIROPOPS
        ports:
        - containerPort: 80

```

Config map
```bash
kubectl create configmap cmap-app1 --from-file index.html=index-app1.html 
kubectl create configmap cmap-app2 --from-file index.html=index-app2.html 
#kubectl create configmap apps-configmap --from-file app1=index-app1.html --from-file app2=index-app2.html
```

Atenção para os seguintes parâmetros no arquivo anterior:

terminationGracePeriodSeconds => Tempo em segundos que ele irá aguardar o pod ser finalizado com o sinal SIGTERM, antes de realizar a finalização forçada com o sinal de SIGKILL.
livenessProbe => Verifica se o pod continua em execução, caso não esteja, o kubelet irá remover o contêiner e iniciará outro em seu lugar.
readnessProbe => Verifica se o container está pronto para receber requisições vindas do service.
initialDelaySeconds => Diz ao kubelet quantos segundos ele deverá aguardar para realizar a execução da primeira checagem da livenessProbe
timeoutSeconds => Tempo em segundos que será considerado o timeout da execução da probe, o valor padrão é 1.
periodSeconds => Determina de quanto em quanto tempo será realizada a verificação do livenessProbe.

```bash
kubectl create namespace ingress

namespace/ingress created

```
Crie o deployment do backend no namespace ingress:
```
kubectl create -f default-backend.yaml -n ingress 

deployment.apps/default-backend created
```

Crie um arquivo para definir um service para o backend:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: default-backend
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
  selector:
    app: default-backend
```


Crie o service para o backend no namespace ingress:
```bash
kubectl create -f default-backend-service.yaml -n ingress 

service/default-backend created
```

## Configurações do ingress

Agora crie o um arquivo para definir um configMap a ser utilizado pela nossa aplicação:
```bash
vim nginx-ingress-controller-config-map.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-ingress-controller-conf
  namespace: ingress
  labels:
    app: nginx-ingress-lb
data:
  enable-vts-status: 'true'
```
Crie o configMap no namespace ingress:
```bash
kubectl create -f nginx-ingress-controller-config-map.yaml -n ingress

configmap/nginx-ingress-controller-conf created


kubectl get configmaps -n ingress


```

## Definindo as permissões do ingress


Parte 03
https://school.linuxtips.io/courses/1259521/lectures/28043179

```
vim nginx-ingress-controller-service-account.yaml
```


Vamos criar os arquivos para definir as permissões para o nosso deployment:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nginx
  namespace: ingress
```

```bash
vim nginx-ingress-controller-clusterrole.yaml
```

```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nginx-role
rules:
- apiGroups:
  - ""
  - "extensions"
  resources:
  - configmaps
  - secrets
  - endpoints
  - ingresses
  - nodes
  - pods
  verbs:
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - services
  verbs:
  - list
  - watch
  - get
  - update
- apiGroups:
  - "extensions"
  resources:
  - ingresses
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
- apiGroups:
  - "extensions"
  resources:
  - ingresses/status
  verbs:
  - update
- apiGroups:
  - ""
  resources:
  - configmaps
  verbs:
  - get
  - create
```

```bash
vim nginx-ingress-controller-clusterrolebinding.yaml
```

```yaml
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nginx-role
  namespace: ingress
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: nginx-role
subjects:
- kind: ServiceAccount
  name: nginx
  namespace: ingress
```

```bash
vim nginx-ingress-controller-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ingress-lb
  revisionHistoryLimit: 3
  template:
    metadata:
      labels:
        app: nginx-ingress-lb
    spec:
      terminationGracePeriodSeconds: 60
      serviceAccount: nginx
      containers:
        - name: nginx-ingress-controller
          image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.9.0
          imagePullPolicy: Always
          readinessProbe:
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
          livenessProbe:
            httpGet:
              path: /healthz
              port: 10254
              scheme: HTTP
            initialDelaySeconds: 10
            timeoutSeconds: 5
          args:
            - /nginx-ingress-controller
            - --default-backend-service=ingress/default-backend
            - --configmap=ingress/nginx-ingress-controller-conf
            - --v=2
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - containerPort: 80
            - containerPort: 18080
```


https://school.linuxtips.io/courses/1259521/lectures/28043179

13 minutos


```bash

```

```yaml

```

```
https://github.com/badtuxx/ingress

kubectl create -f app-deployment.yaml
kubectl create -f app-service.yaml
kubectl get pods
kubectl get ep
kubectl create namespace ingress
kubectl create -f default-backend-deployment.yaml -n ingress
kubectl create -f default-backend-service.yaml -n ingress
kubectl get deployments. -n ingress
kubectl get service
kubectl get service -n ingress
kubectl create -f nginx-ingress-controller-config-map.yaml -n ingress
kubectl create -f nginx-ingress-controller-roles.yaml -n ingress
kubectl create -f nginx-ingress-controller-deployment.yaml -n ingress
kubectl create -f nginx-ingress-controller-service.yaml -n=ingress
kubectl get pods
kubectl get pods -n ingress
kubectl create -f nginx-ingress.yaml -n ingress
kubectl create -f app-ingress.yaml
kubectl get pods -n ingress
kubectl get pods
kubectl get services
kubectl get services -n ingress
kubectl get ingress
kubectl get deploy
kubectl get deploy -n ingress
```


kubectl delete -f app-deployment.yaml
kubectl delete -f app-service.yaml
kubectl delete -f default-backend-deployment.yaml -n ingress
kubectl delete -f default-backend-service.yaml -n ingress
kubectl delete -f nginx-ingress-controller-config-map.yaml -n ingress
kubectl delete -f nginx-ingress-controller-roles.yaml -n ingress
kubectl delete -f nginx-ingress-controller-deployment.yaml -n ingress
kubectl delete -f nginx-ingress-controller-service.yaml -n=ingress
kubectl delete -f nginx-ingress.yaml -n ingress
kubectl delete -f app-ingress.yaml