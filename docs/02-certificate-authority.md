# Setting up a Certificate Authority and TLS Cert Generation

In this lab you will setup the necessary PKI infrastructure to secure the Kubernetes components. This lab will leverage CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), to bootstrap a Certificate Authority and generate TLS certificates.

In this lab you will generate a single set of TLS certificates that can be used to secure the following Kubernetes components:

* etcd
* Kubernetes API Server
* Kubernetes Kubelet

> In production you should strongly consider generating individual TLS certificates for each component.

After completing this lab you should have the following TLS keys and certificates:

```
ca-key.pem
ca.pem
kubernetes-key.pem
kubernetes.pem
```


## Install CFSSL

This lab requires the `cfssl` and `cfssljson` binaries. Download them from the [cfssl repository](https://pkg.cfssl.org).

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-arm
chmod +x cfssl_linux-amd64
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
```

```
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-arm
chmod +x cfssljson_linux-amd64
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

## Setting up a Certificate Authority

### Create the CA configuration file

```
echo '{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}' > ca-config.json
```

### Generate the CA certificate and private key

Create the CA CSR:

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
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
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
ca-key.pem
ca.csr
ca.pem
```

### Verification

```
openssl x509 -in ca.pem -text -noout
```

## Generate the single Kubernetes TLS Cert

In this section we will generate a TLS certificate that will be valid for all Kubernetes components. This is being done for ease of use. In production you should strongly consider generating individual TLS certificates for each component. (But all replicas of a given component must share the same certificate.)


Create the `kubernetes-csr.json` file:

```
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "worker0",
    "worker1",
    "worker2",
    "ip-10-240-0-20",
    "ip-10-240-0-21",
    "ip-10-240-0-22",
    "10.32.0.1",
    "10.240.0.10",
    "10.240.0.11",
    "10.240.0.12",
    "10.240.0.20",
    "10.240.0.21",
    "10.240.0.22",
    "${KUBERNETES_PUBLIC_ADDRESS}",
    "127.0.0.1"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "Cluster",
      "ST": "Oregon"
    }
  ]
}
EOF
```

Generate the Kubernetes certificate and private key:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

Results:

```
kubernetes-key.pem
kubernetes.csr
kubernetes.pem
```

### Verification

```
openssl x509 -in kubernetes.pem -text -noout
```

## Copy TLS Certs

Set the list of Kubernetes hosts where the certs should be copied to:

```
KUBERNETES_HOSTS=(controller0 controller1 controller2 worker0 worker1 worker2)
```

```
for host in ${KUBERNETES_HOSTS[*]}; do
  PUBLIC_IP_ADDRESS=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=${host}" | \
    jq -r '.Reservations[].Instances[].PublicIpAddress')
  scp ca.pem kubernetes-key.pem kubernetes.pem \
    ubuntu@${PUBLIC_IP_ADDRESS}:~/
done
```

