# Setting up a Certificate Authority and TLS Cert Generation

In this lab you will setup the necessary PKI infrastructure to secure the Kubernetes components. This lab will leverage CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), to bootstrap a Certificate Authority and generate TLS certificates.

In this lab you will generate a single set of TLS certificates that can be used to secure the following Kubernetes components:

* etcd
* Kubernetes API Server
* Kubernetes Kubelet

The steps can be followed on one of the Raspberry Pis, then the certificates can be distributed to the others. I'm using **controller0** for the steps and then I copy the certificates to the rest of the cluster.

After completing this lab you should have the following TLS keys and certificates:

```
ca-key.pem         (the private key for the CA)
ca.pem             (the public key for the CA)
kubernetes-key.pem (the private key )
kubernetes.pem     (the public key 
```

## Working Directory

```
mkdir -p $HOME/kubernetes
cd $HOME/kubernetes
```

## Install CFSSL

This lab requires the `cfssl` and `cfssljson` binaries. Download them from the [cfssl repository](https://pkg.cfssl.org).

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
ca.csr
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
NODE1_HOST=<hostname of your first worker node>
NODE1_IP=<IP of your first worker node>
NODE2_HOST=<hostname of your second worker node>
NODE2_IP=<IP of your second worker node>
```

```
NODE1_HOST=node1
NODE1_IP=192.168.1.21
NODE2_HOST=node2
NODE2_IP=192.168.1.22


cat > ${NODE1_HOST}-csr.json << EOF
{
  "CN": "system:node:${NODE1_HOST}",
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
  -hostname=${NODE1_IP},${NODE1_HOST} \
  -profile=kubernetes \
  ${NODE1_HOST}-csr.json | cfssljson -bare ${NODE1_HOST}

cat > ${NODE2_HOST}-csr.json << EOF
{
  "CN": "system:node:${NODE2_HOST}",
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
  -hostname=${NODE2_IP},${NODE2_HOST} \
  -profile=kubernetes \
  ${NODE2_HOST}-csr.json | cfssljson -bare ${NODE2_HOST}

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
* controller node 1 IP       - the IP address of `controller-0`. **Note:** if you have two IP's on your system, this is the one NOT going to the internet
* controller node 1 hostname - `controller-0`
* controller node 2 IP       - the IP address of `controller-1`
* controller node 2 hostname - `controller-0`
* API load balancer IP       - the IP address for the load balancer
* API load balancer hostname - the hostname for the load balancer
* 127.0.0.1                  - the IP address for loopback, local communication
* localhost                  - the hostname for loopback, local communication

Make sure you include all of the information for all your servers.

```
CERT_HOSTNAME=10.32.0.1,192.168.1.20,controller-0,192.168.1.40,controller-1,192.168.1.30,loadbalancer,127.0.0.1,localhost,kubernetes.default

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
scp ca.pem ${NODE1_HOST}-key.pem ${NODE1_HOST}.pem kubeadmin@${NODE1_IP}:~/
scp ca.pem ${NODE1_HOST}-key.pem ${NODE2_HOST}.pem kubeadmin@${NODE2_IP}:~/
```
##### Move certificate files to the controller nodes:

```
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem kubeadmin@<controller-0 IP>:~/
scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem kubeadmin@<controller-1 IP>:~/
Set the list of Kubernetes hosts where the certs should be copied to:


