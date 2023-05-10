# DevOps workshop part I

## Docker container

### Code

```bash
# build image

docker build ./ -t nginx-image

# login to repository

docker login -u {username} -p {password} {host}:{port}

# retag image

docker tag nginx-image {host}:{port}/nginx-image:0.1.0

# push image

docker push {host}:{port}/nginx-image:0.1.0

# run image

docker run --name nginx -d -p 8080:80 nginx-image

# run image and mount container

docker run --name nginx -d -p 8080:80 --volume //c/Users/path/to/folder/with/index.html:/var/www/html nginx-image
```

### Dockerfile

```dockerfile
FROM ubuntu:20.04

RUN apt update
RUN apt -y install nginx

COPY ./test.html /var/www/html/index.html

CMD ["/bin/bash", "-c", "nginx -g \"daemon off;\""]
```

## Nodes & Namespaces

### Code

```bash

# get nodes

kubectl get no

# or

kubectl get node

# or

kubectl get nodes

# describe node

kubectl describe no {nodename}

# get namespaces

kubectl get ns

# or... you know ;)

# create ns

kubectl create ns

# get ns manifest

kubectl get ns {namespace} -o yaml

# create namespace with manifest

kubectl apply -f {namespace-manifest-file-name}.yaml
```

### Sample manifest

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: hello
```

## Pods

### Code

```bash
# get pods from ns

kubectl get po -n {namespace}

# from all namespaces

kubectl get po -A

# create pod

kubectl run nginx --image=nginx --restart=Never -n {namespace}

# get manifest

kubectl get po -n {namespace} {pod-name} -o yaml

# apply manifest...

kubectl apply -f {pod-manifest-file-name}.yaml

# describe pod

kubectl describe po -n {namespace} {pod-name}

# get logs

kubectl logs -n {namespace} {pod-name}

# proxy pod to localhost

kubectl port-forward pod/{pod-name} -n {namespace} 80
```

### Sample manifest

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: hello
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

## Deployment

### Code

```bash
# get deployment

kubectl get deploy -n {namespace}

# create deployment

kubectl create deployment nginx --image=nginx --replicas=3 -n {namespace}

# describe deployment, get manifest, create from manifest -- you know; logs are only pod feature don't even try (:
```

### Sample manifest

Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
  namespace: hello
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
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
```

ReplicaSet coresponding to deployment
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  labels:
    app: nginx
    pod-template-hash: 7fb96c846b
  name: nginx-deployment-7fb96c846b
  namespace: hello
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
      pod-template-hash: 7fb96c846b
  template:
    metadata:
      labels:
        app: nginx
        pod-template-hash: 7fb96c846b
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
```

Node affinity
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: gpu-type
            operator: In
            values:
            - nvidia
      - weight: 5
        preference:
          matchExpressions:
          - key: gpu-type
            operator: In
            values:
            - tesla
  containers:
  - name: nginx-node-affinity
    image: nginx
```

Pod affinity
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: storage
spec:
  selector:
    matchLabels:
      app: store
  replicas: 3
  template:
    metadata:
      labels:
        app: store
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - store
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - nginx
            topologyKey: "kubernetes.io/hostname"
      containers:
      - name: storage
        image: nginx
```

## Service

### Code

```bash
# get service

kubectl get svc -n {namespace}

# proxy service to localhost

kubectl port-forward service/{service-name} -n {namespace} 80

# you know the rest...
```

### Sample manifest

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: hello
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80 # service port
      targetPort: 80 # pod port
```

## Ingress

### Code

```bash
# don't forget to deploy ingress controller if it is not deployed already
# https://platform9.com/learn/v1.0/tutorials/nginix-controller-via-yaml

# get ingress

kubectl get ingress -n {namespace}

# you know the rest...
```

### Sample manifest

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ingress-nginx
  namespace: hello
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /my-ingress(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: nginx
            port:
              number: 80
```