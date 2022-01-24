---
title: "Highly Available Kubernetes Cluster using Ubuntu 20.04 with Keepalived and Haproxy"
date: 2022-01-24
categories: [cloud, kubernetes]
tags:
  - Jekyll
  - update
---
<img src="/assets/images/kubernetes_logo.png">

## VM Specification
Berikut adalah spesifikasi VM untuk membangun sebuah kubernetes cluster

<img src="/assets/images/spec.png">

## Virtual IP
192.168.130.100

## Software Version
- kubeadm: 1.22.4-00
- kubelet: 1.22.4-00
- kubectl: 1.22.4-00

## Provisioning
>Run all command on all nodes

### Creating Working Directories
```s
mkdir -p .kube/ workdir/kubernetes/provisioning/ && cd workdir/kubernetes/
```

### Setting up NTS-Secured NTP with NTPsec
Buat file baru `install-ntp.sh` di directory `~/workdir/kubernetes/provisioning` lalu tambahkan konfigurasi berikut
```s
#!/bin/bash
systemctl stop systemd-timesyncd.service
systemctl disable systemd-timesyncd.service
apt-get remove --purge ntp
git clone https://gitlab.com/NTPsec/ntpsec.git
cd ntpsec/
./buildprep
./waf configure --refclock=all
./waf build
./waf install
cd .. && rm -rf $PWD/ntpsec/
mkdir -p /var/lib/ntp/certs /var/log/ntpstats
adduser --system --no-create-home --disabled-login --gecos '' ntp
addgroup --system ntp
addgroup ntp ntp
chown -R ntp:ntp /var/lib/ntp /var/log/ntpstats
systemctl enable ntpd
cat << EOF > /etc/ntp.conf
server time.cloudflare.com iburst nts

# allows a fast frequency error correction on startup
driftfile /var/lib/ntp/ntp.drift

# collect statistics
statsdir /var/log/ntpstats
statistics loopstats peerstats clockstats rawstats
filegen loopstats file loopstats type day enable
filegen peerstats file peerstats type day enable
filegen clockstats file clockstats type day enable
filegen rawstats file rawstats type day enable

# logging
logfile /var/log/ntp.log
logconfig =syncall +clockall +peerall +sysall
EOF

systemctl start ntpd
date
timedatectl set-timezone Asia/Makassar
date
```

Ubah permission menjadi execute
```s
chmod +x install-ntp.sh
./install-ntp.sh
```

## Set up Load Balancer 

>Run all command on sh-lb-01 and sh-lb-02

### Install Keepalived & Haproxy

```s
apt update && apt install -y keepalived haproxy
```

### Configure Keepalived

Di kedua node buat health check script di `/etc/keepalived/check_apiserver.sh`
```s
#!/bin/sh

errorExit() {
  echo "*** $@" 1>&2
  exit 1
}

curl --silent --max-time 2 --insecure https://localhost:6443/ -o /dev/null || errorExit "Error GET https://localhost:6443/"
if ip addr | grep -q 192.168.130.100; then
  curl --silent --max-time 2 --insecure https://192.168.130.100:6443/ -o /dev/null || errorExit "Error GET https://192.168.130.100:6443/"
fi
```
Ubah permission menjadi execute
```s
chmod +x /etc/keepalived/check_apiserver.sh
./check_apiserver.sh
```

Buat keepalived config di `/etc/keepalived/keepalived.conf`
```s
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  timeout 10
  fall 5
  rise 2
  weight -2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens33
    virtual_router_id 1
    priority 100
    advert_int 5
    authentication {
        auth_type PASS
        auth_pass secret123
    }
    virtual_ipaddress {
        192.168.130.100
    }
    track_script {
        check_apiserver
    }
}
```
Enable & start keepalived service
```s
systemctl enable --now keepalived
```

### Configure Haproxy

Tambahkan konfigurasi berikut di `/etc/haproxy/haproxy.cfg`
```s
frontend kubernetes-frontend
  bind *:6443
  mode tcp
  option tcplog
  default_backend kubernetes-backend

backend kubernetes-backend
  option httpchk GET /healthz
  http-check expect status 200
  mode tcp
  option ssl-hello-chk
  balance roundrobin
    server sh-master-01 192.168.130.101:6443 check fall 3 rise 2
    server sh-master-02 192.168.130.102:6443 check fall 3 rise 2
    server sh-master-03 192.168.130.103:6443 check fall 3 rise 2
```

Enable & restart haproxy service
```s
systemctl enable haproxy && systemctl restart haproxy
```

## Install Kubernetes Component

>Run all command on all master node and worker node

### Pre-requisites

Disable swap agar kubelet dapat berjalan dengan baik
```s
swapoff -a; sed -i '/swap/d' /etc/fstab
```

Tambahkan Kernel settings
```s
cat <<EOF | tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```

### Container Runtime Interface (CRI)

Buat file baru `install-docker.sh` di directory `~/workdir/kubernetes/provisioning/` dan
copy script berikut
```s
#!/bin/bash
# Install Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
echo "deb https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list
apt-get update && apt-get install docker-ce docker-ce-cli containerd.io -y

# Configure Cgroup
mkdir -p /etc/docker/
cat <<EOF | tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

# Restart Docker & Enable on boot
systemctl enable docker
systemctl daemon-reload
systemctl restart docker
```

Ubah permission menjadi execute
```s
chmod +x install-docker.sh
./install-docker.sh
```

Tambahkan repo kubernetes
```s
curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | tee /etc/apt/sources.list.d/kubernetes.list
```

Install Kubernetes components dan disable update package
```s
apt-get update && apt-get install -y kubelet=1.22.4-00 kubeadm=1.22.4-00 kubectl=1.22.4-00
apt-mark hold kubelet kubeadm kubectl
```

## Bootstrap the Cluster

### Initialize the First Control Plane on sh-master-01
```s
kubeadm init --control-plane-endpoint "192.168.130.100:6443" --upload-certs
```

Berikut adalah hasil bootstrap yang di gunakan untuk join ke master nodes and worker nodes
```s
You can now join any number of the control-plane node running the following command on each as root:
  kubeadm join 192.168.130.101:6443 --token m6f7v9.at32n1bcvic6pokz \
        --discovery-token-ca-cert-hash sha256:cfe25f1334dbb8dc09ebb7c6662d482d9d82c765f6fd31f3854d5fb391262107 \
        --control-plane --certificate-key fe817adde19431b4aab3afc462778a75393c5c914237a5d388a76f8ec1e233e9

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.130.101:6443 --token m6f7v9.at32n1bcvic6pokz \
        --discovery-token-ca-cert-hash sha256:cfe25f1334dbb8dc09ebb7c6662d482d9d82c765f6fd31f3854d5fb391262107
```

Copy kubectl config file
```s
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

Deploy Calico network
```s
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### Add New Node to The Cluster
#### Master Node
Copy perintah yang diberikan sebelumnya dan jalankan di sh-master-02 and sh-master-03
```s
kubeadm join 192.168.130.101:6443 --token m6f7v9.at32n1bcvic6pokz \
        --discovery-token-ca-cert-hash sha256:cfe25f1334dbb8dc09ebb7c6662d482d9d82c765f6fd31f3854d5fb391262107 \
        --control-plane --certificate-key fe817adde19431b4aab3afc462778a75393c5c914237a5d388a76f8ec1e233e9
```

#### Worker Node
Copy perintah yang diberikan sebelumnya dan jalankan di semua worker node
```s
kubeadm join 192.168.130.101:6443 --token m6f7v9.at32n1bcvic6pokz \
        --discovery-token-ca-cert-hash sha256:cfe25f1334dbb8dc09ebb7c6662d482d9d82c765f6fd31f3854d5fb391262107
```

Verifikasi cluster
```s
kubectl cluster-info
kubectl get nodes
```

**Reference:**
* [Set up a Highly Available Kubernetes Cluster using kubeadm](https://github.com/justmeandopensource/kubernetes/tree/master/kubeadm-ha-keepalived-haproxy/external-keepalived-haproxy)

