# Kubernetes The Hard Way On OpenStack

An adaptation of Kelsey Hightower's infamous [Kubernetes The Hard Way lab](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/79a3f79b27bd28f82f071bb877a266c2e62ee506/docs/14-cleanup.md) for the CERN OpenStack environment and updated to the latest components:

* [etcd v3.5.5](https://github.com/etcd-io/etcd/releases/tag/v3.5.5)
* [Kubernetes v1.25.3](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.25.md)
* [containerd v1.6.9](https://github.com/containerd/containerd/releases/tag/v1.6.9)
* [runc 1.1.4](https://github.com/opencontainers/runc/releases/tag/v1.1.4)
* [CNI plugins v1.1.1](https://github.com/containernetworking/plugins/releases/tag/v1.1.1)

Required tools:

* openstack CLI
* jq

The labs omit the networking steps (VPC, floating IP, security groups) since these are not available in the CERN OpenStack Cloud.
