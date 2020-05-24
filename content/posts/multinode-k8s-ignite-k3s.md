---
title: "Multinode k8s cluster using ignite and k3s"
date: 2020-05-17T18:22:05-03:00
draft: true
---
Sometimes if you are working with kubernetes, or developing applications that require a multinode setup to test some functionality running a multinode cluster is a must, in some cases you could use kind which you can spin up multinode/multimaster clusters, however there might be scenarios were you still need to test or develop functions that need the real feel of a cluster with multiple nodes.  
In the past I have run this in my local environment running vms with vagrant and virtualbox that worked very well, and I still use it for some special scenarios. But I needed something I could run more workers/masters on my local laptop one clear example I wanted to test the postgres operator from crunchydata (https://access.crunchydata.com/documentation/postgres-operator) that allows to create a postgres cluster with a stanby cluster in a different k8s cluster. Or try kilo (https://github.com/squat/kilo) to setup encrypted communications between pods and test the multidatacenter setup. Also if you want to play with CNI plugins or some advanced features this setup allows more felxibility to do so.  
My need could be solved by vagrant and virtualbox, and in fact is a great aproach and very easy to setup everything as code in an automated fashion. But I wanted to take my laptop to the next level with something more efficient and I found a great project that allows to run firecracker https://firecracker-microvm.github.io/ micro-vms in a way that is very similar to running docker containers. The projetc is called ignite (https://github.com/weaveworks/ignite) and you will have in no time vms running using very little resources which allows to run much more worker nodes and multiple clusters at the same time.  
We will install ignite in our ubuntu laptop/desktop/server and run a 3 node kubernetes cluster. The following have been tested in my ubuntu 18.04 server I run at home.

## Requirements

- adm64 linux machine (No mac support at the moment as firecracker relies on linux kvm). It should be technically possible to run on arm64 as there are binaries for firecracked and some distributions support kvm on arm platforms. I have tried to run on an ubuntu 18.04 and got ignite to work but I was not able to run a vm yet (probably need to build my own image an kernel with arm support)
- Ubuntu 18.04 (same steps should work in 20.04 or other ubuntu versions without many changes)
- console and root access to the host where we will install everything
- internet access to be able to pull images and software


# Install ignite
As mentioned before ignite is a software that can be used to start firecracker micro vms with the `look and feel` of docker, In fact there are a lot of similarities in the `ignite` command and the `docker` command. You can find everything https://github.com/weaveworks/ignite and  https://ignite.readthedocs.io/en/stable/  


```bash
# Become root
sudo -i

# Check that cpu has virtualization capabilities
$ lscpu | grep Virtualization
Virtualization:      VT-x

# Install Dependencies
apt-get update && apt-get install -y --no-install-recommends dmsetup openssh-client git binutils

which containerd || apt-get install -y --no-install-recommends containerd

# Install CNI plugins
export CNI_VERSION=v0.8.5
export ARCH=$([ $(uname -m) = "x86_64" ] && echo amd64 || echo arm64)
sudo mkdir -p /opt/cni/bin
curl -sSL https://github.com/containernetworking/plugins/releases/download/${CNI_VERSION}/cni-plugins-linux-${ARCH}-${CNI_VERSION}.tgz | sudo tar -xz -C /opt/cni/bin

# Install ignite binaries
export VERSION=v0.6.3
export GOARCH=$(go env GOARCH 2>/dev/null || echo "amd64")

for binary in ignite ignited; do
    echo "Installing ${binary}..."
    curl -sfLo ${binary} https://github.com/weaveworks/ignite/releases/download/${VERSION}/${binary}-${GOARCH}
    chmod +x ${binary}
    sudo mv ${binary} /usr/local/bin
done
```

Check your installation
```bash
$ ignite version
Ignite version: version.Info{Major:"0", Minor:"6", GitVersion:"v0.6.3", GitCommit:"ed51b9378f6e0982461074734beb145d571d56a6", GitTreeState:"clean", BuildDate:"2019-12-10T06:19:57Z", GoVersion:"go1.12.10", Compiler:"gc", Platform:"linux/amd64"}
Firecracker version: v0.18.1
Runtime: containerd
```
If there are any problems please refer to the ignite docs as it is a project under heavy development and there might be breaking changes in the current versions

# Create vms
Now that we have ignite installed and ready we can start creating vms for our kubernetes cluster. We will have 1 node named `master` and 2 worker nodes named `worker1` and `worker2`  
Master node
```bash
# Start the master node
$ sudo ignite run weaveworks/ignite-ubuntu \
  --name master \
  --cpus 1 \
  --memory 2GB \
  --size 6GB \
  --ssh
INFO[0001] Created VM with ID "a0679083cb8cb9d4" and name "master"
INFO[0001] Networking is handled by "cni"
INFO[0001] Started Firecracker VM "a0679083cb8cb9d4" in a container with ID "ignite-a0679083cb8cb9d4"

# log in to the master
$ sudo ignite ssh master

# On master vm change hostname
$ hostname master && \
echo 'master' > /etc/hostname && \
echo '127.0.0.1 master' >> /etc/hosts

# On master vm update the OS
$ apt update && \
apt upgrade -y && \
exit

```


Worker node 1
```bash
# Start worked node 1
$ sudo ignite run weaveworks/ignite-ubuntu \
  --name worker1 \
  --cpus 1 \
  --memory 2GB \
  --size 6GB \
  --ssh
INFO[0001] Created VM with ID "176870ae0df03d49" and name "worker1"
INFO[0001] Networking is handled by "cni"
INFO[0001] Started Firecracker VM "176870ae0df03d49" in a container with ID "ignite-176870ae0df03d49"

# log in to the worker node 1
$ sudo ignite ssh worker1

# On worker 1 vm change hostname
$ hostname worker1 && \
echo 'worker1' > /etc/hostname && \
echo '127.0.0.1 worker1' >> /etc/hosts

# On worker1 vm update the OS
$ apt update && \
apt upgrade -y && \
exit

```

Worker node 2
```bash
# Start worked node 1
$ sudo ignite run weaveworks/ignite-ubuntu \
  --name worker2 \
  --cpus 1 \
  --memory 2GB \
  --size 6GB \
  --ssh
INFO[0001] Created VM with ID "fa96691aabac2a3f" and name "worker2"
INFO[0001] Networking is handled by "cni"
INFO[0001] Started Firecracker VM "fa96691aabac2a3f" in a container with ID "ignite-fa96691aabac2a3f"

# log in to the worker node 1
$ sudo ignite ssh worker2

# On worker 1 vm change hostname
$ hostname worker2 && \
echo 'worker2' > /etc/hostname && \
echo '127.0.0.1 worker2' >> /etc/hosts

# On worker2 vm update the OS
$ apt update && \
apt upgrade -y && \
exit

```

# Install k3s
Now that we have 3 clean vms ready isntall a kubernetes cluster. For that we will use a lightweight kubernetes distribution called k3s (https://rancher.com/docs/k3s/latest/en/quick-start/)  
Before we start doing the installation we need to run the following to get IP informations of the vms
```bash
# we need to list the vms in order to get ip information needed for following steps
$ ignite ps
VM ID			IMAGE				KERNEL					SIZE	CPUS	MEMORY	CREATED		STATUS		IPS				PORTS	NAME
176870ae0df03d49	weaveworks/ignite-ubuntu:latest	weaveworks/ignite-kernel:4.19.47	6.0 GB	1	2.0 GB	8m26s ago	Up 8m26s	10.61.0.3, 127.0.0.1, ::1		worker1
a0679083cb8cb9d4	weaveworks/ignite-ubuntu:latest	weaveworks/ignite-kernel:4.19.47	6.0 GB	1	2.0 GB	14m ago		Up 14m		10.61.0.2, 127.0.0.1, ::1		master
fa96691aabac2a3f	weaveworks/ignite-ubuntu:latest	weaveworks/ignite-kernel:4.19.47	6.0 GB	1	2.0 GB	6m23s ago	Up 6m23s	10.61.0.4, 127.0.0.1, ::1		worker2
``` 

From this command we will ned the `master` node IP which in this case is `10.61.0.2`


## Install Master
In this tutorial we will run the quick install, if there is a need for a special setup or HA masters please check k3s full documentation for options.  
On the master run:
```bash
# log in to the master
$ sudo ignite ssh master

# Run Install script
$ curl -sfL https://get.k3s.io | sh -

[INFO]  Finding release for channel stable
[INFO]  Using v1.18.2+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.18.2+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.18.2+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s


# To get the kubectl config, NOTE: that you will need to change the 127.0.0.1 IP with the master ip is you want to run kubectl straight from you laptop. In this tutorial we will run all kubectl commands form the master so there is no need to copy this file anywhere
$ cat /etc/rancher/k3s/k3s.yaml

apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUJWekNCL3FBREFnRUNBZ0VBTUFvR0NDcUdTTTQ5QkFNQ01DTXhJVEFmQmdOVkJBTU1HR3N6Y3kxelpYSjIKWlhJdFkyRkFNVFU0T1RjME5EQXhNREFlRncweU1EQTFNVGN4T1RNek16QmFGdzB6TURBMU1UVXhPVE16TXpCYQpNQ014SVRBZkJnTlZCQU1NR0dzemN5MXpaWEoyWlhJdFkyRkFNVFU0T1RjME5EQXhNREJaTUJNR0J5cUdTTTQ5CkFnRUdDQ3FHU000OUF3RUhBMElBQlBLeDcrMmZzQjY2Ti9qaHNEbWRKbW9hSHJJZEhyNFlSNC9mdi9HOVo5czIKWlRxZkxGS3ZMVEdVR1kzWDR6WStnbk0xbGlTWmU3dTZEUDBKTkpzR1FOU2pJekFoTUE0R0ExVWREd0VCL3dRRQpBd0lDcERBUEJnTlZIUk1CQWY4RUJUQURBUUgvTUFvR0NDcUdTTTQ5QkFNQ0EwZ0FNRVVDSUZMODN4eFRPOVp4CnZiY056bk9tdW0zRlJrRDNiZG9rNDZjZkxZMGtPTG5FQWlFQXBLNzhESXlTZnVIZytZS09FcCs2M1E4WlhJbVUKSmlUaGdOMUhQeFRTZjRVPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://127.0.0.1:6443
  name: default
contexts:
- context:
    cluster: default
    user: default
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: default
  user:
    password: 1c8af2866442d22f6a3053ffd776ece9
    username: admin

# To get the cluster token needed to configure the worker nodes
$ cat /var/lib/rancher/k3s/server/node-token

K10a3037f19183b43823cb1394f466b489c988c2110b122728821d33eaa4c9f144a::server:28f6e3c31e97e08c0a5ecb500398f931

```

## Worker nodes
On Worker nodes issue de following commands
```bash
# Log in to worker1
$ sudo ignite ssh worker1

# Install k3s
$ curl -sfL https://get.k3s.io | K3S_URL=https://10.61.0.2:6443 K3S_TOKEN=K10a3037f19183b43823cb1394f466b489c988c2110b122728821d33eaa4c9f144a::server:28f6e3c31e97e08c0a5ecb500398f931 sh -

[INFO]  Finding release for channel stable
[INFO]  Using v1.18.2+k3s1 as release
[INFO]  Downloading hash https://github.com/rancher/k3s/releases/download/v1.18.2+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/rancher/k3s/releases/download/v1.18.2+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-agent-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s-agent.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s-agent.service
[INFO]  systemd: Enabling k3s-agent unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s-agent.service → /etc/systemd/system/k3s-agent.service.
[INFO]  systemd: Starting k3s-agent

$ exit

# Now log to worker2
$ sudo ignite ssh worker2

# Install k3s
$ curl -sfL https://get.k3s.io | K3S_URL=https://10.61.0.2:6443 K3S_TOKEN=K10a3037f19183b43823cb1394f466b489c988c2110b122728821d33eaa4c9f144a::server:28f6e3c31e97e08c0a5ecb500398f931 sh -


```

## Check the installation

```bash
# On the master node run
$ kubectl get nodes -o wide

NAME      STATUS   ROLES    AGE     VERSION        INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
master    Ready    master   10m     v1.18.2+k3s1   10.61.0.2     <none>        Ubuntu 18.04.4 LTS   4.19.47          containerd://1.3.3-k3s2
worker1   Ready    <none>   3m20s   v1.18.2+k3s1   10.61.0.3     <none>        Ubuntu 18.04.4 LTS   4.19.47          containerd://1.3.3-k3s2
worker2   Ready    <none>   17s     v1.18.2+k3s1   10.61.0.4     <none>        Ubuntu 18.04.4 LTS   4.19.47          containerd://1.3.3-k3s2

# Check pods running in the fhresly installed cluster
$ kubectl get pods --all-namespaces

NAMESPACE     NAME                                     READY   STATUS      RESTARTS   AGE
kube-system   local-path-provisioner-6d59f47c7-mhfjf   1/1     Running     0          14m
kube-system   metrics-server-7566d596c8-gfbj5          1/1     Running     0          14m
kube-system   helm-install-traefik-wkmjw               0/1     Completed   0          14m
kube-system   coredns-8655855d6-588ln                  1/1     Running     0          14m
kube-system   svclb-traefik-z94lc                      2/2     Running     0          13m
kube-system   traefik-758cd5fc85-tcgl4                 1/1     Running     0          13m
kube-system   svclb-traefik-smfch                      2/2     Running     0          7m16s
kube-system   svclb-traefik-4c8cx                      2/2     Running     0          4m12s

# Check services running
$ kubectl get services --all-namespaces

NAMESPACE     NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
default       kubernetes           ClusterIP      10.43.0.1       <none>        443/TCP                      15m
kube-system   kube-dns             ClusterIP      10.43.0.10      <none>        53/UDP,53/TCP,9153/TCP       15m
kube-system   metrics-server       ClusterIP      10.43.116.88    <none>        443/TCP                      15m
kube-system   traefik-prometheus   ClusterIP      10.43.228.116   <none>        9100/TCP                     15m
kube-system   traefik              LoadBalancer   10.43.143.31    10.61.0.2     80:32686/TCP,443:31374/TCP   15m

```
As we can see out of the box we have services and pods running already. In fact, k3s basic installation installed traefik as ingress controller and load balancers on by default on port 80 and 443, which means we already have an ingress proxy running in all the nodes in the cluster running in those ports. And any service we create as a lod balancer on a specific port will be available in all nodes.


## Run some basic workload

```bash
# Run an echoserver to test the installation
$ kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4

deployment.apps/hello-node created

# Show the deployment
$ kubectl get deployments

NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   1/1     1            1           29s

# Show the pods running
$ kubectl get pods

NAME                          READY   STATUS    RESTARTS   AGE
hello-node-7bf657c596-cfp6q   1/1     Running   0          3m10s

# WE can expose the service as load balanced in port 8080 with the ingress coontroller
$ kubectl expose deployment hello-node --type=LoadBalancer --port=8080

service/hello-node exposed


# Show services running as we can see the service load balaced with external IP 10.61.0.4 which is one of the nodes
$ kubectl get services

NAME         TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
kubernetes   ClusterIP      10.43.0.1      <none>        443/TCP          25m
hello-node   LoadBalancer   10.43.144.49   10.61.0.4     8080:31371/TCP   41s

# If we show pods again we can see after exposing it through the ingress controller we have 3 pods which are the proxies set by the load balancer which got exposed in all node on port 8080
$ kubectl get pods

NAME                          READY   STATUS    RESTARTS   AGE
hello-node-7bf657c596-cfp6q   1/1     Running   0          8m8s
svclb-hello-node-n4knw        1/1     Running   0          4m34s
svclb-hello-node-7sbnw        1/1     Running   0          4m34s
svclb-hello-node-w2tnp        1/1     Running   0          4m34s


# Now from the host running the vms or any of the worker nodes we can hit our ingress controller
$ curl http://10.61.0.4:8080
CLIENT VALUES:
client_address=10.42.0.0
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://10.61.0.4:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=10.61.0.4:8080
user-agent=curl/7.58.0
BODY:
-no body in request-

# but since the ingress contrioller is running in all nodes we can try all other node ips
$ curl http://10.61.0.3:8080
CLIENT VALUES:
client_address=10.42.1.3
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://10.61.0.3:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=10.61.0.3:8080
user-agent=curl/7.58.0
BODY:
-no body in request-

$ curl http://10.61.0.2:8080
CLIENT VALUES:
client_address=10.42.0.8
command=GET
real path=/
query=nil
request_version=1.1
request_uri=http://10.61.0.2:8080/

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=10.61.0.2:8080
user-agent=curl/7.58.0
BODY:
-no body in request-

```