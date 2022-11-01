# Bootstrapping the Kubernetes Control Plane

<https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/79a3f79b27bd28f82f071bb877a266c2e62ee506/docs/08-bootstrapping-kubernetes-controllers.md>

## Install the components

```sh
K8S_VERSION=v1.25.3
for instance in jh-k8s-cp-{1..3}; do
    ssh ubuntu@${instance} sh -x - <<EOF
    sudo mkdir -p /etc/kubernetes/config
    wget -q --https-only --timestamping "https://dl.k8s.io/${K8S_VERSION}/kubernetes-server-linux-amd64.tar.gz"
    tar xf kubernetes-server-linux-amd64.tar.gz
    sudo mv kubernetes/server/bin/kube-apiserver /usr/local/bin/
    sudo mv kubernetes/server/bin/kube-controller-manager /usr/local/bin/
    sudo mv kubernetes/server/bin/kube-scheduler /usr/local/bin/
    sudo mv kubernetes/server/bin/kubectl /usr/local/bin/    x
EOF
done
```

## Configure the Kubernetes API server

```sh
KUBERNETES_PUBLIC_ADDRESS=$(cat k8s-api-address.txt)
ETCD_SERVERS=''
for instance in jh-k8s-cp-{1..3}; do
    ETCD_SERVERS+="https://$(host ${instance} | awk '{print $4}'):2379,"
done
ETCD_SERVERS=${ETCD_SERVERS%,}

for instance in jh-k8s-cp-{1..3}; do
    INTERNAL_IP=$(host ${instance} | awk '{print $4 }')
    ssh ubuntu@${instance} sh -x - <<EOF
    sudo mkdir -p /var/lib/kubernetes
    sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
        service-account-key.pem service-account.pem encryption-config.yaml \
        /var/lib/kubernetes/

    cat <<DELIM | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=3 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=${ETCD_SERVERS} \\
  --event-ttl=1h \\
  --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --runtime-config='api/all=true' \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-account-signing-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-account-issuer=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
DELIM

EOF
done
```

## Configure the Kubernetes Controller Manager

```sh
for instance in jh-k8s-cp-{1..3}; do
    ssh ubuntu@${instance} sh -x - <<EOF
    sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
    cat <<DELIM | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
DELIM

EOF
done
```

## Configure the Kubernetes Scheduler

```sh
for instance in jh-k8s-cp-{1..3}; do
    ssh ubuntu@${instance} sh -x - <<EOF
    sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
    cat <<DELIM | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
DELIM

cat <<DELIM | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
DELIM

EOF
done
```

Adjusted to `v1` due to <https://github.com/kelseyhightower/kubernetes-the-hard-way/issues/687>

Latest reference for kube scheduler config: https://kubernetes.io/docs/reference/scheduling/config/

## Start the Controller Services

```sh
for instance in jh-k8s-cp-{1..3}; do
    ssh ubuntu@${instance} sh -x - <<EOF
    sudo systemctl daemon-reload
    sudo systemctl enable --now kube-apiserver kube-controller-manager kube-scheduler
EOF
done
```

## Verification

```sh
ssh ubuntu@jh-k8s-cp-1 sudo systemctl status kube-apiserver;
ssh ubuntu@jh-k8s-cp-2 sudo systemctl status kube-scheduler;
ssh ubuntu@jh-k8s-cp-3 sudo systemctl status kube-controller-manager;

ssh ubuntu@jh-k8s-cp-1 kubectl get ns --kubeconfig admin.kubeconfig
NAME              STATUS   AGE
default           Active   19m
kube-node-lease   Active   19m
kube-public       Active   19m
kube-system       Active   19m
```

*It's alive!*

## RBAC for Kubelet Authorization

```sh
ssh ubuntu@jh-k8s-cp-1

cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF

cat <<EOF | kubectl apply --kubeconfig admin.kubeconfig -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF
```

## The Kubernetes Frontend Load Balancer

Add members to loadbalancer created in 03-compute-resources.md

```sh
for instance in jh-k8s-cp-{1..3}; do
    IP=$(host ${instance} | awk '{print $4}')
    OS_REGION_NAME=sdn1 openstack loadbalancer member create --name ${instance} --address ${IP} --protocol-port 6443 k8s-api-servers
done
```

## Verification

```sh
curl --cacert ca.pem https://${KUBERNETES_PUBLIC_ADDRESS}:6443/version
{
  "major": "1",
  "minor": "25",
  "gitVersion": "v1.25.3",
  "gitCommit": "434bfd82814af038ad94d62ebe59b133fcb50506",
  "gitTreeState": "clean",
  "buildDate": "2022-10-12T10:49:09Z",
  "goVersion": "go1.19.2",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```
