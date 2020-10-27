
## 09-control-plane
* changed the kubenetes versions from 1.18.6 to 1.19.2

cat << EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/**v1alpha1**
changed to apiVersion: kubescheduler.config.k8s.io/**v1beta1**

**ClusterRole** changed in 1.18.6 from **v1beta1** to **v1** in 1.19.2
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
