# Smoke Test

<https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/79a3f79b27bd28f82f071bb877a266c2e62ee506/docs/13-smoke-test.md>

## Data Encryption

```sh
kubectl create secret generic kubernetes-the-hard-way \
  --from-literal="mykey=mydata"

ssh ubuntu@jh-k8s-cp-1 sudo ETCDCTL_API=3 etcdctl get \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem\
  /registry/secrets/default/kubernetes-the-hard-way | hexdump -C
```

## Deployments

```sh
$ kubectl create deployment nginx --image=nginx
deployment.apps/nginx created
$ kubectl get pods
NAME                    READY   STATUS    RESTARTS   AGE
nginx-76d6c9b8c-qtghs   1/1     Running   0          8s
```

## Port Forwarding

```sh
$ kubectl port-forward nginx-76d6c9b8c-qtghs 8080:80 &
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
$ curl -I localhost:8080
HTTP/1.1 200 OK
Server: nginx/1.23.2
Date: Tue, 01 Nov 2022 13:35:36 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 19 Oct 2022 07:56:21 GMT
Connection: keep-alive
ETag: "634fada5-267"
Accept-Ranges: bytes
```

## Logs

```sh
$ kubectl logs nginx-76d6c9b8c-qtghs
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2022/11/01 13:34:15 [notice] 1#1: using the "epoll" event method
2022/11/01 13:34:15 [notice] 1#1: nginx/1.23.2
2022/11/01 13:34:15 [notice] 1#1: built by gcc 10.2.1 20210110 (Debian 10.2.1-6)
2022/11/01 13:34:15 [notice] 1#1: OS: Linux 5.4.0-88-generic
2022/11/01 13:34:15 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 1048576:1048576
2022/11/01 13:34:15 [notice] 1#1: start worker processes
2022/11/01 13:34:15 [notice] 1#1: start worker process 28
2022/11/01 13:34:15 [notice] 1#1: start worker process 29
2022/11/01 13:34:15 [notice] 1#1: start worker process 30
2022/11/01 13:34:15 [notice] 1#1: start worker process 31
127.0.0.1 - - [01/Nov/2022:13:35:31 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.61.1" "-"
127.0.0.1 - - [01/Nov/2022:13:35:36 +0000] "HEAD / HTTP/1.1" 200 0 "-" "curl/7.61.1" "-"
```

## Exec

```sh
$ kubectl exec -it nginx-76d6c9b8c-qtghs -- nginx -v
nginx version: nginx/1.23.2
```

## Services

```sh
$ kubectl expose deploy/nginx --port 80 --type NodePort
service/nginx exposed
$ kubectl get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.32.0.1    <none>        443/TCP        109m
nginx        NodePort    10.32.0.37   <none>        80:30873/TCP   5s
$ curl -I jh-k8s-worker-1:30873
HTTP/1.1 200 OK
Server: nginx/1.23.2
Date: Tue, 01 Nov 2022 13:38:27 GMT
Content-Type: text/html
Content-Length: 615
Last-Modified: Wed, 19 Oct 2022 07:56:21 GMT
Connection: keep-alive
ETag: "634fada5-267"
Accept-Ranges: bytes
```
