# Kubernetes Highly Available

[https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/high-availability/)

[https://www.linuxtechi.com/setup-highly-available-kubernetes-cluster-kubeadm/](https://www.linuxtechi.com/setup-highly-available-kubernetes-cluster-kubeadm/)


![aaa](https://user-images.githubusercontent.com/38559302/233606228-ee94ec24-b7a9-430d-9a4b-781aae2793cd.jpg)

## Install
```
/etc/containerd/config.toml
```
```
version = 2
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
```
```
service containerd restart
```
