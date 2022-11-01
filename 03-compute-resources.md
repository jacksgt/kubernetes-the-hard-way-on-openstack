# Provisioning Compute Resources

<https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/79a3f79b27bd28f82f071bb877a266c2e62ee506/docs/03-compute-resources.md>

## Kubernetes Public IP Address

Create a loadbalancer which we can use to have a floating IP in front of our API.

```sh
ORIG_REGION_NAME=$OS_REGION_NAME
export OS_REGION_NAME=sdn1
openstack loadbalancer create --name jh-k8s-lb --vip-network-id public
openstack loadbalancer listener create --name k8s-api --protocol TCP --protocol-port 6443 jh-k8s-lb
openstack loadbalancer pool create --name k8s-api-servers --listener k8s-api --protocol TCP --lb-algorithm ROUND_ROBIN --session-persistence type=SOURCE_IP
openstack loadbalancer show jh-k8s-lb -f json | jq -r .vip_address > k8s-api-address.txt
# members will be added later

export OS_REGION_NAME=$ORIG_REGION_NAME
```

## Kubernetes Controllers

Find the UUID of a suitable Ubuntu image:

```sh
openstack image list --community | grep Ubuntu
IMAGE_UUID=f3799482-e852-4a77-9383-408370b81df6 # Ubuntu 20.04
```

```sh
cat - > ubuntu-systemd-resolved.yaml <<EOF
#cloud-config
bootcmd:
- printf "[Resolve]\nDNS=137.138.16.5 137.138.17.5\nDomains=cern.ch\n" > /etc/systemd/resolved.conf
- [systemctl, restart, systemd-resolved]
EOF

openstack key create jh --public-key ~/.ssh/id_rsa.pub

for i in {1..3}; do
    openstack server create "jh-k8s-cp-${i}" --user-data ubuntu-systemd-resolved.yaml --key-name jh --flavor m2.large --image "$IMAGE_UUID"
done
```

## Kubernetes Workers

```sh
for i in 1; do
    openstack server create "jh-k8s-worker-${i}" --user-data ubuntu-systemd-resolved.yaml --key-name jh --flavor m2.large --image "$IMAGE_UUID"
done
```

## Verification

```sh
$ openstack server list | grep jh-k8s-
| 22418beb-93b4-4e70-b039-8b419d4e3535 | jh-k8s-cp-3                      | ACTIVE | CERN_NETWORK=188.184.102.10, 2001:1458:d00:3b::100:41e  |                                                | m2.large  |
| 724858f3-e0b7-48a8-bf46-14cb9189e950 | jh-k8s-cp-2                      | ACTIVE | CERN_NETWORK=188.185.4.74, 2001:1458:d00:62::100:407    |                                                | m2.large  |
| 19c3341e-8927-4912-b7d5-d4e2720dbb04 | jh-k8s-cp-1                      | ACTIVE | CERN_NETWORK=188.185.37.79, 2001:1458:d00:6a::100:423   |                                                | m2.large  |
| 0df2e646-6e00-4657-bc17-4919063aaa63 | jh-k8s-worker-1                  | ACTIVE | CERN_NETWORK=188.185.29.70, 2001:1458:d00:68::100:261   |                                                | m2.large  |

$ ssh ubuntu@jh-k8s-worker-1
The authenticity of host 'jh-k8s-worker-1 (188.185.29.70)' can't be established.
ECDSA key fingerprint is SHA256:xYrTfxcbw0whLwHdTeoMUD3KXprTQ75adcbpkBe2Z5s.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes

Warning: Permanently added 'jh-k8s-worker-1,188.185.29.70' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-88-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Tue Nov  1 09:45:44 UTC 2022

  System load:           0.0
  Usage of /:            3.3% of 38.60GB
  Memory usage:          3%
  Swap usage:            0%
  Processes:             138
  Users logged in:       0
  IPv4 address for ens3: 188.185.29.70
  IPv6 address for ens3: 2001:1458:d00:68::100:261

To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

ubuntu@jh-k8s-worker-1:~$ uptime
 09:46:30 up 13 min,  1 user,  load average: 0.00, 0.00, 0.00
```
