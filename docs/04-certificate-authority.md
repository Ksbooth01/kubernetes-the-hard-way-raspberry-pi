# Setting up a Certificate Authority and TLS Cert Generation

In this lab you will setup the necessary PKI infrastructure to secure the Kubernetes components. This lab will leverage CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), to bootstrap a Certificate Authority and generate TLS certificates.

In this lab you will generate a single set of TLS certificates that can be used to secure the following Kubernetes components:

* etcd
* Kubernetes API Server
* Kubernetes Kubelet

The steps can be followed on one of the Raspberry Pis, then the certificates can be distributed to the others. I'm using **loadbalancer** for the steps and then I copy the certificates to the rest of the cluster.


## Make the Working Directory

```
mkdir -p $HOME/kubernetes
cd $HOME/kubernetes
```

## Install CFSSL

We will require the `cfssl` and `cfssljson` binaries. Download them from the [cfssl repository](https://pkg.cfssl.org).

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-arm
chmod +x cfssl_linux-arm
sudo mv cfssl_linux-arm /usr/local/bin/cfssl
```
...and now the cfssljson binaries
```
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-arm
chmod +x cfssljson_linux-arm
sudo mv cfssljson_linux-arm /usr/local/bin/cfssljson
````

## Setting up a Certificate Authority

### Create the CA configuration file

```
echo '{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "87600h"
      }
    }
  }
}' > ca-config.json
```

### Generate the CA certificate and private key

A certificate signing request (CSR) is one of the first steps towards getting your own SSL Certificate. Generated on the same server you plan to install the certificate on, the CSR contains information (e.g. common name, organization, country) the Certificate Authority (CA) will use to create your certificate. It also contains the public key that will be included in your certificate and is signed with the corresponding private key.
To create the CA CSR template and Generate the CA certificate with private key:

```
echo '{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Fawn Grove",
      "O": "Kubernetes",
      "OU": "PA",
      "ST": "Pennsylvania"
    }
  ]
}' > ca-csr.json

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```
### Results:

```
ca-key.pem   (the Private key for the CA)
ca.csr       (the certificate signing request)
ca.pem       (the Public key for the CA)
```

### Verification

```
openssl x509 -in ca.pem -text -noout
```
## Create Certificates for the Kubernetes Services


Creating the Admin Client certificate:
```
{
cat > admin-csr.json << EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Fawn Grove",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way - Pi edition",
      "ST": "Pennsylvania"
    }
  ]
} 
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin

}

```
Next we'll be creating the Kubelet Client certificates. Be sure to enter YOUR actual cloud server values for all four of the variables at the top:
```
WORKER1_HOST=<PUBLIC hostname of your first worker node>
WORKER1_IP=<PRIVATE IP of your first worker node>
WORKER2_HOST=<PUBLIC hostname of your second worker node>
WORKER2_IP=<PRIVATE IP of your second worker node>
```

```
WORKER1_HOST=worker-1
WORKER1_IP=172.16.0.21
WORKER2_HOST=worker-2
WORKER2_IP=172.16.0.22


cat > ${WORKER1_HOST}-csr.json << EOF
{
  "CN": "system:node:${WORKER1_HOST}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Fawn Grove",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way - Pi Edition",
      "ST": "Pennsylvania"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${WORKER1_IP},${WORKER1_HOST} \
  -profile=kubernetes \
  ${WORKER1_HOST}-csr.json | cfssljson -bare ${WORKER1_HOST}

cat > ${WORKER2_HOST}-csr.json << EOF
{
  "CN": "system:node:${WORKER2_HOST}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Fawn Grove",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way - Pi Edition",
      "ST": "Pennsylvania"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${WORKER2_IP},${WORKER2_HOST} \
  -profile=kubernetes \
  ${WORKER2_HOST}-csr.json | cfssljson -bare ${WORKER2_HOST}

}
```

Controller Manager Client certificate:

```
{

cat > kube-controller-manager-csr.json << EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Fawn Grove",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way - Pi Edition",
      "ST": "Pennsylvania"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

}
```
Kube Proxy Client certificate: these will be used on the worker nodes.
```
{

cat > kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Fawn Grove",
      "O": "system:node-proxy",
      "OU": "Kubernetes The Hard Way - Pi Edition",
      "ST": "Pennsylvania"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy

}
```

The Kube Scheduler Client Certificate.  This will be ued on the Control Servers to authenticat the Kubernetes schedulte service.
```
{

cat > kube-scheduler-csr.json << EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Fawn Grove",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way - Pi Edition",
      "ST": "Pennsylvania"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler

}
```
### The Kubernetes API Server Certificate
This will create a server certificate for the Kubernetes API. The script will generate one, signed with all of the hostnames and IPs that may be used to access the Kubernetes API. Once completed you will have a Kubernetes API server certificate in the form of two files called ```kubernetes-key.pem``` and ```kubernetes.pem```.


CERT_HOSTNAME=10.32.0.1,<controller node 1 IP>,<controller node 1 hostname>,<controller node 2 IP>,<controller node 2 hostname>,<API load balancer IP>,<API load balancer hostname>,127.0.0.1,localhost,kubernetes.default
  
**Some explaination:**
* 10.32.0.1                  - the default IP address of the Kubernetes API service on the Kubernetes service network 10.32.0.0/24
* kubernetes.default         - the hostanme for the Kurernetes API service
* controller node 1 IP       - the PRIVATE IP address of `controller-0`. **Note:** if you have two IP's on your system, this is the one NOT going to the internet
* controller node 1 hostname - `controller-0 hostane`
* controller node 2 IP       - the PRIVATE IP address of `controller-1`
* controller node 2 hostname - `controller-1 hostname`
* API load balancer IP       - the PRIVATE IP address for the load balancer
* API load balancer hostname - the hostname for the load balancer
* 127.0.0.1                  - the IP address for loopback, local communication
* localhost                  - the hostname for loopback, local communication

Make sure you include all of the information for all your servers.

```
CERT_HOSTNAME=10.32.0.1,172.16.0.20,controller-0,172.16.0.40,controller-1,172.16.0.30,loadbalancer,127.0.0.1,localhost,kubernetes.default

{

cat > kubernetes-csr.json << EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Fawn Grove",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way - Pi edition",
      "ST": "Pennsylvania"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${CERT_HOSTNAME} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes

}
```
#### Output
  ```
  <date> <time> [INFO] generate received request
  <date> <time> [INFO] received CSR 
  <date> <time> [INFO] generating key: rsa-2048
  <date> <time> [INFO] encoded CSR
  <date> <time> [INFO] signed certificate with serial number ################################################
  [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for websites. For more information see the Baseline Requirements 
  for the Issuance and Management of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
  specifically, section 10.2.3 ("Information Requirements").
  ```
#### The Service Account Key Pair

Kubernetes provides the ability for service accounts to authenticate using tokens. It uses a key-pair to provide signatures for those tokens. We will generate a certificate that will be used as that key-pair. Upon completion, you will have a certificate ready to be used as a service account key-pair in the form of two files: `service-account-key.pem` and `service-account.pem`.

```
{

cat > service-account-csr.json << EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Fawn Grove",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way - Pi Edition",
      "ST": "Pennsylvania"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account

}
```

## Copy TLS Certs to their appropriate Servers

##### Move certificate files to the worker nodes:

```
scp ca.pem admin.pem admin-key.pem ${WORKER1_HOST}-key.pem ${WORKER1_HOST}.pem kubeadmin@${WORKER1_IP}:~/
scp ca.pem admin.pem admin-key.pem ${WORKER1_HOST}-key.pem ${WORKER2_HOST}.pem kubeadmin@${WORKER2_IP}:~/
```
##### Move certificate files to the controller nodes:

```
CONTROLLER0_IP=<IP of your second worker node>
CONTROLLER1_IP=<IP of your second worker node>

CONTROLLER0_IP=192.168.1.20
CONTROLLER1_IP=192.168.1.40

scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem kubeadmin@${CONTROLLER0_IP}:~/
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem kubeadmin@${CONTROLLER1_IP}:~/
Set the list of Kubernetes hosts where the certs should be copied to:
```

Next: [Generating Kubernetes Configuration files for Authentication](05-kubernetes-configuration-files.md)
