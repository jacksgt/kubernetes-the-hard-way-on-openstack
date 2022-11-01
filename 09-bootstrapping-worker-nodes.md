# Bootstrapping the Kubernetes Worker Nodes

<https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/79a3f79b27bd28f82f071bb877a266c2e62ee506/docs/09-bootstrapping-kubernetes-workers.md>

Note: I'm just using one worker node because it serves the same purpose.

## Provision Kubernetes Worker Node

```sh
K8S_VERSION=v1.25.3
ssh ubuntu@jh-k8s-worker-1 sh -ex - <<EOF
    sudo apt-get update
    sudo apt-get install -y socat conntrack ipset
    sudo swapoff -va
    wget -q --https-only --timestamping     https://dl.k8s.io/${K8S_VERSION}/kubernetes-node-linux-amd64.tar.gz
    tar xf kubernetes-node-linux-amd64.tar.gz
    sudo mv kubernetes/node/bin/kubectl kubernetes/node/bin/kubelet kubernetes/node/bin/kube-proxy /usr/local/bin/
    sudo mkdir -p \
         /etc/cni/net.d \
         /opt/cni/bin \
         /var/lib/kubelet \
         /var/lib/kube-proxy \
         /var/lib/kubernetes \
         /var/run/kubernetes
    mkdir -p containerd
    wget -q --https-only --timestamping https://github.com/containerd/containerd/releases/download/v1.6.9/containerd-1.6.9-linux-amd64.tar.gz
    tar -xvf containerd-1.6.9-linux-amd64.tar.gz -C containerd
    sudo mv containerd/bin/* /usr/local/bin/

    wget -q --https-only --timestamping https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
    sudo tar -xvf cni-plugins-linux-amd64-v1.1.1.tgz -C /opt/cni/bin/

    wget -q --https-only --timestamping https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.25.0/crictl-v1.25.0-linux-amd64.tar.gz
    tar -xvf crictl-v1.25.0-linux-amd64.tar.gz
    sudo mv crictl /usr/local/bin/

    wget -q --https-only --timestamping https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64
    chmod +x runc.amd64
    sudo mv runc.amd64 /usr/local/bin/runc
EOF
```

## Configure CNI Networking

```sh
ssh ubuntu@jh-k8s-worker-1 sh -x - <<EOF
cat <<DELIM | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.4.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "10.200.1.0/24"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
DELIM

cat <<DELIM | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.4.0",
    "name": "lo",
    "type": "loopback"
}
DELIM

EOF
```

## Configure containerd

```sh
ssh ubuntu@jh-k8s-worker-1 sh -ex - <<EOF
    sudo mkdir -p /etc/containerd/
    cat <<DELIM | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
DELIM

    cat <<DELIM | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
DELIM

EOF
```

## Configure the Kubelet

```sh
instance=jh-k8s-worker-1
ssh ubuntu@${instance} sh -x - <<EOF
    sudo mv ${instance}-key.pem ${instance}.pem /var/lib/kubelet/
    sudo mv ${instance}.kubeconfig /var/lib/kubelet/kubeconfig
    sudo mv ca.pem /var/lib/kubernetes/

cat <<DELIM | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "10.200.1.0/24"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${instance}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${instance}-key.pem"
DELIM

cat <<DELIM | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
DELIM

EOF
```

## Configure the Kube Proxy

```sh
ssh ubuntu@jh-k8s-worker-1 sh -ex - <<EOF
    sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
    cat <<DELIM | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
DELIM

    cat <<DELIM | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
DELIM

EOF
```

## Start the worker services

```sh
for instance in jh-k8s-worker-1; do
    ssh ubuntu@${instance} sh -xe - <<EOF
    sudo systemctl daemon-reload
    sudo systemctl enable --now containerd kubelet kube-proxy
    sudo systemctl status containerd kubelet kube-proxy
EOF
done
```

## Verification

```sh
ssh ubuntu@jh-k8s-cp-1 kubectl get node --kubeconfig admin.kubeconfig
NAME              STATUS   ROLES    AGE   VERSION
jh-k8s-worker-1   Ready    <none>   47s   v1.25.3
```
