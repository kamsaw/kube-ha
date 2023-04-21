# Kubernetes Highly Available
- [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)
- [https://www.linuxtechi.com/setup-highly-available-kubernetes-cluster-kubeadm/](https://www.linuxtechi.com/setup-highly-available-kubernetes-cluster-kubeadm/)

![aaa](https://user-images.githubusercontent.com/38559302/233606228-ee94ec24-b7a9-430d-9a4b-781aae2793cd.jpg)

# Container Runtimes
- [https://kubernetes.io/docs/setup/production-environment/container-runtimes/](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)
- [https://docs.docker.com/engine/install/debian/](https://docs.docker.com/engine/install/debian/)
- [https://docs.docker.com/engine/install/ubuntu/](https://docs.docker.com/engine/install/ubuntu/)
### => master-n, node-n
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
```
swapoff -a
nano /etc/fstab // komentujemy swap_1
reboot
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
> w przypadku błedów można zrestetować instalacje **sudo kubeadm reset**
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
# Add controlplane
```
kubeadm join k8s1-controlplane-vip:8443 --token spkxtq.rhek8wzs7pin0jk5 \
        --discovery-token-ca-cert-hash sha256:33521a793005523c9a8e887269ed1d1640a9f5b2575fd78bdafd3734363bd045 \
        --control-plane --certificate-key a5d770ca5d14fc249605aad8e097a94c301f61b5219e428799f6637706befdbd
```
# Add worker
```
kubeadm join k8s1-controlplane-vip:8443 --token uubbza.6n434aitbgsutgcn \
		--discovery-token-ca-cert-hash sha256:33521a793005523c9a8e887269ed1d1640a9f5b2575fd78bdafd3734363bd045
```

