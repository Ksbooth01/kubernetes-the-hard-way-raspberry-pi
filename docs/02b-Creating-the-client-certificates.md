

#### The following tasks are being completed on the server hosting fsSSL. 

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

{
cat > ${NODE1_HOST}-csr.json << EOF
{
  "CN": "system:node:${
  NODE1_HOST}",
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
  -hostname=$(NODE1_IP},${NODE1_HOST} \
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
  ${WORKER1_HOST}-csr.json | cfssljson -bare ${NODE2_HOST}

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

