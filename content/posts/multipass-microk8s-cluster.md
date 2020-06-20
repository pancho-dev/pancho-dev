---
title: "Multipass Microk8s Cluster on multiple nodes"
date: 2020-06-13T11:37:56-03:00
draft: true
---

A couple of posts ago I talked about multinode kubernetes clusters and the benefis of them running them for developing automations and testing software. I still bring up the question is that most developers probably don't need this setup. My motivation is because I work on cloud infrastructure and automation od delployments, databases, and lots of complex scenarios that running single node k8s cluster doesn't fit my needs. This time I will try another approach. I have used [kind](https://kind.sigs.k8s.io/docs/user/quick-start/), [ignite](https://github.com/weaveworks/ignite) micro vms and [k3s](https://rancher.com/docs/k3s/latest/en/quick-start/) in poast posts. This time is the turn of some tools from Cannonical which seemt o work together pretty well. I will create a multinode cluster running vms created with [multipass](https://multipass.run/) and [microk8s](https://microk8s.io/).

# Motivation
My motivations still lies that I have different use cases and I want to test the tools that suit my needs better. In this case I will give some reference why this time I chose multipass and microk8s. Other combinations like kind has the restriction that it's not possible to access the network directly when running on Mac and Windows for example, I could use vagrant and virtualbox to have a setup that can be reproduced on Linux, Mac and Windows, however I found that setup is the one that has the most features and felxibility, but vms are heavier to run in a laptop when combining it with k8s the laptop will be overloaded pretty quickly. Then I took the aproach os using firecracker micro vms with ignite and combined it with k3s, that worked very well but it is restricted to run only in linux with kvm support, so Mac and Windows are out of that one.  
I usually develop on a Mac and on a regular basis I have a home server with linux, so I want to be able to reproduce some things in both, then my motivation for portability and also in some cases I want to be able to run several nodes in the k8s cluster or run multiple clusters to try things like federation or multicluster service mesh, or some other wried scenario.  
Given my motivations, I am in search for a technology that allows me to run multple vms that are as lightweight as possible (to be able to run everything in a single machine of laptop) with the resemblance of a cloud native setup and networking. So I need a felxyble networking scenario, good isolation given by vms, and use the least resources possible. I will go throuhg this post explaining how to setup the cluster using multipass vms and get some conclusions at the end.

# Create the vms
I wanted to try multiplass for a few reasons. First seems very simple and fast to create vms; the network stack integrates seemsless with the OS in linux, Mac and Windows using bridges; and vms run as natively as possible, on linux uses kvm , on Mac uses Hypervisor.framework / hyperkit and on Windows uses Hyper-V. It can also use virtualbox for the vms but I wont go into that detail as if choosing vitualbox I would choose vagrant to manage it as it is very flexible.  
To install multipass just follow the official installation for your OS of preferences [here](https://multipass.run/docs).  
Now lets create 3 vms that wil later form our k8s cluster.
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
Creating the 2 vms is very easy and fast also generates a loca bridge and the vms IP are accessible to form the laptop making the networking setup very handy.


# Install microk8s
This time I chose microk8s because it integrates seemless with muultipass and I wanted to experiment with a different kubernetes disstribution.  

To create the master we will install microk8s and get some basic configuration and modules up and running.
```bash
# log in to the master
$ multipass shell master

# Install the microk8s snap
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
Now we have created the master we need to join teh worker nodes to the cluster. On microk8s to join a node to the cluster you need to issue a command on the master for each node. Copy the join command and repeat in each worker node the same procedure.

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

Joint the nodes to the cluster (remember you need to repear the procedure for each node)
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
# log in to the master
$ multipass shell master

# Sow the nodes
$ kubectl get nodes
NAME           STATUS   ROLES    AGE     VERSION
192.168.64.5   Ready    <none>   4m23s   v1.18.3
192.168.64.6   Ready    <none>   114s    v1.18.3
master         Ready    <none>   24m     v1.18.3
```

Now we are ready to start deploying pods and creating services.

# Run some basic workload

```bash
# log in to the master, or wherever you configured kubectl.
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

# Now lets spice up a bit the deployment we just did
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
See how easy was to create the cluster and deploy a basic service and access it. This probably takes abot 20 minutes and you get a full flodged k8s cluster with multiple nodes to do you development.


# Conclusions

I really enjoyed experimenting with multipass and microk8s. Both tools seem pretty easy to work with and they integrate smoothly. The part I like the most that multipass intgrates pretty well with all many OS and using their native virtualization framworks. That makes it incredibly lightweight to run the vms. Other feature I liked is that creates a local bridge and you have full unrestricted access to those IPs from your laptop, I tested this in Mac and Linux and the experience for the networking was almost the same, which is a huge plus. And also multipass and microk8s both solve a lot of boilerplate configurations which just by running a couple of commands you get a full fledged k8s cluster. I liked the approach of the plugins and how easy is to enable and disable them in microk8s giving you a very clean k8s cluster if you want to do things yourself it doesn't come bloated with stuff installed in it, which is a huge plus for microk8s. Also microk8s seem very lightweight similar to k3s and both ahve very similar features including running an high availability version which makes them both suitable for edge computing production use cases.  
The disadvantages I found with microk8s is that the pluggin system seems a bit canned and hard to know what they are doing, but still if you want to do things yourself you can refrain from installing the plugins that install things yourself.  ABout multipass disadvantages, I found only one restriction (Huge one) that multipass runs only ubuntu vms and it all revolves around ubuntu. So if you want ot play with other linux version this is a major hurdle.  
In general terms it's one of the setup that fit most of my needs while keeping it simple to create and adapts to be able to run in a lightweight way in multiple environments and OS. Also we could combine microk8s and multipass with other technologies. We coul use microk8s with iginite vms or virtualbox vms, or run k3s with multipass vms, the beauty of this tools is that you can combine them into whatever makes the most sense for your use case and helps speed up development.
