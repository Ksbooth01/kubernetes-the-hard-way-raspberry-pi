# Configuring the Kubernetes Client - Remote Access

**Note:** The original Kubernetes-the-hard-way (Kelseykightower) installs this on a remote node. I find it helpful to install this on `worker1` and `worker2`.  That way you can run kubectl commands on them as well.
On the location that you created you CA certs:
```
cd &HOME/kubernetes

for instance in worker-1 worker-2; do
  scp admin.pem admin-key.pem kubeadmin@${instance}:~/
done
```
on each work node: `worker-1  and  worker-2` 
```
sudo cp admin* /var/lib/kubernetes
```

# Configuring kubectl for Remote Access

In this lab you will generate a kubeconfig file for the `kubectl` command line utility based on the `admin` user credentials.

> Run the commands in this lab from the same directory used to generate the admin client certificates.

## The Admin Kubernetes Configuration File

Each kubeconfig requires a Kubernetes API Server to connect to. To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

Generate a kubeconfig file suitable for authenticating as the `admin` user:

```
{
  KUBERNETES_LB_ADDRESS=172.16.0.30

  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=/var/lib/kubernetes/ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_LB_ADDRESS}:6443

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem

  kubectl config set-context kubernetes-the-hard-way \
    --cluster=kubernetes-the-hard-way \
    --user=admin

  kubectl config use-context kubernetes-the-hard-way
}

```
Reference doc for kubectl config [here](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/)

## Verification

Check the health of the remote Kubernetes cluster:

```
kubectl get componentstatuses
```

> output

```
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-1               Healthy   {"health":"true"}
etcd-0               Healthy   {"health":"true"}
```

List the nodes in the remote Kubernetes cluster:

```
kubectl get nodes
```

> output

```
NAME       STATUS   ROLES    AGE    VERSION
worker-1   NotReady    <none>   118s   v1.19.2
worker-2   NotReady    <none>   118s   v1.19.2
```

Note: It is OK for the worker node to be in a `NotReady` state. Worker nodes will come into `Ready` state once networking is configured.

Next: [Deploy Pod Networking](11-configure-pod-networking.md)

