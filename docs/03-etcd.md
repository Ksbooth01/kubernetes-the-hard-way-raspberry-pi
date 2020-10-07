# Bootstrapping a H/A etcd cluster

In this lab you will bootstrap an etcd cluster. The following machines will be used:

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

Copy the TLS certificates to the etcd configuration directory:

```
sudo mkdir -p /etc/etcd/
```

```
cd $HOME/kubernetes
sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

### Download and Install the etcd binaries

At the time of this writing etcd for ARM was not supported and downloads were not available, so it was built from the source files on a Raspberry Pi.
To save you the time and trouble, I'm providing the version I built as part of the repository hosting this tutorial.

```
wget https://raw.githubusercontent.com/ksbooth01/kubernetes-the-hard-way-raspberry-pi/master/etcd/etcd-3.1.5-arm.tar.gz
```

Extract and install the `etcd` server binary and the `etcdctl` command line client: 

```
tar -xvf etcd-3.1.5-arm.tar.gz
```

```
sudo mv etcd-3.1.5-arm/etcd* /usr/local/bin/
rm -rf etcd-3.1.5-arm*
```

All etcd data is stored under the etcd data directory. In a production cluster the data directory should be backed by a persistent disk. Create the etcd data directory:

```
sudo mkdir -p /var/lib/etcd
```
### The etcd server will be started and managed by systemd. Create the etcd systemd unit file:

First, let's Set up the following environment variables. Be sure you replace all of the <placeholder values> with their corresponding real values:
```
ETCD_NAME=<cloud server hostname>
INTERNAL_IP=$(echo "$(ifconfig wlan0 | awk '/inet / {print $2}')")
INITIAL_CLUSTER=<controller 1 hostname>=https://<controller 1 private ip>:2380,<controller 2 hostname>=https://<controller 2 private ip>:2380
```
**Example** 
```
ETCD_NAME=$(hostname)

INTERNAL_IP=$(echo "$(ifconfig eth0 | awk '/\<inet addr\>/ { print substr( $2, 6)}')")
 - OR - 
INTERNAL_IP=$(echo "$(ifconfig wlan0 | awk '/inet / {print $2}')")

INITIAL_CLUSTER=controller-0=https://192.168.1.20:2380,controller-1=https://192.168.1.40:2380
```
Make sure the IP addresses in the **--initial-cluster** match your environment.

### Set The Internal IP Address

The internal IP address will be used by etcd to serve client requests and communicate with other etcd peers.
Each etcd member must have a unique name within an etcd cluster. Set the etcd name:

```


```
Notice that I'm using **eth0** or LAN connection to find the IP Address to replace in the Unit file. If you are using a Wi-Fi connection, you will probably need to change this to **wlan0** and use the **SECOND** INTERNAL_IP option
The following command will display the interfaces with their IP Addresses:
The ARM architecture is currently not supported, so we need to tell etcd that we want to use an unsupported architecture.
This is done by providing the environment variable **ETCD_UNSUPPORTED_ARCH=arm**
```
cat << EOF | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Environment=ETCD_UNSUPPORTED_ARCH=arm
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

> Remember to run these steps on `control0`, `control1`

## Verification

Once both etcd nodes have been bootstrapped verify the etcd cluster is healthy:

* On one of the controller nodes run the following command:

```
etcdctl --ca-file=/etc/etcd/ca.pem cluster-health
```
*warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated*  Need to see if there's a way to clean this up.

At first all cluster members were reporting unhealthy for some reason. It took a while (10+ minutes) for them to become healthy.
In any event, I was able to continue with the installation of Kubernetes just fine.

```
member 68326dea8aa5233d is healthy: got healthy result from https://192.168.1.40:2379
member db49ef42428b90ee is healthy: got healthy result from https://192.168.1.20:2379
cluster is healthy
```
