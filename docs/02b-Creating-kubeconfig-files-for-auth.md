

## Creating Kubeconfig files for Authentication. 

Next we will generate **kubeconfigs** which will be used by the various services that will make up the cluster. In this lesson, we will generate these kubeconfigs. Once compoete there will be a set of kubeconfigs which will be used later to configure the Kubernetes cluster.
 

## Install Kubectl client tool
In this step we will download **kubectl** client version 0.19.2:
```
wget https://storage.googleapis.com/kubernetes-release/release/v1.19.2/bin/linux/arm64/kubectl

chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```


(OPTIONAL) if you feeling adventuresome you can try the latest version:
```
(OPTONAL) curl -LO "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/arm/kubectl
```
Now, we'll create an environment variable to store the address of the Kubernetes API, and set it to the private IP of your load balancer cloud server:
```
KUBERNETES_ADDRESS=<load balancer private ip>

KUBERNETES_ADDRESS=192.168.1.30
```
If you haven't got them set from the last section set the following variables for the worker nodes
```
NODE1_HOST=<worker 1 hostname> 
NODE2_HOST=<worker 2 hostname>

NODE1_HOST=node1 
NODE2_HOST=node2
```
Now, generate a kubelet kubeconfig for each worker node:
```
for instance in ${NODE1_HOST} ${NODE2_HOST}; do
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
scp ${NODE1_HOST}.kubeconfig kube-proxy.kubeconfig kubeadmin@${NODE1_IP}:~/
scp ${NODE2_HOST}.kubeconfig kube-proxy.kubeconfig kubeadmin@${NODE2_IP}:~/
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
scp encryption-config.yaml kubeadmin@${CONTROLLER0_IP}:~/
scp encryption-config.yaml kubeadmin@${CONTROLLER1_IP}:~/
```
