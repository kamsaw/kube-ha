# Kubernetes Highly Available
- [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [https://www.linuxtechi.com/setup-highly-available-kubernetes-cluster-kubeadm/](https://www.linuxtechi.com/setup-highly-available-kubernetes-cluster-kubeadm/)

![aaa](https://user-images.githubusercontent.com/38559302/233606228-ee94ec24-b7a9-430d-9a4b-781aae2793cd.jpg)

# preinstall
### => master-n, node-n
```
adduser kube
ssh -> master-n <> master-n
ssh -> master-n > node-n
```
```
apt install htop mc systemd-timesyncd

apt install hyperv-daemons
```
```
nano /etc/hosts
---
192.168.2.1 k8s1-controlplane01
192.168.2.2 k8s1-controlplane02
192.168.2.3 k8s1-node01
192.168.2.4 k8s1-node02
192.168.2.5 k8s1-node03
192.168.2.6 k8s1-controlplane03
192.168.2.7 k8s1-controlplane-vip
```
```
dpkg-reconfigure locales
nano ~/.bashrc
export PATH=$PATH:/usr/sbin
alias k=kubectl

swapoff -a
nano /etc/fstab // komentujemy swap_1
reboot
```
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system

lsmod | grep br_netfilter
lsmod | grep overlay
```
> sprawdzamy czy parametry są ustawione na 1
```
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```
# Keepalive
### => master-n
```
sudo apt install keepalived
```
```
sudo nano /etc/keepalived/check_apiserver.sh

#!/bin/sh
APISERVER_VIP=192.168.2.7
APISERVER_DEST_PORT=6443

errorExit() {
    echo "*** $*" 1>&2
    exit 1
}

curl --silent --max-time 2 --insecure https://localhost:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://localhost:${APISERVER_DEST_PORT}/"
if ip addr | grep -q ${APISERVER_VIP}; then
    curl --silent --max-time 2 --insecure https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/ -o /dev/null || errorExit "Error GET https://${APISERVER_VIP}:${APISERVER_DEST_PORT}/"
fi
```
```
sudo chmod +x /etc/keepalived/check_apiserver.sh
sudo cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf-org
sudo sh -c '> /etc/keepalived/keepalived.conf'
```
```
sudo nano /etc/keepalived/keepalived.conf

! /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id LVS_DEVEL
}
vrrp_script check_apiserver {
  script "/etc/keepalived/check_apiserver.sh"
  interval 3
  weight -2
  fall 10
  rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 151
    priority 255
    authentication {
        auth_type PASS
        auth_pass P@##D321!
    }
    virtual_ipaddress {
        192.168.2.7/24
    }
    track_script {
        check_apiserver
    }
}
```
> SLAVE master1=>255, master2=>254, master3=>253
```
sudo systemctl enable keepalived --now
```

# HAProxy
### => master-n
```
apt install haproxy
sudo cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg-org
```
```
sudo nano /etc/haproxy/haproxy.cfg

global
    log /dev/log local0 info alert
    log /dev/log local1 notice alert
    daemon

defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 1
    timeout http-request    10s
    timeout queue           20s
    timeout connect         5s
    timeout client          20s
    timeout server          20s
    timeout http-keep-alive 10s
    timeout check           10s

#---------------------------------------------------------------------
# apiserver frontend which proxys to the masters
#---------------------------------------------------------------------
frontend apiserver
    bind *:8443 mss 1500
    mode tcp
    option tcplog
    default_backend apiserver
#---------------------------------------------------------------------
# round robin balancing for apiserver
#---------------------------------------------------------------------
backend apiserver
    option httpchk GET /healthz
    http-check expect status 200
    mode tcp
    option ssl-hello-chk
    balance     roundrobin
        server k8s1-controlplane01 192.168.2.1:6443 check fall 3 rise 2
        server k8s1-controlplane02 192.168.2.2:6443 check fall 3 rise 2
        server k8s1-controlplane03 192.168.2.3:6443 check fall 3 rise 2
```
```
sudo systemctl enable haproxy --now
```

# Container Runtimes
- [https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [https://docs.docker.com/engine/install/debian/](https://docs.docker.com/engine/install/debian/)
- [https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)
### => master-n, node-n
```
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release
sudo mkdir -m 0755 -p /etc/apt/keyrings
```
> debian
```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
> ubuntu
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```
sudo apt-get update
sudo apt install containerd.io
sudo systemctl status containerd
```
```
nano /etc/containerd/config.toml

version = 2
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true

systemctl restart containerd
```
# Kubeadm
- [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
### => master-n, node-n
> kopia w razie awarii dostepności klucza
- [https://web.archive.org/web/20230223152417/https://packages.cloud.google.com/apt/doc/apt-key.gpg](https://web.archive.org/web/20230223152417/https://packages.cloud.google.com/apt/doc/apt-key.gpg)
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /etc/apt/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
# Create cluster
- [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
### => master-n, node-n
```
sudo kubeadm init --control-plane-endpoint=k8s1-controlplane-vip:8443 --upload-certs --v=5
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/calico.yaml
```
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```
# Add controlplane
```
kubeadm join k8s1-controlplane-vip:8443 --token uu1bza.61434ait1gsutgcn \
        --discovery-token-ca-cert-hash sha256:33121a793005123c9a8e887269ed1d1140a9f5b2575fd78bdafd3734363bd045 \
        --control-plane --certificate-key a5d170c15d14fc249605aad81097a94c301f61b5219e428799f16377061efdbd
```
# Add worker
```
kubeadm join k8s1-controlplane-vip:8443 --token uu1bza.61434ait1gsutgcn \
		--discovery-token-ca-cert-hash sha256:33121a793005123c9a8e887269ed1d1140a9f5b2575fd78bdafd3734363bd045
```

# Reset install
```
sudo kubeadm reset
rm /etc/cni/net.d/*
rm -R $HOME/.kube
iptables --list
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
ip route
??? ip route flush proto bird
```
# Delete node
```
kubectl get nodes
kubectl describe node <node name>
kubectl drain node <node name>
kubectl delete node <node name>
```

# Upgrade
- [https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
```
apt update
apt-cache madison kubeadm

kubectl drain <node-to-drain> --ignore-daemonsets

apt-mark unhold kubeadm && \
apt-get install -y kubeadm=1.27.1-00 && \
apt-mark hold kubeadm

kubeadm upgrade plan

kubeadm upgrade node
// lub dla pierwszego mastera
kubeadm upgrade apply

apt-mark unhold kubelet kubectl && \
apt-get install -y kubelet=1.27.1-00 kubectl=1.27.1-00 && \
apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

kubectl uncordon <node-to-uncordon>

```

