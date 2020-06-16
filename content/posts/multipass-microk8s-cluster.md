---
title: "Multipass Microk8s Cluster"
date: 2020-06-13T11:37:56-03:00
draft: true
---

INTRO TODO


# Create the vms

```bash
# create master
$ multipass launch --name master -m 2G
Launched: master

# Create workers
$ multipass launch --name worker1 -m 2G
Launched: worker1

$ multipass launch --name worker2 -m 2G
Launched: worker2

# Show the newly created vms
$ multipass ls
Name                    State             IPv4             Image
master                  Running           192.168.64.4     Ubuntu 18.04 LTS
worker1                 Running           192.168.64.5     Ubuntu 18.04 LTS
worker2                 Running           192.168.64.6     Ubuntu 18.04 LTS
```


# Install microk8s
```bash
$ multipass shell master

$ sudo snap install microk8s --classic --channel=1.18/stable
2020-06-13T12:03:41-03:00 INFO Waiting for restart...
microk8s (1.18/stable) v1.18.3 from Canonical✓ installed

$ sudo usermod -a -G microk8s $USER
$ sudo chown -f -R $USER ~/.kube
$ microk8s status --wait-ready
microk8s is running
addons:
cilium: disabled
dashboard: disabled
dns: disabled
fluentd: disabled
gpu: disabled
helm: disabled
helm3: disabled
ingress: disabled
istio: disabled
jaeger: disabled
knative: disabled
kubeflow: disabled
linkerd: disabled
metallb: disabled
metrics-server: disabled
prometheus: disabled
rbac: disabled
registry: disabled
storage: disabled

# to get alias for kubectl
$ echo "alias kubectl='microk8s kubectl'" >> ~/.bash_aliases
$ sudo usermod -a -G microk8s ubuntu
$ sudo chown -f -R ubuntu ~/.kube


# Lets add some basic things to the cluster
$ microk8s enable dns storage
Enabling DNS
Applying manifest
serviceaccount/coredns created
configmap/coredns created
deployment.apps/coredns created
service/kube-dns created
clusterrole.rbac.authorization.k8s.io/coredns created
clusterrolebinding.rbac.authorization.k8s.io/coredns created
Restarting kubelet
DNS is enabled
Enabling default storage class
deployment.apps/hostpath-provisioner created
storageclass.storage.k8s.io/microk8s-hostpath created
serviceaccount/microk8s-hostpath created
clusterrole.rbac.authorization.k8s.io/microk8s-hostpath created
clusterrolebinding.rbac.authorization.k8s.io/microk8s-hostpath created
Storage will be available soon
```


```bash
# Get the join command 
# NOTE, you need to issue this command for every node you are joining
# as the tokes is only valid for one time
$ sudo microk8s add-node
Join node with: microk8s join 192.168.64.4:25000/IfrgUOBCMGxZyAcRgEXXLONcwMKWpstO

If the node you are adding is not reachable through the default interface you can use one of the following:
 microk8s join 192.168.64.4:25000/IfrgUOBCMGxZyAcRgEXXLONcwMKWpstO
 microk8s join 10.1.15.0:25000/IfrgUOBCMGxZyAcRgEXXLONcwMKWpstO
```




Joint the nodes to the cluster
```bash
# Get A Shell to worker1 and then repeat the same for worker2
$ multipass shell worker1

# install microk8s in worker node
$ sudo snap install microk8s --classic --channel=1.18/stable

# Now join the node to the master
$ sudo microk8s join 192.168.64.4:25000/IfrgUOBCMGxZyAcRgEXXLONcwMKWpstO
```

After running joining in worker1 and worker2 we can check in the master
```bash
$ multipass shell master

$ kubectl get nodes
NAME           STATUS   ROLES    AGE     VERSION
192.168.64.5   Ready    <none>   4m23s   v1.18.3
192.168.64.6   Ready    <none>   114s    v1.18.3
master         Ready    <none>   24m     v1.18.3
```

Now we are ready to start deploying things


# Run some basic workload

```bash
$ kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4
deployment.apps/hello-node created

# Show the deployment
$ kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   1/1     1            1           28s

# Show the pods running
$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-7bf657c596-htx9p   1/1     Running   0          52s

# Now lets spice up a bit the the deployment we just did
$ kubectl scale deployments/hello-node --replicas=5
deployment.apps/hello-node scaled

# Show the pods running again with having it scaled up
# We will see 5 pods running now and they will be spanned
# accross all 3 nodes
$ kubectl get pods -o wide
NAME                          READY   STATUS    RESTARTS   AGE     IP          NODE           NOMINATED NODE   READINESS GATES
hello-node-7bf657c596-2vkgq   1/1     Running   0          61s     10.1.85.3   192.168.64.5   <none>           <none>
hello-node-7bf657c596-5x8kj   1/1     Running   0          61s     10.1.15.8   master         <none>           <none>
hello-node-7bf657c596-frr2j   1/1     Running   0          61s     10.1.85.2   192.168.64.5   <none>           <none>
hello-node-7bf657c596-htx9p   1/1     Running   2          2d18h   10.1.71.4   192.168.64.6   <none>           <none>
hello-node-7bf657c596-vthtr   1/1     Running   0          61s     10.1.71.5   192.168.64.6   <none>           <none>

# Enable ingress in the cluster
# this will create an nginx ingress
$ microk8s enable ingress
Enabling Ingress
namespace/ingress created
serviceaccount/nginx-ingress-microk8s-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-microk8s-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-microk8s-role created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-microk8s created
configmap/nginx-load-balancer-microk8s-conf created
daemonset.apps/nginx-ingress-microk8s-controller created
Ingress is enabled


# Create the service for the hello-node app
$ cat <<EOF | kubectl apply -f -
kind: Service
apiVersion: v1
metadata:
  name: hello-node
spec:
  selector:
    app: hello-node
  ports:
  # Default port used by the image
  - port: 8080
EOF
service/hello-node created


# setup the ingress controller to forward
# /hello path to helo-node service.
$ cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  rules:
  - http:
      paths:
      - path: /hello
        backend:
          serviceName: hello-node
          servicePort: 8080
EOF
ingress.extensions/my-ingress created

# get the new service through the ingress
# using the master IP
# This adddess is avaiable also from the laptop as
# is is bridged to an interface in the laptop
# ip's depend on ips that were assigned to the vms.
# you can get the ip's issuing `multipass ls`
$ curl http://192.168.64.4/hello
CLIENT VALUES:
client_address=10.1.21.1
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://192.168.64.4:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=192.168.64.4
user-agent=curl/7.58.0
x-forwarded-for=192.168.64.1
x-forwarded-host=192.168.64.4
x-forwarded-port=80
x-forwarded-proto=http
x-original-uri=/hello
x-real-ip=192.168.64.5
x-request-id=c128fb95cd6919745659310198f823c1
x-scheme=http
BODY:
-no body in request-

# If we try the worker ips it will also work
$ curl http://192.168.64.5/hello
CLIENT VALUES:
client_address=10.1.96.0
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://192.168.64.5:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=192.168.64.5
user-agent=curl/7.58.0
x-forwarded-for=192.168.64.1
x-forwarded-host=192.168.64.5
x-forwarded-port=80
x-forwarded-proto=http
x-original-uri=/hello
x-real-ip=192.168.64.5
x-request-id=883155681549409a4d28d10cf14fd033
x-scheme=http
BODY:
-no body in request-

$ curl http://192.168.64.6/hello
CLIENT VALUES:
client_address=10.1.93.0
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://192.168.64.6:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=192.168.64.6
user-agent=curl/7.54.0
x-forwarded-for=192.168.64.1
x-forwarded-host=192.168.64.6
x-forwarded-port=80
x-forwarded-proto=http
x-original-uri=/hello
x-real-ip=192.168.64.1
x-request-id=84e80e5017782584433bd1c364c5d9d2
x-scheme=http
BODY:
-no body in request-
```