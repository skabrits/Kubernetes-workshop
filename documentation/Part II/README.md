# DevOps workshop part II

## Configmap

### Code

```bash
# create configmap from literal values

kubectl create configmap proxy-var --from-literal=http_proxy=127.0.0.1

# create configmap with ready-to-use environment variables

kubectl apply -f <path to env-vars.yaml> # see in sample manifests section

# create configmap from file

kubectl create configmap my-html-file --from-file=index.html=<path to index.html> # see in sample manifests section

# run nginx image with mounted configmaps

kubectl apply -f <path to pod.yaml> # see in sample manifests section

# check env variables

kubectl exec nginx-mounted -- printenv

# see mounted file

kubectl port-forward pod/nginx-mounted 80 # open http://127.0.0.1:80 in browser
```

### Sample manifests

Configmap proxy-var
```yaml
apiVersion: v1
data:
  http_proxy: 127.0.0.1
kind: ConfigMap
metadata:
  name: proxy-var
```

Configmap env-vars
env-vars.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-vars
  namespace: default
data:
  LOG_LEVEL: debug
  LOG_PATH: /var/log/log_file
```

Html page
index.html
```html
<p>DevOps workshop, part II</p>
<p>This is multiline file</p>
```

Configmap my-html-file
```yaml
apiVersion: v1
data:
  index.html: |
    <p>DevOps workshop, part II</p>
    <p>This is multiline file</p>
kind: ConfigMap
metadata:
  name: my-html-file
```

Nginx pod
pod.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-mounted
spec:
  containers:
    - name: nginx
      image: nginx
      env:
        - name: HTTP_PROXY
          valueFrom:
            configMapKeyRef:
              name: proxy-var
              key: http_proxy
              optional: true # mark the variable as optional
      envFrom:
        - configMapRef:
            name: env-vars
      volumeMounts:
      - name: config-volume
        mountPath: /usr/share/nginx/html
  volumes:
    - name: config-volume
      configMap:
        name: my-html-file
```

## Secrets

### Code

```bash
# image pull secret

kubectl create secret docker-registry my-registry-secret \
  --docker-email=<email> \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-server=<registry>

# opaque secret

kubectl create secret generic key-var --from-literal=key_val=testuser123

# opaque secret from file

kubectl create secret generic my-secret-html-file --from-file=<path to index.html>

# test pod manifest is located in Sample manifests section
```

### Sample manifests

Nginx pod
pod.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-secret-mounted
spec:
  imagePullSecrets:
    - name: my-registry-secret
  containers:
    - name: nginx
      image: nginx
      env:
        - name: SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: key-var
              key: key_val
      volumeMounts:
      - name: secret-volume
        mountPath: /usr/share/nginx/html
        readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: my-secret-html-file
        optional: true
```

## PVC/PV

### Code

```bash
# apply manifests from sample manifests section

# use comand to exec to pod

kubectl exec -it pod-pv -- bash

# create two files with hello world

echo "Hello world!" > /testfile.txt
echo "Hello world!" > /myPV/testfile.txt

# exit interective shell and delete pod

kubectl delete pod-pv

# create pod again and check which file exists
```

### Sample manifests

[local-path](https://github.com/rancher/local-path-provisioner) provisioner and storageclass

```bash
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.24/deploy/local-path-storage.yaml
```

PVC
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-claim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-path
```

Pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-pv
spec:
  containers:
  - name: editor
    image: python:3.7
    tty: true
    volumeMounts:
        - mountPath: "/myPV"
          name: test-storage
  volumes:
  - name: test-storage
    persistentVolumeClaim:
      claimName: test-claim
```

## Stateful set

### Code

```bash
# apply manifests from sample manifests section

# now inside cluster pods are available at web-{0..2}.nginx.default.svc.cluster.local
```

### Sample manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3
  podManagementPolicy: OrderedReady # alternative mode - Parallel
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
        env:
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "local-path"
      resources:
        requests:
          storage: 1Gi
```

## Helm

### Code

```bash
# create hello world app

helm create {chart name}

# update dependencies

helm dependency update {chart name}

# build dependencies

helm dependency build {chart name}

# template locally

helm template {name for app in cluster} {chart name}

# package chart

helm package {chart name}

# add repo

helm repo add {repo NAME} {repo URL}

# login to registry

helm registry login {repo URL} -p {password} -u {username}

# push

helm push {path to package} {repo NAME}

# update latest charts from repo

helm repo update {repo NAME}

# install chart

helm install {name for app in cluster} {path to chart folder} --namespace {app namespace} --create-namespace -f override-values.yaml --set service.type=LoadBalancer

# or

helm install {name for app in cluster} {repo NAME}/{chart name} --namespace {app namespace} --create-namespace -f override-values.yaml --set service.type=LoadBalancer

# or

helm upgrade --install {name for app in cluster} {repo NAME}/{chart name} --namespace {app namespace} --create-namespace -f override-values.yaml --set service.type=LoadBalancer

# get values from deployed chart

helm get values {name for app in cluster} -n {app namespace}
```

### Sample manifest

override-values.yaml
```yaml
replicaCount: 2
service:
  type: NodePort
```

## Extras: debugging

### Code

```bash
# if resource fails to work -- try describing and scroll to events section

kubectl describe pod -n my-app nginx

kubectl describe svc -n my-app nginx-svc

kubectl describe ing -n my-app nginx-ing

# if pod fails after launching container try seeing logs

kubectl logs pod -n my-app nginx

# watch for the matching selectors and labels across different resources

# exec into pod for deeper debugging, including connectivity tests

kubectl exec -it -n my-app nginx -- bash

# when cluster is running behind the proxy it is important to have certificates installed and proxy settings set inside each pod if pod needs to connect somewhere outside of the cluster

# check twice to see if communication across nodes has no obstacles

# to edit resource manifest on-the-go use kubectl edit command:
kubectl edit deploy -n my-app nginx-deploy
kubectl edit svc -n my-app nginx-svc
kubectl edit ing -n my-app nginx-ing

# after changing configmap or secret don't forget to restart deployment
kubectl rollout restart deploy -n my-app nginx-deploy
```
