# Provision PKI infrastructure

This step is pretty much the same as the original guide except for a few tweaks:

* Internal and external IPs are different from the proposed computing resources.
* Hostnames are different from the proposed scheme.
* Only one master node in this setup, therefore only one IP to look after for etcd.
* I've updated the TLS fields to my location in Manchester, England.

All commands will be run in `~/pki` directory.

## Certificate Authority

```shell
cat > ca-config.json <<EOF
{
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
}
EOF
```

```shell
cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CZ",
      "L": "Dvur Kralove nad Labem",
      "O": "Kubernetes",
      "OU": "CZ",
      "ST": "Czech Republic"
    }
  ]
}
EOF
```

```shell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
2022/08/15 12:01:56 [INFO] generating a new CA key and certificate from CSR
2022/08/15 12:01:56 [INFO] generate received request
2022/08/15 12:01:56 [INFO] received CSR
2022/08/15 12:01:56 [INFO] generating key: rsa-2048
2022/08/15 12:01:56 [INFO] encoded CSR
2022/08/15 12:01:56 [INFO] signed certificate with serial number 287296973733057800908098566923579012820171233503
```

## The Admin Client Certificate

```shell
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CZ",
      "L": "Dvur Kralove nad Labem",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Czech Republic"
    }
  ]
}
EOF
```

```shell
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
2022/08/15 12:03:32 [INFO] generate received request
2022/08/15 12:03:32 [INFO] received CSR
2022/08/15 12:03:32 [INFO] generating key: rsa-2048
2022/08/15 12:03:33 [INFO] encoded CSR
2022/08/15 12:03:33 [INFO] signed certificate with serial number 568976151753147020374438622124698179997553641817
```

## The Kubelet Client Certificates

One client certificate for each worker node.

```shell
for instance in 1 2 3; do
cat > p${instance}-csr.json <<EOF
{
  "CN": "system:node:p${instance}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CZ",
      "L": "Dvur Kralove nad Labem",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Czech Republic"
    }
  ]
}
EOF

INTERNAL_IP=172.19.181.${instance}

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=p${instance},${INTERNAL_IP} \
  -profile=kubernetes \
  p${instance}-csr.json | cfssljson -bare p${instance}
done
2022/08/15 12:06:28 [INFO] generate received request
2022/08/15 12:06:28 [INFO] received CSR
2022/08/15 12:06:28 [INFO] generating key: rsa-2048
2022/08/15 12:06:29 [INFO] encoded CSR
2022/08/15 12:06:29 [INFO] signed certificate with serial number 28104006092254220968970681420606377282132748960
2022/08/15 12:06:29 [INFO] generate received request
2022/08/15 12:06:29 [INFO] received CSR
2022/08/15 12:06:29 [INFO] generating key: rsa-2048
2022/08/15 12:06:30 [INFO] encoded CSR
2022/08/15 12:06:30 [INFO] signed certificate with serial number 138709888965617123780314033893249702741536600126
2022/08/15 12:06:30 [INFO] generate received request
2022/08/15 12:06:30 [INFO] received CSR
2022/08/15 12:06:30 [INFO] generating key: rsa-2048
2022/08/15 12:06:31 [INFO] encoded CSR
2022/08/15 12:06:31 [INFO] signed certificate with serial number 9635456406402900043651968347865299440845422917
```

## The Controller Manager Client Certificate

```shell
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CZ",
      "L": "Dvur Kralove nad Labem",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Czech Republic"
    }
  ]
}
EOF
```

```shell
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
2022/08/15 12:08:02 [INFO] generate received request
2022/08/15 12:08:02 [INFO] received CSR
2022/08/15 12:08:02 [INFO] generating key: rsa-2048
2022/08/15 12:08:03 [INFO] encoded CSR
2022/08/15 12:08:03 [INFO] signed certificate with serial number 248012867203482239945857713390021855993203565164
```

## The Kube Proxy Client Certificate

```shell
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CZ",
      "L": "Dvur Kralove nad Labem",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Czech Republic"
    }
  ]
}
EOF
```

```shell
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
2022/08/15 12:09:25 [INFO] generate received request
2022/08/15 12:09:25 [INFO] received CSR
2022/08/15 12:09:25 [INFO] generating key: rsa-2048
2022/08/15 12:09:26 [INFO] encoded CSR
2022/08/15 12:09:26 [INFO] signed certificate with serial number 683231017079990333078954987023637962650458066638
```

## The Scheduler Client Certificate

```shell
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CZ",
      "L": "Dvur Kralove nad Labem",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Czech Republic"
    }
  ]
}
EOF
```

```shell
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
2022/08/15 12:11:24 [INFO] generate received request
2022/08/15 12:11:24 [INFO] received CSR
2022/08/15 12:11:24 [INFO] generating key: rsa-2048
2022/08/15 12:11:25 [INFO] encoded CSR
2022/08/15 12:11:25 [INFO] signed certificate with serial number 684196441443145125050997355197089393129383748736
```

## The Kubernetes API Server Certificate

Don't forget to include the IP `10.32.0.1` as it's used by the ClusterIP service and CoreDNS will connect to it. I discovered this the hard way.

```
KUBERNETES_PUBLIC_ADDRESS=192.168.5.150
KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local
```

```shell
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CZ",
      "L": "Dvur Kralove nad Labem",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Czech Republic"
    }
  ]
}
EOF
```

```shell
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,172.19.181.254,rpi-k8s-master,rpi-k8s-master.local,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
2022/08/15 12:14:17 [INFO] generate received request
2022/08/15 12:14:17 [INFO] received CSR
2022/08/15 12:14:17 [INFO] generating key: rsa-2048
2022/08/15 12:14:19 [INFO] encoded CSR
2022/08/15 12:14:19 [INFO] signed certificate with serial number 239594823383694161338796346213592088785025148372
```

## The Service Account Key Pair

```shell
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CZ",
      "L": "Dvur Kralove nad Labem",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Czech Republic"
    }
  ]
}
EOF
```

```shell
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
2022/08/15 12:15:31 [INFO] generate received request
2022/08/15 12:15:31 [INFO] received CSR
2022/08/15 12:15:31 [INFO] generating key: rsa-2048
2022/08/15 12:15:31 [INFO] encoded CSR
2022/08/15 12:15:31 [INFO] signed certificate with serial number 101835948591957890898991252630745904812705776871
```

## Distribute the Files

```shell
for instance in p1 p2 p3; do
  ssh ${instance} "mkdir certs"
  scp ca.pem ${instance}-key.pem ${instance}.pem ${instance}:~/certs/
done

cp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem service-account-key.pem service-account.pem ../certs/
```
