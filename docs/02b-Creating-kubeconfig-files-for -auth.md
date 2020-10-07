

## Creating Kubeconfig files for Authentication. 

Next we will generate **kubeconfigs** which will be used by the various services that will make up the cluster. In this lesson, we will generate these kubeconfigs. Once compoete there will be a set of kubeconfigs which will be used later to configure the Kubernetes cluster.
 
First, we'll create an environment variable to store the address of the Kubernetes API, and set it to the private IP of your load balancer cloud server:
```
KUBERNETES_ADDRESS=<load balancer private ip>

KUBERNETES_ADDRESS=192.168.1.30
```
If you haven't got them set from the last section set the following variables for the worker nodes
```
NODE1_HOST=<worker 1 hostname> 
NODE2_HOST=<worker 2 hostname>
```
Now, generate a kubelet kubeconfig for each worker node:

for instance in ${NODE1_HOST} ${NODE2_HOST} ; do
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
