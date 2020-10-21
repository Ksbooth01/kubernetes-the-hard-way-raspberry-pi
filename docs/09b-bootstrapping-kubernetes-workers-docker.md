# Bootstrapping Kubernetes Workers (Docker Edition)

The following virtual machines will be usedin theis section: `worker1`, and `worker2`

### Why

Kubernetes worker nodes are responsible for running your containers. All Kubernetes clusters need one or more worker nodes. We are running the worker nodes on dedicated machines for the following reasons:

* Ease of deployment and configuration
* Avoid mixing arbitrary workloads with critical cluster components. We are building machines with just enough resources so we don't have to worry about wasting resources.

Some people would like to run workers and cluster services anywhere in the cluster. This is totally possible, and you'll have to decide what's best for your environment.

### Install the OS dependencies:
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


### Confirm Swap is Disabled
* By default the kubelet will fail to start if swap is enabled. It is recommended that swap be disabled to ensure Kubernetes can provide proper resource allocation and quality of service.
* By default ubuntu 20.04 image does not have the swap file enabled.  To validate this is the case type:
```
sudo swapon --show
```
If the swap file is disabled there should be no results returned.

## Install containerd 

```
sudo apt-get install containerd
```
Configure containerd
Create the containerd configuration file:
```
sudo mkdir -p /etc/containerd/
```

```
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/sbin/runc"
      runtime_root = ""
EOF
```
Create the containerd.service systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/usr/sbin/modprobe overlay
ExecStart=/usr/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF

```
## Download and install the Kubernetes worker binaries:

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
  https://github.com/containernetworking/plugins/releases/download/${CNI_VER}/cni-plugins-linux-${K8S_ARCH}-${CNI_VER}.tgz \
  https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/${K8S_ARCH}/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/${K8S_ARCH}/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/${K8S_ARCH}/kubelet
```
**What you installing**
* **CRIctl**  - Container Runtime Interface (CRI) CLI. crictl provides a CLI for CRI-compatible container runtimes. This allows the CRI runtime developers to debug their runtime without needing to set up Kubernetes components
Create the installation directories:
* **CNI** -  Container Networking Interface
* **kubectl** - allows you to run commands against Kubernetes clusters.
* **kube-proxy** -  A network proxy that runs on each node in your cluster. It maintains network rules on the nodes which allow network communication to your Pods from network sessions inside or outside of your cluster.
* **kubelet**  - The primary "node agent" that runs on each node and works in terms of a PodSpec. A PodSpec is a YAML or JSON object that describes a pod. The kubelet takes a set of PodSpecs that are provided and ensures that the containers described in those PodSpecs are running and healthy. 

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
```
{
  tar -xvf crictl-${CRICTL_VER}-linux-${K8S_ARCH}.tar.gz
  sudo tar -xvf cni-plugins-linux-${K8S_ARCH}-${CNI_VER}.tgz -C /opt/cni/bin/
  chmod +x crictl kubectl kube-proxy kubelet  
  sudo mv crictl kubectl kube-proxy kubelet /usr/local/bin/
}

```
## Configure CNI Networking
cluster-cidr=10.200.0.0/16 was set in the kube-controller-manager. each node will get a POD_CIDR which is a portion of this. 
POD_CIDR="10.200.1.0/24" set as this on `worker1` 
POD_CIDR="10.200.2.0/24" set as this on `worker2` 

Create the `bridge` network configuration file:

```
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF

```
Create the loopback network configuration file:

```
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
EOF

```
## Configuring Kubelet
Kubelet is the Kubernetes agent which runs on each worker node. Acting as a middleman between the Kubernetes control plane and the underlying container runtime, it coordinates the running of containers on the worker node

Set a HOSTNAME environment variable that will be used to generate your config files
```
HOSTNAME=$(hostname)
sudo mkdir -p /var/lib/kubernetes
sudo cp ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
sudo cp ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
sudo cp ca.pem /var/lib/kubernetes/
# sudo cp ca.pem kubernetes-key.pem kubernetes.pem /var/lib/kubernetes/
```
Create the `kubelet-config` file:

```
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF

```
set this to the known default of 10.200.0.0./16 podCIDR: "${POD_CIDR}"

Create the kubelet systemd unit file:

```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/docker.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Configure the Kubernetes Proxy
Put hte kube proxy kubecongiruation file in the appropriate directory
```
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```
Create the kube-proxy-config.yaml configuration file:
```
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```
Create the `kube-proxy.service` systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
```
sudo systemctl daemon-reload
sudo systemctl enable containerd kubelet kube-proxy
sudo systemctl start containerd kubelet kube-proxy
```

```
sudo systemctl status containerd --no-pager -l
sudo systemctl status kubelet --no-pager -l
sudo systemctl status kube-proxy --no-pager -l
```
> Remember to run these steps on `worker1`, and `worker2`


## Verification

> The compute instances created in this tutorial will not have permission to complete this section. Run the following commands from the same machine used to create the compute instances `controller0`.

List the registered Kubernetes nodes:

``` 
kubectl get nodes --kubeconfig admin.kubeconfig"
```

> output

```
NAME       STATUS   ROLES    AGE   VERSION
worker1    Ready    <none>   24s   v1.18.6
worker2    Ready    <none>   24s   v1.18.6
```
Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
