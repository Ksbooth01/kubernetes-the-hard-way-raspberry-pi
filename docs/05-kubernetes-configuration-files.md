## Generating Kubernetes Configuration files for Authentication. 


In this section you will generate [Kubernetes configuration files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/), also known as kubeconfigs, which enable Kubernetes clients to locate and authenticate to the Kubernetes API Servers.

## Client Authentication Configs

In this section you will generate kubeconfig files for the `controller manager`, `kubelet`, `kube-proxy`, and `scheduler` clients and the `admin` user.
 
##### Files this will generate
* worker-1.kubeconfig
* worker-2.kubeconfig
* admin.kubeconfig
* kube-controller-manager.kubeconfig
* kube-scheduler.kubeconfig

## Install Kubectl client tool
In this step we will download **kubectl** client version 1.19.2:
```
KUBE_VER="v1.19.2"
wget https://storage.googleapis.com/kubernetes-release/release/${KUBE_VER}/bin/linux/arm64/kubectl

chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```
#### Set up Environment variables for this section
Now, we'll create an environment variable to store the address of the Kubernetes API, and set it to the private IP of your load balancer cloud server:
If you haven't got them set from the last section set the following variables for the worker nodes


| Variable                                        |
|:-----------------------------------------------:|
| KUBERNETES_ADDRESS=<load balancer private ip>   | NOTE: kelsey hightower is using the external address here
| WORKER1_HOST=<worker 1 hostname>                |
| WORKER2_HOST=<worker 2 hostname>                |
| WORKER1_IP=<worker-1 Internal IP address>       |
| WORKER2_IP=<worker-2 Internal IP address>       |

```
KUBERNETES_ADDRESS=172.16.0.30
WORKER1_HOST=worker-1 
WORKER2_HOST=worker-2
WORKER1_IP=172.16.0.21
WORKER2_IP=172.16.0.22

```

### The kubelet Kubernetes Configuration File

When generating kubeconfig files for Kubelets the client certificate matching the Kubelet's node name must be used. This will ensure Kubelets are properly authorized by the Kubernetes [Node Authorizer](https://kubernetes.io/docs/admin/authorization/node/).

> The following commands must be run in the same directory used to generate the SSL certificates during the [Generating TLS Certificates](04-certificate-authority.md) lab.

Generate a kubeconfig file for each worker node:
```
for instance in ${WORKER1_HOST} ${WORKER2_HOST}; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```
Results:

```
worker-1.kubeconfig
worker-2.kubeconfig
```
### The kube-proxy Kubernetes Configuration File

Generate a kubeconfig file for the `kube-proxy` service:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_ADDRESS}:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```

Results:

```
kube-proxy.kubeconfig
```
### The kube-controller-manager Kubernetes Configuration File
Generate a kubeconfig file for the `kube-controller-manager` service:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

Results:

```
kube-controller-manager.kubeconfig
```

### The kube-scheduler Kubernetes Configuration File

Generate a kubeconfig file for the `kube-scheduler` service:

```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```
```
kube-scheduler.kubeconfig
```

### The admin Kubernetes Configuration File

Generate a kubeconfig file for the `admin` user:
```
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

Results:

```
admin.kubeconfig
```


## Distributing the kubeconfig files to the appropriate servers
Now that we have generated the kubeconfig files that we will need in order to configure our Kubernetes cluster, we need to make sure that each cloud server has a copy of the kubeconfig files that it will need.

Copy  the appropriate `kubelet` and `kube-proxy` kubeconfig files to each worker node instance:
```
scp ${WORKER1_HOST}.kubeconfig kube-proxy.kubeconfig kubeadmin@${WORKER1_IP}:~/
scp ${WORKER2_HOST}.kubeconfig kube-proxy.kubeconfig kubeadmin@${WORKER2_IP}:~/
```
Copy the appropriate `kube-controller-manager` and `kube-scheduler` kubeconfig files to each controller instance:
```
CONTROLLER0_IP=172.16.0.20
CONTROLLER1_IP=172.16.0.40
scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig kubeadmin@${CONTROLLER0_IP}:~/
scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig kubeadmin@${CONTROLLER1_IP}:~/
```

Next: [Generating the Data Encryption Config and Key](06-data-encryption-keys.md)


