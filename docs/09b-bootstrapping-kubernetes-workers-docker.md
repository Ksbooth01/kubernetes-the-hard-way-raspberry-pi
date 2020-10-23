# Bootstrapping Kubernetes Workers (Docker Edition)

The following virtual machines will be usedin theis section: `worker1`, and `worker2`

### Why

Kubernetes worker nodes are responsible for running your containers. All Kubernetes clusters need one or more worker nodes. We are running the worker nodes on dedicated machines for the following reasons:

* Ease of deployment and configuration
* Avoid mixing arbitrary workloads with critical cluster components. We are building machines with just enough resources so we don't have to worry about wasting resources.

Some people would like to run workers and cluster services anywhere in the cluster. This is totally possible, and you'll have to decide what's best for your environment.

### Pre-flight check:
the following files shoud be in the home directory of your worker nodes prior to starting this section:
for instance in  worker1 worker2 ; do
|                   **worker1**                 |   |                    **worker2**     |                   |
|:----------------------------------------------:|:---------------------------------------------------:|
|  admin.pem           |  worker1.pem            |  admin.pem           |  worker2.pem            |
|  admin-key.pem       |  worker1-key.pem        |  admin-key.pem       |  worker2-key.pem        |
|  worker1.kubeconfig  |  kube-proxy.kubeconfig  |  worker2.kubeconfig  |  kube-proxy.kubeconfig  |


### Confirm Swap is Disabled
* By default the kubelet will fail to start if swap is enabled. It is recommended that swap be disabled to ensure Kubernetes can provide proper resource allocation and quality of service.
* By default ubuntu 20.04 image does not have the swap file enabled.  To validate this is the case type:
```
sudo swapon --show
```
If the swap file is disabled there should be no results returned.

## Install Docker
This fetches and installs the latest stable version of Docker installs it on the system:
```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

<output truncated>
```
To use Docker as a non-root user, you need to add your user to the “docker” group:
```
  sudo usermod -aG docker kubeadmin
```


## Download and install the Kubernetes worker binaries:

Set the version and architectures for the downloads.
```
K8S_VER=v1.18.6
K8S_ARCH=arm64
```
Download the kubernetes files
```
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/${K8S_ARCH}/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/${K8S_ARCH}/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/${K8S_VER}/bin/linux/${K8S_ARCH}/kubelet
```
**What you installing**

* **kubectl** - allows you to run commands against Kubernetes clusters.
* **kube-proxy** -  A network proxy that runs on each node in your cluster. It maintains network rules on the nodes which allow network communication to your Pods from network sessions inside or outside of your cluster.
* **kubelet**  - The primary "node agent" that runs on each node and works in terms of a PodSpec. A PodSpec is a YAML or JSON object that describes a pod. The kubelet takes a set of PodSpecs that are provided and ensures that the containers described in those PodSpecs are running and healthy. 

#### Make the Kubernetes working directories
```
sudo mkdir -p \

  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```
Install the worker binaries:
```
{
  chmod +x kubectl kube-proxy kubelet  
  sudo mv kubectl kube-proxy kubelet /usr/local/bin/
}
```
## Configuring Kubelet
Kubelet is the Kubernetes agent which runs on each worker node. Acting as a middleman between the Kubernetes control plane and the underlying container runtime, it coordinates the running of containers on the worker node

Set a HOSTNAME environment variable that will be used to generate your config files
```
HOSTNAME=$(hostname)
INTERNAL_IP=<INSERT HERE>
```
Copy the certs and the kube config file to thier working directories:
```
mkdir -p /var/lib/kubelet/
sudo cp ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
sudo cp ca.pem /var/lib/kubernetes/
sudo cp ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
```
confirm: `ls -la /var/lib/kubelet/`

Create the **`kubelet-config`** file:
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
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"

EOF

```
confirm: `cat /var/lib/kubelet/kubelet-config.yaml`
Create the **`kubelet.service`** systemd unit file:

```
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/kubelet-config.yaml \
  --image-pull-progress-deadline=2m \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --container-runtime=docker \
  --network-plugin=cni \
  --cni-bin-dir=/opt/cni/bin \
  --cni-conf-dir=/etc/cni/net.d \
  --register-node=true \
  --node-ip=${INTERNAL_IP} \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target```

 *removed from after -- kubeconfig*
 --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd.sock \\
*if it were for docker **should be** *
  --container-runtime=docker \\
  --container-runtime-endpoint=unix:///var/run/dockershim.sock \\
 
## Configure the Kubernetes Proxy
Put the kube proxy kube configuration file in the appropriate directory
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
sudo systemctl enable kubelet kube-proxy
sudo systemctl start kubelet kube-proxy
```

```

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

* **CRIctl**  - Container Runtime Interface (CRI) CLI. crictl provides a CLI for CRI-compatible container runtimes. This allows the CRI runtime developers to debug their runtime without needing to set up Kubernetes components
Create the installation directories:
* **CNI** -  Container Networking Interface  
  /etc/cni/net.d \
  /opt/cni/bin \
  
  
Next: [Configuring kubectl for Remote Access](10-configuring-kubectl.md)
