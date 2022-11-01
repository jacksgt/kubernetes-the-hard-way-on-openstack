# Bootstrapping the etcd Cluster

<https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/79a3f79b27bd28f82f071bb877a266c2e62ee506/docs/07-bootstrapping-etcd.md>

## Download and install etcd binaries

```sh
ETCD_VERSION=v3.5.5
for instance in jh-k8s-cp-{1..3}; do
    ssh ubuntu@${instance} sh - <<EOF
    wget -q --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz"
  tar -xvf etcd-${ETCD_VERSION}-linux-amd64.tar.gz
  sudo mv etcd-${ETCD_VERSION}-linux-amd64/etcd* /usr/local/bin/
EOF
done
```

## Configure etcd server

```sh
INITIAL_CLUSTER=''
for instance in jh-k8s-cp-{1..3}; do
    INITIAL_CLUSTER+="${instance}=https://$(host ${instance} | awk '{print $4}'):2380,"
done
for instance in jh-k8s-cp-{1..3}; do
    INTERNAL_IP=$(host ${instance} | awk '{print $4 }')
    ETCD_NAME=${instance}
    ssh ubuntu@${instance} sh - <<EOF
    sudo mkdir -p /etc/etcd /var/lib/etcd
    sudo chmod 700 /var/lib/etcd
    sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/

    cat <<DELIM | sudo tee /etc/systemd/system/etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster ${INITIAL_CLUSTER} \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
DELIM

EOF
done
```

## Start the etcd server

```sh
for instance in jh-k8s-cp-{1..3}; do
    ssh ubuntu@${instance} sh -x - <<EOF
    sudo systemctl daemon-reload
    sudo systemctl enable --now etcd
    sudo systemctl status etcd

EOF
done
```

## Verification

```sh

ssh ubuntu@jh-k8s-cp-3 sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

## Reset

(in case something went wrong an etcd did start cleanly)

```sh
for instance in jh-k8s-cp-{1..3}; do
    ssh ubuntu@${instance} sh -x - <<EOF
    sudo systemctl stop etcd
    sudo rm -rf /var/lib/etcd /etc/etcd /etc/systemd/system/etcd.service
EOF
done
```
