---
title: "Nginx Ingress Controller"
date: 2022-01-24
categories: [cloud, kubernetes]
tags:
  - Jekyll
  - update
---

Karena service load balancer hanya tersedia di cloud provider saja, ada salah satu solusi untuk membuat service load balancer yang bisa di terapkan di baremetal, yaitu dengan menggunakan MetalLB.

## Creating Working Directories Ingress 
Buat directory untuk ingress, agar semua file konfigurasi terorganisir di dalam satu directory.

```s
mkdir ~/ingress
cd ~/ingress
```

## Create 2 deployments
Buat 2 deployment dengan image nginx dan httpd. Kemudian expose service dengan menggunakan ClusterIP. Service ClusterIP hanya bisa di akses di dalam cluster saja, oleh karena itu nantinya kita akan membuat ingress, untuk mengekspose service yang ada di dalam cluster agar bisa di akses dari external.

```
kubectl create deployment web-nginx --image nginx
kubectl expose deployment web-nginx --port 80
kubectl create deployment web-apache --image httpd
kubectl expose deployment web-apache --port 80
```

## Deploy Nginx Ingress Controller for kubeadm/baremetal
```s
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/deploy.yaml
kubectl apply -f deploy.yaml
```

## Verify Nginx Ingress Controller
```
kubectl get pod -n ingress-nginx --watch
kubectl -n ingress-nginx get service
```

## Create ingress rewrite

```s
nano rewrite.yaml
...
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$1
	  kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: web-nginx.ok
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-nginx
                port:
                  number: 80
    - host: web-apache.ok
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-apache
                port:
                  number: 80
...
kubectl create -f rewrite.yaml
kubectl get svc --all-namespaces
```

## Add externalIP

```s
nano external-ips.yaml
...
spec:
  externalIPs:
  - 192.168.130.101 # ip master node
...
kubectl -n ingress-nginx patch svc ingress-nginx-controller --patch "$(cat external-ips.yaml)"
kubectl get svc  ingress-nginx-controller -n ingress-ngin
```

## Add host
```s
nano /etc/hosts
...
192.168.130.101 web-nginx.ok web-apache.ok
...
```

Verifikasi
```s
kubectl get ingress
curl web-nginx
curl web-apache
```

**References:**
* [https://metallb.universe.tf/installation/](https://metallb.universe.tf/installation/)

