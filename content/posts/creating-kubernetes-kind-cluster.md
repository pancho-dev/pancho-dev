---
title: "Creating Kubernetes Kind Cluster"
date: 2020-05-30T15:11:30-03:00
draft: false
---

In an earlier post I showed how to create [Multinode k8s cluster using ignite and k3s]({{< relref "multinode-k8s-ignite-k3s.md" >}}), while this was a good experience I needed to test some other tools, and this time I decided to go for kind (kubernetes in docker). Which looks like a good approach to work with clusters locally and it will still be lightweight as the kubernetes "nodes" will be actually running as docker containers. This at first glance looks like a easier approach and seems to work in a similar way in Mac, linux or wirndows, which gives a great advantage.  
Also this time I choose to work with kind because it allows to do a lot of stuff like simulating multiple control plane nodes and multiple workernodes. In my case I did wanted multiple nodes to try things like running postgres clusters and use anti affinity to develop in my laptop some automation around postgres clusters. It is very fast to start a cluster and play around with multiple clusters. Another cool feature is that is easy to start a cluster with multiple masters and the setup of the cluster seems a very "vanilla" k8s cluster which is and advantage to be running things similar to what you would run in production others like k3s or microk8s use a lot of lightweight components that might not actually look like "production clusters", however this is still a very personal opinion and depends on the needs of the developer and all have pros and cons. We will see how it looks like after I actually finish the post and get some conclusions.

# Installation

Installation of kind is pretty straightforward, the requirement is to have docker already installed in your laptop/server/vm and follow the [Official Kind Guide](https://kind.sigs.k8s.io/docs/user/quick-start/)

# Getting a cluster going

We will be creating a cluster in a similar way we created a cluster with other tools [in a past post]({{< relref "multinode-k8s-ignite-k3s.md" >}}) and at the end of the post I will do some basic comparison of what I experienced working with both.  

## Creating a cluster

We will start creating the most basic cluster...
```bash
# Just by issuing this command you get a fresh new cluster
$ kind create cluster
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.18.2) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

# Let's show the cluster nodes
$ kubectl get nodes -o wide
NAME                 STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
kind-control-plane   Ready    master   2m45s   v1.18.2   172.18.0.2    <none>        Ubuntu 19.10   4.19.76-linuxkit   containerd://1.3.3-14-g449e9269

# Let's check what docker is running there
$ docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                       NAMES
4a116756e882        kindest/node:v1.18.2   "/usr/local/bin/entr…"   3 minutes ago       Up 3 minutes        127.0.0.1:65236->6443/tcp   kind-control-plane

# now let's show docker netowrks, kind created it's own network 
# to be isolated for other containers that might be running in docker.
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
227c42913b8d        bridge              bridge              local
3bb54436f4f5        host                host                local
052d7b1dcf12        kind                bridge              local
41310fdd38ac        none                null                local

# Now show all pods that were created on the go
$ kubectl get pods --all-namespaces
NAMESPACE            NAME                                         READY   STATUS    RESTARTS   AGE
kube-system          coredns-66bff467f8-9zwn4                     1/1     Running   0          10m
kube-system          coredns-66bff467f8-jc24h                     1/1     Running   0          10m
kube-system          etcd-kind-control-plane                      1/1     Running   0          11m
kube-system          kindnet-mltj7                                1/1     Running   0          10m
kube-system          kube-apiserver-kind-control-plane            1/1     Running   0          11m
kube-system          kube-controller-manager-kind-control-plane   1/1     Running   0          11m
kube-system          kube-proxy-btv4q                             1/1     Running   0          10m
kube-system          kube-scheduler-kind-control-plane            1/1     Running   0          11m
local-path-storage   local-path-provisioner-bd4bb6b75-4w9cr       1/1     Running   0          10m


# now let's delete the cluster, as easy as that.
$ kind delete cluster
Deleting cluster "kind" ...
```
This a a very basic cluster setup, however I really wanted something more complex to work with. Follow the link if you want to see a full list of what can be tweaked in [kind configurations](https://kind.sigs.k8s.io/docs/user/configuration/)  
Let's see an example configuration with 1 control plane (master node) and 2 worker nodes.  
We need to create a config file lets name it `my-cluster.yml` with the following.
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
# One control plane node and two "workers".
nodes:
- role: control-plane
- role: worker
- role: worker
```
Then run
```bash
# Create the cluster named my-cluster
$ kind create cluster --name my-cluster --config my-cluster.yml
Creating cluster "my-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.18.2) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-my-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-my-cluster

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂

# Show the recently created cluster
$ kind get clusters
my-cluster

# Show the nodes
$ kubectl get nodes -o wide
NAME                       STATUS   ROLES    AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION     CONTAINER-RUNTIME
my-cluster-control-plane   Ready    master   3m      v1.18.2   172.18.0.4    <none>        Ubuntu 19.10   4.19.76-linuxkit   containerd://1.3.3-14-g449e9269
my-cluster-worker          Ready    <none>   2m27s   v1.18.2   172.18.0.2    <none>        Ubuntu 19.10   4.19.76-linuxkit   containerd://1.3.3-14-g449e9269
my-cluster-worker2         Ready    <none>   2m21s   v1.18.2   172.18.0.3    <none>        Ubuntu 19.10   4.19.76-linuxkit   containerd://1.3.3-14-g449e9269

# Show all pods
$ kubectl get pods --all-namespaces
NAMESPACE            NAME                                               READY   STATUS    RESTARTS   AGE
kube-system          coredns-66bff467f8-97kfh                           1/1     Running   0          5m45s
kube-system          coredns-66bff467f8-lxwl6                           1/1     Running   0          5m45s
kube-system          etcd-my-cluster-control-plane                      1/1     Running   0          6m1s
kube-system          kindnet-2db6f                                      1/1     Running   0          5m45s
kube-system          kindnet-p6c5f                                      1/1     Running   1          5m25s
kube-system          kindnet-wqht8                                      1/1     Running   0          5m31s
kube-system          kube-apiserver-my-cluster-control-plane            1/1     Running   0          6m1s
kube-system          kube-controller-manager-my-cluster-control-plane   1/1     Running   0          6m1s
kube-system          kube-proxy-98d9r                                   1/1     Running   0          5m25s
kube-system          kube-proxy-9xvn4                                   1/1     Running   0          5m45s
kube-system          kube-proxy-ph5z8                                   1/1     Running   0          5m31s
kube-system          kube-scheduler-my-cluster-control-plane            1/1     Running   0          6m
local-path-storage   local-path-provisioner-bd4bb6b75-2rdpn             1/1     Running   0          5m45s
```
With this setup we are running a 3 "nodes" k8s cluster so we are ready to start applying yaml files to deploy some workloads...  
But wait, if you are running on linux you will be able to access the k8s "nodes" with the ip docker assigned to the nodes, but when running in Mac or Windows there are some extra steps to take, as we all know docker desktop in mac or windows still runs inside a linux vm, so we need to take extra steps to get access to the workloads we will be setting up.  
Before proceeding we will have to delete our cluster and 
```bash
$ kind delete clusters my-cluster
Deleted clusters: ["my-cluster"]
```
We will need to setup ingress controller and some port mappings to the workloads we want to run, if we want to access the workloads from the machine. Here are the official docs how to configure an [ingres](https://kind.sigs.k8s.io/docs/user/ingress/) in kind.  

```bash
# To create the cluster again we will take a dirrefent approach.
# we will use a one shot command passing the yaml config from the cli, 
# this saves a step to prevent creating the file.
# we will add the ports we want to access. normally would be 80 
# and 443 but in this case I also added 8080
$ cat <<EOF | kind create cluster --name=my-cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
EOF
Creating cluster "my-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.18.2) 🖼
 ✓ Preparing nodes 📦 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-my-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-my-cluster

Have a nice day! 👋

# kind doesn't have an ingress controller set by default
# so we will need to setup one.
# In the following example we will setup an nginx ingress 
# proxy but we could usin any other ingresses 
# like contour, ambassador, traefik, etc

# lets check the nodes we have
$ kubectl get nodes
NAME                       STATUS   ROLES    AGE     VERSION
my-cluster-control-plane   Ready    master   6m23s   v1.18.2
my-cluster-worker          Ready    <none>   5m48s   v1.18.2
my-cluster-worker2         Ready    <none>   5m48s   v1.18.2

# Now lets apply the config to run an nginx ingress
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml
namespace/ingress-nginx created
serviceaccount/ingress-nginx created
configmap/ingress-nginx-controller created
clusterrole.rbac.authorization.k8s.io/ingress-nginx created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx created
role.rbac.authorization.k8s.io/ingress-nginx created
rolebinding.rbac.authorization.k8s.io/ingress-nginx created
service/ingress-nginx-controller-admission created
service/ingress-nginx-controller created
deployment.apps/ingress-nginx-controller created
validatingwebhookconfiguration.admissionregistration.k8s.io/ingress-nginx-admission created
clusterrole.rbac.authorization.k8s.io/ingress-nginx-admission created
clusterrolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
job.batch/ingress-nginx-admission-create created
job.batch/ingress-nginx-admission-patch created
role.rbac.authorization.k8s.io/ingress-nginx-admission created
rolebinding.rbac.authorization.k8s.io/ingress-nginx-admission created
serviceaccount/ingress-nginx-admission created


# show that pods are creating
$ kubectl get pods --all-namespaces
NAMESPACE            NAME                                               READY   STATUS              RESTARTS   AGE
ingress-nginx        ingress-nginx-admission-create-nqgq9               0/1     ContainerCreating   0          8s
ingress-nginx        ingress-nginx-admission-patch-p5cd8                0/1     ContainerCreating   0          8s
ingress-nginx        ingress-nginx-controller-cc8dd9868-rgx5b           0/1     ContainerCreating   0          18s
kube-system          coredns-66bff467f8-6gxmh                           1/1     Running             0          8m48s
kube-system          coredns-66bff467f8-fvlz6                           1/1     Running             0          8m48s
kube-system          etcd-my-cluster-control-plane                      1/1     Running             0          9m5s
kube-system          kindnet-b2qcw                                      1/1     Running             0          8m34s
kube-system          kindnet-bz87f                                      1/1     Running             2          8m34s
kube-system          kindnet-rpb8k                                      1/1     Running             0          8m48s
kube-system          kube-apiserver-my-cluster-control-plane            1/1     Running             0          9m5s
kube-system          kube-controller-manager-my-cluster-control-plane   1/1     Running             0          9m5s
kube-system          kube-proxy-5sc27                                   1/1     Running             0          8m34s
kube-system          kube-proxy-q24kf                                   1/1     Running             0          8m48s
kube-system          kube-proxy-tlhsk                                   1/1     Running             0          8m34s
kube-system          kube-scheduler-my-cluster-control-plane            1/1     Running             0          9m5s
local-path-storage   local-path-provisioner-bd4bb6b75-nqdgc             1/1     Running             0          8m48s

# And after it finishes we can see it here
$ kubectl get pods --all-namespaces
NAMESPACE            NAME                                               READY   STATUS      RESTARTS   AGE
ingress-nginx        ingress-nginx-admission-create-nqgq9               0/1     Completed   0          112s
ingress-nginx        ingress-nginx-admission-patch-p5cd8                0/1     Completed   1          112s
ingress-nginx        ingress-nginx-controller-cc8dd9868-rgx5b           1/1     Running     0          2m2s
kube-system          coredns-66bff467f8-6gxmh                           1/1     Running     0          10m
kube-system          coredns-66bff467f8-fvlz6                           1/1     Running     0          10m
kube-system          etcd-my-cluster-control-plane                      1/1     Running     0          10m
kube-system          kindnet-b2qcw                                      1/1     Running     0          10m
kube-system          kindnet-bz87f                                      1/1     Running     2          10m
kube-system          kindnet-rpb8k                                      1/1     Running     0          10m
kube-system          kube-apiserver-my-cluster-control-plane            1/1     Running     0          10m
kube-system          kube-controller-manager-my-cluster-control-plane   1/1     Running     0          10m
kube-system          kube-proxy-5sc27                                   1/1     Running     0          10m
kube-system          kube-proxy-q24kf                                   1/1     Running     0          10m
kube-system          kube-proxy-tlhsk                                   1/1     Running     0          10m
kube-system          kube-scheduler-my-cluster-control-plane            1/1     Running     0          10m
local-path-storage   local-path-provisioner-bd4bb6b75-nqdgc             1/1     Running     0          10m

# After finishing the setup lets show the 
# containers running in docker. 
# We can see in this case the my-cluster-control-plane 
# container has the ports 80, 443, and 8080 
# forwarded to allow traffic on the ingress. 
# That will allow use to use http://localhost 
# or https://localhost 
# there is a lmitation that we can only expose 
# those ports only in 1 container.
$ docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                                                                                         NAMES
a9b53a526c9e        kindest/node:v1.18.2   "/usr/local/bin/entr…"   13 minutes ago      Up 13 minutes                                                                                                     my-cluster-worker
dc1c539eb315        kindest/node:v1.18.2   "/usr/local/bin/entr…"   13 minutes ago      Up 13 minutes       0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 127.0.0.1:50462->6443/tcp   my-cluster-control-plane
9c92b0528a6a        kindest/node:v1.18.2   "/usr/local/bin/entr…"   13 minutes ago      Up 13 minutes                                                                                                     my-cluster-worker2
```
Now we have a ready to go cluster to play around running my workloads...  
In the next setion we will run a hello world app that will help

# Running a workload
We will run the workload the same way as a previous [post]({{< relref "multinode-k8s-ignite-k3s.md" >}}) so we can compare one another.  

```bash
# Run an echoserver to test the installation
$ kubectl create deployment hello-node --image=k8s.gcr.io/echoserver:1.4
deployment.apps/hello-node created

# Show the deployment
$ kubectl get deployments
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
hello-node   1/1     1            1           45s

# Show the pods running
$ kubectl get pods
NAME                          READY   STATUS    RESTARTS   AGE
hello-node-7bf657c596-g5jdd   1/1     Running   0          75s

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

# setup the ingress controller to forward
# /helo path to helo-node service.
$ cat <<EOF | kubectl apply -f -
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - http:
      paths:
      - path: /helo
        backend:
          serviceName: hello-node
          servicePort: 8080
EOF

# After that we can see the service in 
# our laptop/vm responding in localhost
$ curl http://localhost/helo
CLIENT VALUES:
client_address=10.244.0.2
command=GET
real path=/helo
query=nil
request_version=1.1
request_uri=http://localhost:8080/helo

SERVER VALUES:
server_version=nginx: 1.10.0 - lua: 10001

HEADERS RECEIVED:
accept=*/*
host=localhost
user-agent=curl/7.54.0
x-forwarded-for=172.18.0.1
x-forwarded-host=localhost
x-forwarded-port=80
x-forwarded-proto=http
x-real-ip=172.18.0.1
x-request-id=6b5eda14f6519007efa646317cc3f371
x-scheme=http
BODY:
-no body in request-
```

# Conclusions

I really enjoyed running kind on my laptop for this test. It's a great tool to run kubernetes clusters very fast in a local laptop, the official documentation is good.  
I like the fact that I could run multiple "nodes" in my cluster also there is some tunning that can be done to suit your needs, it is very easy to spin up clusters with multiple control planes, it is very lightweight as it is running docker containers (if you are running on windows or Mac, it's still running 1 vm).  
Takes care of all boilerplate configs specially kubectl config, and integrates in a seemless way with your exisiting kube config adding and removing contexts as you create clusters, this is great because in the past I had trouble with other tools screwing my kube config.  
But the feature I liked the most is speed, everything happens really fast and it is easy to teardown and spin it back up in a matter of minutes without a lot of effort, making it easy to start over when things go wrong.  
What I din't like, is that there are some limitations because of the fact that it's running in docker, specially if you are working with ingresses configurations or load balancers some special configurations or hacks need to be put in place to make them work which makes harder to reproduce staging or production environments when trying to automate deployments. It is limited in the networking side when tryning some CNI plugins like multus where you might need to have "nodes" with special configurations in order to get multiple networks working.  
However, it is still a great tool to develop and work with and still have a good environment to develop in a local environment. It is still considered alpha stage and it can't be deployed in production environments, it's main purpose is development of applications and kubernetes components.