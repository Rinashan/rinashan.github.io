---
title: "Create Load Balancer Service using MetalLB"
date: 2022-01-24
categories: [cloud, kubernetes]
tags:
  - Jekyll
  - update
---
<img src="/assets/images/metallb-logo.png">

Karena service load balancer hanya tersedia di cloud provider saja, ada salah satu solusi untuk membuat service load balancer yang bisa di terapkan di baremetal, yaitu dengan menggunakan MetalLB.

## Installation Metallb
```s
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.11.0/manifests/metallb.yaml
```

## Define and Deploy configmap
```s
nano metallb.yaml
. . .
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.130.200-192.168.130.250
. . .
kubectl apply -f metallb.yaml
```

## Exposing a Service through the Load Balancer

Buat deployment nginx dan buat juga service load balancer

```s
nano nginx-deployment.yaml
...
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    name: http
...
kubectl apply -f nginx-deployment
```

Verifikasi 
```s
kubectl get svc
kubectl get pods
curl http://<nginx-service-external-ip>
```

**Reference:**
* [https://metallb.universe.tf/installation/](https://metallb.universe.tf/installation/)

