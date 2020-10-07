# Setting up a Certificate Authority and TLS Cert Generation

In this lab you will setup the necessary PKI infrastructure to secure the Kubernetes components. This lab will leverage CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), to bootstrap a Certificate Authority and generate TLS certificates.

In this lab you will generate a single set of TLS certificates that can be used to secure the following Kubernetes components:

* etcd
* Kubernetes API Server
* Kubernetes Kubelet

> In production you should strongly consider generating individual TLS certificates for each component.

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

```
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-arm
chmod +x cfssljson_linux-arm
sudo mv cfssljson_linux-arm /usr/local/bin/cfssljson
```

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
To create the CA CSR:

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
```

Generate the CA certificate and private key:

```
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Results:

```
ca-key.pem   (the Private key for the CA)
ca.csr
ca.pem       (the Public key for the CA)
```

### Verification

```
openssl x509 -in ca.pem -text -noout
```

## Copy TLS Certs

Set the list of Kubernetes hosts where the certs should be copied to:

```
KUBERNETES_HOSTS=(controller-0 controller-1 node1 node2)
```

```
for host in ${KUBERNETES_HOSTS[*]}; do
  ssh ${host} "mkdir -p ~/kubernetes"
  scp ca.pem kubernetes-key.pem kubernetes.pem ${host}  ~/kubernetes
done
```


