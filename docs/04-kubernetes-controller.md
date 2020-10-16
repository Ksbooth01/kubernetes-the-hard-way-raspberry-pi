# Bootstrapping an H/A Kubernetes Control Plane

## Provision the Kubernetes Controller Cluster
Run the following commands on `controller0`, `controller1`:

The Kubernetes components that make up the control plane include the following components:

* **Kubernetes API Server** - This server the Kubernetes API and allows users to interact with the cluster
* **Kubernetes Scheduler** - Schedules pose on available worker nodes.
* **Kubernetes Controller Manager** Manages a series of controllers the provide a wide range os functionality to the cluster
* **etcd** (already installed) a distributed datastore used by the cluster.  integeral, independent

Each component is being run on the same machines for the following reasons:
* The Scheduler and Controller Manager are tightly coupled with the API Server
* Only one Scheduler and Controller Manager can be active at a given time, but it's ok to run multiple at the same time. Each component will elect a leader via the API Server.
* Running multiple copies of each component is required for H/A
* Running each component next to the API Server eases configuration.

### Download and install the Kubernetes controller binaries

create a tempory directory for the binaries and set the environment variables for you to download : 
```
mkdir -p $HOME/kubebits
cd ~/kubebits
K8S_VER=v1.19.2
K8S_ARCH=arm64
```
Download the official Kubernetes release binaries:
```
wget https://storage.googleapis.com/kubernetes-release/release/$K8S_VER/bin/linux/$K8S_ARCH/kube-apiserver
wget https://storage.googleapis.com/kubernetes-release/release/$K8S_VER/bin/linux/$K8S_ARCH/kube-controller-manager
wget https://storage.googleapis.com/kubernetes-release/release/$K8S_VER/bin/linux/$K8S_ARCH/kube-scheduler
wget https://storage.googleapis.com/kubernetes-release/release/$K8S_VER/bin/linux/$K8S_ARCH/kubectl
```

Set the binary permissions to allow execute and Install the Kubernetes binaries in the working directory:
```
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
```
```
sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
```

### Setting up the Kubernetes API Server
The Kubernetes API server provides the primary interface for the Kubernetes control plane and the cluster as a whole. When you interact with Kubernetes, you are nearly always doing it through the Kubernetes API server.

```
cd ~
sudo mkdir -p /var/lib/kubernetes/

sudo cp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
  service-account-key.pem service-account.pem \
  encryption-config.yaml /var/lib/kubernetes/
 ```
Set environment variables needed to create the systemd unit file. **Note:** Make sure you replace the placeholders with their actual values:
``` 
INTERNAL_IP=$(echo "$(ip a show eth0 | awk '/inet / {print $2}'| cut -b 1-11 )")
 CONTROLLER0_IP=<private ip of controller 0>
 CONTROLLER1_IP=<private ip of controller 1>
 ```
**example**
 ``` 
 INTERNAL_IP=$(echo "$(ip a show eth0 | awk '/inet / {print $2}'| cut -b 1-11 )")
 CONTROLLER0_IP=172.16.0.20 
 CONTROLLER1_IP=172.16.0.40 
```
  
##### Create the kube-apiserver unit file
```
cat << EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
    --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://$CONTROLLER0_IP:2379,https://$CONTROLLER1_IP:2379 \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2 \\
  --kubelet-preferred-address-types=InternalIP,InternalDNS,Hostname,ExternalIP,ExternalDNS
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

**Notes:** 
* --kubelet-preferred-address allows you to set prioritization of NIC resolution for your cluster.  So, if you have you  Raspberries dual homes to physical and wireless this will allow you to control which NIC get priority for kubernetes.
* -service-cluster-ip-range=*10.32.0.0/24* 
## Setting up the kube-controller-manager-service
Next, we go through the process of configuring a systemd service for the Kubernetes Controller Manager. Once complete the kubeconfig and systemd unit file set up and ready to run the kube-controller-manager service on both of your control nodes.

##### Create the kube-controller-manager systemd unit file:
```
cat << EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

**explanation** 
--cluster-cidr=10.200.0.0/16 \\
--service-cluster-ip-range=10.32.0.0/24 \\ - This is the default subnet for the kubernetes api service

```
sudo cp kube-controller-manager.kubeconfig /var/lib/kubernetes/
```


## Setting up the Kubernetes Scheduler

```
sudo cp kube-scheduler.kubeconfig /var/lib/kubernetes/
```
create the kube-scheduler yaml config file.
```
cat << EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: componentconfig/v1alpha1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```
Create the kube-scheduler systemd unit file:
```
cat << EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
Start and enable all of the services:
```
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```
Make sure all the services are active (running):
```
sudo systemctl status kube-apiserver kube-controller-manager kube-scheduler --no-page -l
```
Use kubectl to check componentstatuses:
```
kubectl get componentstatuses --kubeconfig admin.kubeconfig
```
You should get output that looks like this:
```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

Attribute-Based Access Control (ABAC) will be used to authorize access to the Kubernetes API. In this lab ABAC will be setup using the Kubernetes policy file backend as documented in the [Kubernetes authorization guide](http://kubernetes.io/docs/admin/authorization).

Download the example authorization policy file:

```
wget https://raw.githubusercontent.com/robertojrojas/kubernetes-the-hard-way-raspberry-pi/master/authorization-policy.jsonl
```

Review the example authorization policy file. No changes are required.

```
cat authorization-policy.jsonl
```

Move the authorization policy file into the Kubernetes configuration directory so it can be read by the Kubernetes API server.

```
sudo mv authorization-policy.jsonl /var/lib/kubernetes/
```

### Create the systemd unit file 

Capture the internal IP address:

```
INTERNAL_IP=$(echo "$(ifconfig eth0 | awk '/\<inet addr\>/ { print substr( $2, 6)}')")
```

---

Create the systemd unit file:

```
cat > kube-apiserver.service <<"EOF"
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota \
  --advertise-address=INTERNAL_IP \
  --allow-privileged=true \
  --apiserver-count=3 \
  --authorization-mode=ABAC \
  --authorization-policy-file=/var/lib/kubernetes/authorization-policy.jsonl \
  --bind-address=0.0.0.0 \
  --enable-swagger-ui=true \
  --etcd-cafile=/var/lib/kubernetes/ca.pem \
  --insecure-bind-address=0.0.0.0 \
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \
  --etcd-servers=https://10.0.1.94:2379,https://10.0.1.95:2379,https://10.0.1.96:2379 \
  --service-account-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --token-auth-file=/var/lib/kubernetes/token.csv \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
the 10.32.0.0 network is a service network that kubernetes API server uses to internally communicate with the other Kubernetes services. the Kubernetes API usually assigns itself the IP address of 10.32.01.1

```
sed -i s/INTERNAL_IP/$INTERNAL_IP/g kube-apiserver.service
```

```
sudo mv kube-apiserver.service /etc/systemd/system/
```


```
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver
sudo systemctl start kube-apiserver
```

```
sudo systemctl status kube-apiserver --no-pager
```

### Kubernetes Controller Manager

```
cat > kube-controller-manager.service <<"EOF"
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-controller-manager \
  --allocate-node-cidrs=true \
  --cluster-cidr=10.200.0.0/16 \
  --cluster-name=kubernetes \
  --leader-elect=true \
  --master=http://INTERNAL_IP:8080 \
  --root-ca-file=/var/lib/kubernetes/ca.pem \
  --service-account-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
sed -i s/INTERNAL_IP/$INTERNAL_IP/g kube-controller-manager.service
```

```
sudo mv kube-controller-manager.service /etc/systemd/system/
```


```
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

```
sudo systemctl status kube-apiserver --no-pager -l 
sudo systemctl status kube-controller-manager --no-pager -l 
sudo systemctl status kube-scheduler --no-pager -l 
```

### Kubernetes Scheduler

```
cat > kube-scheduler.service <<"EOF"
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/bin/kube-scheduler \
  --leader-elect=true \
  --master=http://INTERNAL_IP:8080 \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```
sed -i s/INTERNAL_IP/$INTERNAL_IP/g kube-scheduler.service
```

```
sudo mv kube-scheduler.service /etc/systemd/system/
```

```
sudo systemctl daemon-reload
sudo systemctl enable kube-scheduler
sudo systemctl start kube-scheduler
```

```
sudo systemctl status kube-scheduler --no-pager
sudo systemctl status kube-scheduler --no-pager
sudo systemctl status kube-scheduler --no-pager

```


### Verification

```
kubectl get componentstatuses
```
```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-1               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}  
```

> Remember to run these steps on `controller0`, `controller1`, and `controller2`
