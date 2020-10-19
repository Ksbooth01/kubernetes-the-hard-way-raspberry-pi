# Bootstrapping Kubernetes Workers

In this lab you will bootstrap the Kubernetes worker nodes. The following virtual machines will be used: `worker1`, and `worker2`

## Why

Kubernetes worker nodes are responsible for running your containers. All Kubernetes clusters need one or more worker nodes. We are running the worker nodes on dedicated machines for the following reasons:

* Ease of deployment and configuration
* Avoid mixing arbitrary workloads with critical cluster components. We are building machines with just enough resources so we don't have to worry about wasting resources.

Some people would like to run workers and cluster services anywhere in the cluster. This is totally possible, and you'll have to decide what's best for your environment.

## Install the OS dependencies:
```
{
  sudo apt-get update
  sudo apt-get -y install socat conntrack ipset
}
```
#### Brief Explanation
* **socat**      - utility that is a relay for bidirectional data transfers between two independent data channels.
* **conntrack**  - provides an interface to the connnection tracking system, which you can show, delete and update the existing state entries; and listens to flow events.
* **ipset**      - allows you to organize a list of networks, IP or MAC addresses, etc. which is very convenient to use for example with IPTables.
## Provision the Kubernetes Worker Nodes

Run the following commands on `worker1`, and `worker2`:

### Disable Swap
* By default the kubelet will fail to start if swap is enabled. It is recommended that swap be disabled to ensure Kubernetes can provide proper resource allocation and quality of service.
* By default ubuntu 20.04 image does not have the swap file enabled.  To validate this is the case type:
```
sudo swapon --show
```
If the swap file is disabled there should be no results returned.

#### Docker

Install docker on the Raspberry Pi is so easy, a caveman could do it:

```
  curl -sSL http://get.docker.com  | sh
  sudo usermod -aG docker pi
```
### Download and install the Kubernetes worker binaries:
Set the version and architectures for the downloads.
```
K8S_VER=v1.18.6
CRICTL_VER=v1.18.0
CNI_VER=v0.8.7
K8S_ARCH=arm64
```

```
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/${CRICTL_VER}/crictl-${CRICTL_VER}-linux-${K8S_ARCH}.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc91/runc.${K8S_ARCH} \
  https://github.com/containernetworking/plugins/releases/download/${CNI_VER}/cni-plugins-linux-${K8S_ARCH}-${CNI_VER}.tgz \
  https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/${K8S_ARCH}/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/${K8S_ARCH}/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/${K8S_ARCH}/kubelet
```

Create the installation directories:

```
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```
Install the worker binaries:

{
  mkdir containerd
  tar -xvf crictl-${CRICTL_VER}-linux-${K8S_ARCH}.tar.gz
  tar -xvf containerd-1.3.6-linux-${K8S_ARCH}.tar.gz -C containerd
  sudo tar -xvf cni-plugins-linux-${K8S_ARCH}-v0.8.6.tgz -C /opt/cni/bin/
  sudo mv runc.${K8S_ARCH} runc
  chmod +x crictl kubectl kube-proxy kubelet runc 
  sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
  sudo mv containerd/bin/* /bin/
}


#### Move the TLS certificates in place
Previously, we copied the TLS certificates to the `$HOME` directory of the worker nodes. We will now copy them a kubernetes working directory. 
```
sudo mkdir -p /var/lib/kubernetes
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /var/lib/kubernetes/
```

```
sudo sh -c 'echo "apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /var/lib/kubernetes/ca.pem
    server: https://172.16.1.94:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubelet
  name: kubelet
current-context: kubelet
users:
- name: kubelet
  user:
    token: chAng3m3" > /var/lib/kubelet/kubeconfig'
```

Create the kubelet systemd unit file:

```
sudo sh -c 'echo "[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/bin/kubelet \
  --allow-privileged=true \
  --api-servers=https://10.0.1.94:6443,https://10.0.1.95:6443,https://10.0.1.96:6443 \
  --cloud-provider= \
  --cluster-dns=10.32.0.10 \
  --cluster-domain=cluster.local \
  --configure-cbr0=true \
  --container-runtime=docker \
  --docker=unix:///var/run/docker.sock \
  --network-plugin=kubenet \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --reconcile-cidr=true \
  --serialize-image-pulls=false \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --v=2
  
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target" > /etc/systemd/system/kubelet.service'
```

```
sudo systemctl daemon-reload
sudo systemctl enable kubelet
sudo systemctl start kubelet
```

```
sudo systemctl status kubelet --no-pager
```


#### kube-proxy


```
sudo sh -c 'echo "[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-proxy \
  --master=https://10.0.1.94:6443 \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --proxy-mode=iptables \
  --v=2
  
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target" > /etc/systemd/system/kube-proxy.service'
```

```
sudo systemctl daemon-reload
sudo systemctl enable kube-proxy
sudo systemctl start kube-proxy
```

```
sudo systemctl status kube-proxy --no-pager
```

> Remember to run these steps on `worker0`, and `worker1`
