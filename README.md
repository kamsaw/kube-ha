# Kubernetes Highly Available
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/
- https://www.linuxtechi.com/setup-highly-available-kubernetes-cluster-kubeadm/

![aaa](https://user-images.githubusercontent.com/38559302/233606228-ee94ec24-b7a9-430d-9a4b-781aae2793cd.jpg)

> ETCD Failure Tolerance

| Cluster Size | Majority | Failure Tolerance |
| ------------ | -------- | ----------------- |
| 1 | 1 | 0 |
| 2 | 2 | 0 |
| - 3 - | - 2 - | - 1 - |
| 4 | 3 | 1 |
| 5 | 3 | 2 |
| 6 | 4 | 2 |
| 7 | 4 | 3 |
| 8 | 5 | 3 |
| 9 | 5 | 4 |

# preinstall
### => master-n, worker-n
```
adduser kube
ssh -> master-n <> master-n
ssh -> master-n > worker-n
```
```
apt install htop mc systemd-timesyncd etcd-client

apt install hyperv-daemons
```
```
nano /etc/hosts
---
192.168.2.1 k8s1-controlplane01
192.168.2.2 k8s1-controlplane02
192.168.2.3 k8s1-controlplane03
192.168.2.4 k8s1-node01
192.168.2.5 k8s1-node02
192.168.2.6 k8s1-node03
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
- https://kubernetes.io/docs/setup/production-environment/container-runtimes/
- https://docs.docker.com/engine/install/debian/
- https://docs.docker.com/engine/install/ubuntu/
### => master-n, worker-n
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
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
### => master-n, worker-n
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
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
- https://kubernetes.io/docs/concepts/cluster-administration/addons/
### => master-1
```
sudo kubeadm init --control-plane-endpoint=k8s1-controlplane-vip:8443 --upload-certs --v=5
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.1/manifests/calico.yaml
```
# Add controlplane
### => master-2,3
```
kubeadm join k8s1-controlplane-vip:8443 --token uu1bza.61434ait1gsutgcn \
        --discovery-token-ca-cert-hash sha256:33121a793005123c9a8e887269ed1d1140a9f5b2575fd78bdafd3734363bd045 \
        --control-plane --certificate-key a5d170c15d14fc249605aad81097a94c301f61b5219e428799f16377061efdbd --v=5
```
# Add worker
### => worker-n
```
kubeadm join k8s1-controlplane-vip:8443 --token uu1bza.61434ait1gsutgcn \
		--discovery-token-ca-cert-hash sha256:33121a793005123c9a8e887269ed1d1140a9f5b2575fd78bdafd3734363bd045 --v=5
```
# Add configs
### => master-n
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### => master-n, worker-n
```
crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```
# helm
```
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

# Kubernetes Dashboard
- https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/
- https://github.com/kubernetes/dashboard
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

// change the .spec.type to NodePort
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard

kubectl create serviceaccount admin-user -n kubernetes-dashboard
kubectl create clusterrolebinding dashboard-admin -n kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:admin-user
kubectl -n kubernetes-dashboard create token admin-user

kubectl proxy
```

# Portainer
- https://docs.portainer.io/start/install/server/kubernetes/baremetal
```
kubectl apply -n portainer -f https://downloads.portainer.io/ee2-18/portainer.yaml
lub

helm repo add portainer https://portainer.github.io/k8s/
helm repo update

helm upgrade --install --create-namespace -n portainer portainer portainer/portainer --set enterpriseEdition.enabled=false --set tls.force=true

=> master-n, worker-n
apt-get install nfs-common

=> k8s1-nfs
sudo apt-get install nfs-kernel-server

sudo mkdir -p /home/k8s1-share001
sudo chown nobody:nogroup /home/k8s1-share001
sudo chmod g+rwxs /home/k8s1-share001

nano /etc/exports
/home/k8s1-share001 192.168.72.81(rw,sync,no_root_squash,no_subtree_check) 192.168.72.82(rw,sync,no_root_squash,no_subtree_check) 192.168.72.83(rw,sync,no_root_squash,no_subtree_check) 192.168.72.84(rw,sync,no_root_squash,no_subtree_check) 192.168.72.85(rw,sync,no_root_squash,no_subtree_check) 192.168.72.86(rw,sync,no_root_squash,no_subtree_check)

sudo exportfs -av 

/sbin/showmount -e localhost
/sbin/showmount -e 192.168.72.88

helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner
helm repo list

helm install nfs-subdir-external-provisioner \
nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
--set nfs.server=192.168.72.88 \
--set nfs.path=/home/k8s1-share001 \
--set storageClass.onDelete=true \
--set storageClass.name=k8s1-share001 \
--set replicaCount=1 \
--set storageClass.defaultClass=true

nano pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: partioner
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: k8s1-share001
  resources:
    requests:
      storage: 2Gi
---
k get pvc

// usuwanie
helm uninstall nfs-subdir-external-provisioner
```

# Add Ingress Controller
- https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/
- https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/
- https://www.dbi-services.com/blog/setup-an-nginx-ingress-controller-on-kubernetes/
![nginx](https://user-images.githubusercontent.com/38559302/234502108-4c8029aa-d289-472a-9cf7-3e7611455af8.png)
![0_7970PSI1KE6Qskin](https://user-images.githubusercontent.com/38559302/234670940-a81c66f7-dc27-453a-8f4a-759881bf2f81.png)


```
kubectl patch svc service/nginx-ingress-nginx-ingress-controller -n nginx-ingress -p '{"spec": {"type": "LoadBalancer", "externalIPs":["192.168.72.87"]}}'
```

# Generate new token
> for add controlplane
```
// generate --certificate-key string
kubeadm init phase upload-certs --upload-certs
kubeadm token create --print-join-command

kubeadm join k8s1-controlplane-vip:8443 --token ??? \
        --discovery-token-ca-cert-hash sha256:??? \
        --control-plane --certificate-key ??? --v=5
```
> for add worker
```
kubeadm token create --print-join-command
```
> discovery-token-ca-cert-hash
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
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
systemctl daemon-reload
systemctl restart containerd
--
kubeadm reset -f
sudo apt-get purge kubeadm kubectl kubelet kubernetes-cni kube* containerd.io
sudo apt-get autoremove  
rm -rf /etc/cni /etc/kubernetes /var/lib/dockershim /var/lib/etcd /var/lib/kubelet /var/run/kubernetes ~/.kube/* /var/calio /var/lib/containerd
iptables -F && iptables -X
iptables -t nat -F && iptables -t nat -X
iptables -t raw -F && iptables -t raw -X
iptables -t mangle -F && iptables -t mangle -X
systemctl daemon-reload
```
# Delete node
```
kubectl get nodes
kubectl describe node <node name>
kubectl drain node <node name>
kubectl delete node <node name>
```

# Upgrade
- https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
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
# ETCD help
```
ETCDCTL_API=3 etcdctl --cert=/etc/kubernetes/pki/etcd/server.crt  --key=/etc/kubernetes/pki/etcd/server.key \
			--cacert=/etc/kubernetes/pki/etcd/ca.crt   --endpoints=https://192.168.1.1:2379 \
				endpoint status --write-out=table

ETCDCTL_API=3 etcdctl --cert=/etc/kubernetes/pki/etcd/server.crt  --key=/etc/kubernetes/pki/etcd/server.key \
			--cacert=/etc/kubernetes/pki/etcd/ca.crt --endpoints=https://192.168.1.1:2379 \
			member list --write-out=table		
			
ETCDCTL_API=3 etcdctl --cert=/etc/kubernetes/pki/etcd/server.crt  --key=/etc/kubernetes/pki/etcd/server.key \
			--cacert=/etc/kubernetes/pki/etcd/ca.crt    endpoint health --cluster
			
ETCDCTL_API=3 etcdctl --cert=/etc/kubernetes/pki/etcd/server.crt  --key=/etc/kubernetes/pki/etcd/server.key \
			--cacert=/etc/kubernetes/pki/etcd/ca.crt  member remove ???	
```


