# Provisioning a CA and Generating TLS Certificates

<https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/79a3f79b27bd28f82f071bb877a266c2e62ee506/docs/04-certificate-authority.md>

```sh
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

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CH",
      "L": "Geneva",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Geneva"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

## The Admin Client Certificate

```sh
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CH",
      "L": "Geneva",
      "O": "system:masters",
      "OU": "Kubernetes The Hard Way",
      "ST": "Geneva"
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
```

## The Kubelet Client Certificates

```sh
for instance in jh-k8s-worker-1; do
  cat > ${instance}-csr.json <<EOF
  {
    "CN": "system:node:${instance}",
    "key": {
      "algo": "rsa",
      "size": 2048
    },
    "names": [
      {
        "C": "CH",
        "L": "Geneva",
        "O": "system:nodes",
        "OU": "Kubernetes The Hard Way",
        "ST": "Geneva"
      }
    ]
  }
EOF

  IP=$(openstack server show ${instance} -f json | jq -r '.addresses' | cut -d'=' -f2 | cut -d',' -f1)

  cfssl gencert \
    -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -hostname=${instance},${IP} \
    -profile=kubernetes \
    ${instance}-csr.json | cfssljson -bare ${instance}

done
```

## The Controller Manager Client Certificate

```sh

cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CH",
      "L": "Geneva",
      "O": "system:kube-controller-manager",
      "OU": "Kubernetes The Hard Way",
      "ST": "Geneva"
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
```

## The Kube Proxy Client Certificate

```sh
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CH",
      "L": "Geneva",
      "O": "system:node-proxier",
      "OU": "Kubernetes The Hard Way",
      "ST": "Geneva"
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
```

## The Scheduler Client Certificate

```sh
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
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
```

## The Kubernetes API Server Certificate

```sh
KUBERNETES_PUBLIC_ADDRESS=$(cat k8s-api-address.txt)

KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

KUBERNETES_API_HOSTS=$(for instance in jh-k8s-cp-{1..3}; do echo -n $(host ${instance} | awk '{print $4}'),; done)

cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CH",
      "L": "Geneva",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Geneva"
    }
  ]
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${KUBERNETES_API_HOSTS}${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

## The Service Account Key Pair

```sh
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CH",
      "L": "Geneva",
      "O": "Kubernetes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Geneva"
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
```

## Distribute certificates

```sh
for instance in jh-k8s-worker-1; do
    scp ca.pem ${instance}-key.pem ${instance}.pem ubuntu@${instance}:.
done

for instance in jh-k8s-cp-{1..3}; do
  scp ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem ubuntu@${instance}:.
done
````
