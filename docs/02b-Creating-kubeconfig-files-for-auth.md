## Creating Kubeconfig files for Authentication. 

Next we will generate **kubeconfigs** which will be used by the various services that will make up the cluster. In this section, we will generate these kubeconfigs. Once complete there will be a set of kubeconfigs which will be used later to configure the Kubernetes cluster. 

##### Files this will generate
* worker1.kubeconfig
* worker2.kubeconfig
* admin.kubeconfig
* kube-controller-manager.kubeconfig
* kube-scheduler.kubeconfig

## Install Kubectl client tool
In this step we will download **kubectl** client version 1.18.6:
```
KUBE_VER="v1.18.6"
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
| KUBERNETES_ADDRESS=<load balancer private ip>   |
| WORKER1_HOST=<worker 1 hostname>                |
| WORKER2_HOST=<worker 2 hostname>                |
| WORKER1_IP=<worker 1 External IP address>       |
| WORKER2_IP=<worker 1 External IP address>       |

```
KUBERNETES_ADDRESS=172.16.0.30
WORKER1_HOST=worker1 
WORKER2_HOST=worker2
WORKER1_IP=192.168.1.21
WORKER2_IP=192.168.1.22

```
Now, generate a kubelet kubeconfig for each worker node:
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
Then we'll generate the kube-controller-manager kubeconfig file:
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
Then we'll generate the kube-scheduler kubeconfig file:
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
Generate an admin kubeconfig:
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
## Distributing the kubeconfig files to the appropriate servers
Now that we have generated the kubeconfig files that we will need in order to configure our Kubernetes cluster, we need to make sure that each cloud server has a copy of the kubeconfig files that it will need.

Move kubeconfig files to the worker nodes:
```
scp ${WORKER1_HOST}.kubeconfig kube-proxy.kubeconfig kubeadmin@${WORKER1_IP}:~/
scp ${WORKER2_HOST}.kubeconfig kube-proxy.kubeconfig kubeadmin@${WORKER2_IP}:~/
```
Move kubeconfig files to the controller nodes:
```
scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig kubeadmin@${CONTROLLER0_IP}:~/
scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig kubeadmin@${CONTROLLER1_IP}:~/
```

## Generate the Data Encryption Confie file
In order to make use of Kubernetes' ability to encrypt sensitive data at rest, you need to provide Kubernetes with an encrpytion key using a data encryption config file.
What we do next is create an encryption key, storing it in the necessary file, and then copy that file to your Kubernetes controllers
Generate the Kubernetes Data encrpytion config file containing the encrpytion key
```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

cat > encryption-config.yaml << EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```
```
scp encryption-config.yaml kubeadmin@${CONTROLLER0_IP}:~/
scp encryption-config.yaml kubeadmin@${CONTROLLER1_IP}:~/
```
