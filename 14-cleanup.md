# Cleaning Up

<https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/79a3f79b27bd28f82f071bb877a266c2e62ee506/docs/14-cleanup.md>

## Compute Instances

```sh
openstack server delete jh-k8s-cp-{1..3} jh-k8s-worker-1 # jh-k8s-worker-2 jh-k8s-worker-3
```

## Loadbalancer

```sh
export OS_REGION_NAME=sdn1

openstack loadbalancer pool list | grep jh-k8s
openstack loadbalancer pool delete <UUID>

openstack loadbalancer listener list | grep jh-k8s
openstack loadbalancer listener delete <UUID>

openstack loadbalancer list | grep jh-k8s
openstack loadbalancer delete <UUID>
```
