# Bootstrapping a H/A etcd cluster

In this section you will bootstrap an etcd cluster. The following machines will be used:

`controller-0`      `controller-1`

## Why

All Kubernetes components are stateless which greatly simplifies managing a Kubernetes cluster. All state is stored in etcd, which is a database
and must be treated specially. To limit the number of compute resource to complete this lab etcd is being installed on the Kubernetes controller nodes. 
In production environments etcd should be run on a dedicated set of machines for the following reasons:

* The etcd lifecycle is not tied to Kubernetes. We should be able to upgrade etcd independently of Kubernetes.
* Scaling out etcd is different than scaling out the Kubernetes Control Plane.
* Prevent other applications from taking up resources (CPU, Memory, I/O) required by etcd.

## Provision the etcd Cluster

Run the following commands on `controller-0`, `controller-1`:

### TLS Certificates

The TLS certificates created in the [Setting up a CA and TLS Cert Generation](02-certificate-authority.md) lab will be used to secure communication between the Kubernetes API server and the etcd cluster. The TLS certificates will also be used to limit access to the etcd cluster using TLS client authentication. Only clients with a TLS certificate signed by a trusted CA will be able to access the etcd cluster.

Maket the **etcd** configuration directory and copy the TLS certificates :
```
sudo mkdir -p /etc/etcd/
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

### Download and Install the etcd binaries
As of October, 2020 there are 3 active lineages of etcd that include arm64 as part of thier release schedule.
* v3.4.1 to v3.4.12
* v3.3.0 to v3.3.25
* v3.2.0 to v3.2.31

ARM is not officially supported but is released under the experimental flag, which means there's limited support. 
```
ETCD_VER="v3.4.12"
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-arm64.tar.gz
```

Extract and install the `etcd` server binary and the `etcdctl` command line client: 
```
tar -xvf etcd-${ETCD_VER}-linux-arm64.tar.gz
sudo mv etcd-${ETCD_VER}-linux-arm64/etcd* /usr/local/bin/
rm -rf etcd-${ETCD_VER}-linux-arm*
```
#### Make the etcd data store directory
All etcd data is stored under the etcd data directory. In a production cluster the data directory should be backed by a persistent disk. Create the etcd data directory:
```
sudo mkdir -p /var/lib/etcd
```
###  Create the etcd systemd unit file variables:
The etcd server will be started and managed by systemd.
* Be sure you replace all of the <placeholder values> with their corresponding real values:
* Each etcd member must have a unique name within an etcd cluster. Using the hostname is the easiest way of creating a unique etcd name.
* The internal IP address will be used by etcd to serve client requests and communicate with other etcd peers.
* The Initial_CLUSTER flag needs to contain all the servers and their IPs in a comma seperated list.
```
ETCD_NAME=<cloud server hostname>
INTERNAL_IP=$(echo "$(ip a show eth0 | awk '/inet / {print $2}'| cut -b 1-11 )")
INITIAL_CLUSTER=<controller 1 hostname>=https://<controller 1 private ip>:2380,<controller 2 hostname>=https://<controller 2 private ip>:2380
echo ${INTERNAL_IP}
```
**Example** 
```
ETCD_NAME=$(hostname)
INTERNAL_IP=$(echo "$(ip a show eth0 | awk '/inet / {print $2}'| cut -b 1-11 )")
INITIAL_CLUSTER=controller0=https://172.16.0.20:2380,controller1=https://172.16.0.40:2380
```
Make sure the IP addresses in the **--initial-cluster** match your environment.

### Create the etcd systemd unit file:
**THE BIG WARNING** The ARM architecture is currently not supported, so you need to tell etcd that we want to use an unsupported architecture.
This is done by providing the environment variable **ETCD_UNSUPPORTED_ARCH=arm64**. If this line isn't there etcd is not going to work on your Raspberry.
```
cat << EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Environment=ETCD_UNSUPPORTED_ARCH=arm64
Type=notify
ExecStart=/usr/local/bin/etcd --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster ${INITIAL_CLUSTER} \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
Whenever you have made changes to a service file for a service that's part of systemd you need to reload, and restart the service. To restart the etcd server:

```
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl start etcd
```


### Verification

```
sudo systemctl status etcd --no-pager -l
```


## Verification

List the etcd cluster members:

```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

> output
```
member 68326dea8aa5233d is healthy: got healthy result from https://172.16.0.40:2379
member db49ef42428b90ee is healthy: got healthy result from https://172.16.0.20:2379
cluster is healthy
```

> Remember to run these steps on `controller-0`, `controller-1`

Next: [Bootstrapping the Kubernetes Control Plane](08-bootstrapping-kubernetes-controllers.md)

