---
title: "Linux Router With Containers"
date: 2021-09-04T19:52:08-03:00
draft: false
tags: ["docker","network","linux"]
---
One of my passions in technology is networking, another passion is open source and linux. So sometimes I come across networking software or just to play around with linux standard networking tools like iptables, tc or iproute2, or even play with open source implementations of routing protocols using Quagga/Bird/FRR/go-bgp.  
I always found hard when testing these tools to get a proper setup on my laptop. Of course there are some tools that already help us in this regard like GNS3. However, using GNS3 I need to provide virtual machine images and configure all of them to save specific topologies. I wanted something that I can use quick and simple without the overhead of managing virtual machines and their configurations.  
First thing I though of was set up vagrant linux vms with extra network interfaces and use ansible to configure the virtualized linux routers.  
Then I thought, why not do it in docker?

# Docker and a linux router? Interesting!

So first I wanted to find how to do the link networking of container to create topologies that will help me test the scenarios I want.  
When using docker standard tools it is possible to attach a container to multiple networks, you can use `docker network create` to create new networks and then run the container with the `--net` flag to specify the network you want. To get mutliple networks attached to a docker container (I tested with networks in bridge mode) you can attach secondary networks to a container with `docker network connect [OPTIONS] NETWORK CONTAINER` after the container was created.  
But this last approach still made makes some noise in my head where the containers were linked through a bridge on the host network namespace and containers by default get configs like default GW and all of that. That reason made me think of a different strategy to actually manipulate the network namespaces myself to actually have the topology I really want. Then posibilities of doing complex topologies opens up.  
Just starting a container with `docker run --net none ......` starts a container with an isolated network namespace and only a loopback interface, then I have a blank page to attach interfaces and link them with other containers without the need to use bridges in the host networking.  
A last thing which is pretty good is that, we can take this approach with any container image so each container can be started with an official FRR, go-bgp or bird container image to do more complex cases and routing protocols with very lightweight containers running on the host. And also very easy to clean up the environment after we are finished with the tests. This setup makes it pretty easy to recreate a topology saving extremely precious time to actually do networking research rathen than spending time creaing a topology again.  
I will explain how to do it with a simple example with just ubuntu image containers and a simple topology and static routes as a proof of concept, however we could technically add as many interfaces and connections to other containers as we want.



# Topology
The example I will be showing has the following topology, a simple router in between 2 containers with static routes.
```
 -----------------------
|    (container C1)     |
| eth0 192.168.10.2/24] |
 -----------------------
           ^
           |
           v
 ---------------------- 
| eth0 192.168.10.1/24 |
|   (container R1)     |
| eth1 192.168.11.1/24 |
 ---------------------- 
           ^
           |
           v
 -----------------------
| eth0 192.168.11.2/24] |
|    (container C2)     |
 -----------------------
```

In this example there are some limitations and requiremens. We need root access to the linux host to manipulate the kernel network namespaces. So this is a limitation if you are on a Mac using Docker for Mac because we don't have access to the VM running docker, if you are using a Mac you could still start a linux vm and install docker on it.  

# Setup

- Ubuntu 20.04 LTS for ARM
- Running on Raspberry Pi 4 with 8G RAM (however with a 1G RAM would be more than enough)
- Docker CE 20.10.8

I tested this on a raspberry pi to just show it can be run in very resource constrained setup. Of course if running more complex software like FRR or BIRD might have more memory needs.

# Starting the containers

In this setup we will end with 3 containers running 2 clients and 1 router. Also we will use a custom build container image based on ubuntu 20.04 I build which have multiple tools needed for network troubleshooting and this kind of tests. Here is the [github repo link](https://github.com/fcastello/ubuntu-nework) and the image [dockerhub link](https://hub.docker.com/r/fcastello/ubuntu-network)

```bash
docker run --privileged -d -t --net none --name c1 fcastello/ubuntu-network bash
docker run --privileged -d -t --net none --name r1 fcastello/ubuntu-network bash
docker run --privileged -d -t --net none --name c2 fcastello/ubuntu-network bash
```
Containers:
- c1, client 1 container
- r1, router 1 container
- c2, client 2 container

As we started the containers with `--net none` the containers have an isolated network namespace with no link to anything just yet. One thing to have in mind I started the containers with `--privileged` flag so if we actually want to `ip route` or `ip addr` command inside the container then it needs privileges to manipulate the kernel's routing tables. However in this example the flag is not needed as we will be manipulating the namespaces from the host and not the container. If you don't want to run as privileged, then you can add only the networking capabilities to the container you need rather than giving it all capabilities with privileged mode.

```bash
root@rpi:~$ docker ps
CONTAINER ID   IMAGE                        COMMAND                  CREATED        STATUS        PORTS     NAMES
b083015e0797   fcastello/ubuntu-network     "bash"                   1 minute ago   Up 1 minute             c2
831fbfe63ac1   fcastello/ubuntu-network     "bash"                   1 minute ago   Up 1 minute             r1
199ffcf1ea10   fcastello/ubuntu-network     "bash"                   1 minute ago   Up 1 minute             c1
```


# Find the container network namespace for our containers
All commands done in the next steps should ideally be run in the same bash session as we are using variables to reuse values in later commands.  


```bash

# Get the container id for each container (will be needed later)
c1_id=$(docker ps --format '{{.ID}}' --filter name=c1)
r1_id=$(docker ps --format '{{.ID}}' --filter name=r1)
c2_id=$(docker ps --format '{{.ID}}' --filter name=c2)


# Get the containers pids which will be used to find their network namespace
c1_pid=$(docker inspect -f '{{.State.Pid}}' ${c1_id})
r1_pid=$(docker inspect -f '{{.State.Pid}}' ${r1_id})
c2_pid=$(docker inspect -f '{{.State.Pid}}' ${c2_id})

# create the /var/run/netns/ path if it doesn't already exist
mkdir -p /var/run/netns/


# Create a soft link to the containers network namespace to /var/run/netns/
ln -sfT /proc/$c1_pid/ns/net /var/run/netns/$c1_id
ln -sfT /proc/$r1_pid/ns/net /var/run/netns/$r1_id
ln -sfT /proc/$c2_pid/ns/net /var/run/netns/$c2_id


# Now lets show the ip addresses in each contaier namespace
# C1
root@rpi:~$ ip netns exec $c1_id ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
# R1
root@rpi:~$ ip netns exec $r1_id ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
# C2
root@rpi:~$ ip netns exec $c2_id ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
```
We have now set all links to have access to the containers namespaces already. Lets keep going.

# Create the virtual ethernet interfaces and assign them to the containers
Some of this steps need to be done from the host which has access to cgroups and network namespaces for all containers.  

```bash
# Create the virtual ethernet devices for conecting C1 to R1
ip link add 'c1-eth0' type veth peer name 'r1-eth0'

# Create the virtual ethernet devices for conecting C2 to R1
ip link add 'c2-eth0' type veth peer name 'r1-eth1'

# We created the virtual ethernet pairs but they are still in the host network namespace
# we need to move each virtual interface now to the corresponding containers namespace

# move c1 interface to c1 container
ip link set 'c1-eth0' netns $c1_id

# move r1 interfaces to r1 container
# note that r1 is a router which will
# need at least 2 interfaces
ip link set 'r1-eth0' netns $r1_id
ip link set 'r1-eth1' netns $r1_id

# move c2 interface to c2 container
ip link set 'c2-eth0' netns $c2_id

# Next step is not needed but it is nice to have more standard interface names
# so we will rename interfaces inside the containers

# rename c1 container interface from c1-eth0 to eth0
ip netns exec $c1_id ip link set 'c1-eth0' name 'eth0'

# rename r1 container interfaces from r1-eth0 to eth0 and r1-eth1 to eth1
ip netns exec $r1_id ip link set 'r1-eth0' name 'eth0'
ip netns exec $r1_id ip link set 'r1-eth1' name 'eth1'

# rename c2 container interface form c2-eth0 to eth0
ip netns exec $c2_id ip link set 'c2-eth0' name 'eth0'


# bring up all interfaces in containers
ip netns exec $c1_id ip link set 'eth0' up
ip netns exec $c1_id ip link set 'lo' up
ip netns exec $r1_id ip link set 'eth0' up
ip netns exec $r1_id ip link set 'eth1' up
ip netns exec $r1_id ip link set 'lo' up
ip netns exec $c2_id ip link set 'eth0' up
ip netns exec $c2_id ip link set 'lo' up
```
All containers now have interfaces set in them and they are linked toguether directly with no bridge in between

# Setting ip and routes on the containers

Now we need to set ips and routes to be abel to exchange traffic between containers


```bash

# lets set c1 container ip to 192.168.10.2
ip netns exec $c1_id ip addr add 192.168.10.2/24 dev eth0

# lets set r1 ips to 192.168.10.1 and 192.168.11.1
ip netns exec $r1_id ip addr add 192.168.10.1/24 dev eth0
ip netns exec $r1_id ip addr add 192.168.11.1/24 dev eth1

# lets set c2 container ip to 192.168.11.2
ip netns exec $c2_id ip addr add 192.168.11.2/24 dev eth0


# now our containers have ips set.
# Lets do some testing.

# ping r1 from c1
root@rpi:~$ docker exec -it c1 ping -c 4 192.168.10.1
PING 192.168.10.1 (192.168.10.1): 56 data bytes
64 bytes from 192.168.10.1: icmp_seq=0 ttl=64 time=0.209 ms
64 bytes from 192.168.10.1: icmp_seq=1 ttl=64 time=0.236 ms
64 bytes from 192.168.10.1: icmp_seq=2 ttl=64 time=0.222 ms
64 bytes from 192.168.10.1: icmp_seq=3 ttl=64 time=0.258 ms
--- 192.168.10.1 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.209/0.231/0.258/0.000 ms

# now ping r1 from c2
root@rpi:~$ docker exec -it c2 ping -c 4 192.168.11.1
PING 192.168.11.1 (192.168.11.1): 56 data bytes
64 bytes from 192.168.11.1: icmp_seq=0 ttl=64 time=0.200 ms
64 bytes from 192.168.11.1: icmp_seq=1 ttl=64 time=0.234 ms
64 bytes from 192.168.11.1: icmp_seq=2 ttl=64 time=0.212 ms
64 bytes from 192.168.11.1: icmp_seq=3 ttl=64 time=0.221 ms
--- 192.168.11.1 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.200/0.217/0.234/0.000 ms


# From each c1 and c2 container we can ping the router ips that are in the same subnet,
# but lets try to ping form c1 to c2
root@rpi:~$ docker exec -it c1 ping -c 4 192.168.11.2
PING 192.168.11.2 (192.168.11.2): 56 data bytes
ping: sending packet: Network is unreachable

# Not posible to ping from c1 to c2 because they are in different
# subnets and not directly connected.

# Let's add default routes to c1 and c2 containers that will go through r1

# set default gw on c1 container
ip netns exec $c1_id ip route add default via 192.168.10.1 dev eth0

# set default gw on c2 container
ip netns exec $c2_id ip route add default via 192.168.11.1 dev eth0


# Now if we ping from c1 to c2
root@rpi:~$ docker exec -it c1 ping -c 4 192.168.11.2
PING 192.168.11.2 (192.168.11.2): 56 data bytes
64 bytes from 192.168.11.2: icmp_seq=0 ttl=63 time=0.226 ms
64 bytes from 192.168.11.2: icmp_seq=1 ttl=63 time=0.262 ms
64 bytes from 192.168.11.2: icmp_seq=2 ttl=63 time=0.267 ms
64 bytes from 192.168.11.2: icmp_seq=3 ttl=63 time=0.235 ms
--- 192.168.11.2 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.226/0.247/0.267/0.000 ms
```

Now we just finished creating a linux router between 2 containers. By creating a similar script we can create more complex topologies where we can do more advanced testing. Now let's clean it up.

# Clean up
For cleaning up the environment we just need to stop and remove the container and delete the namespace links

```bash
# Delete the containets
docker stop c1 c2 r1 && docker rm c1 c2 r1

# Clean up the Network namespaces links
rm /var/run/netns/$c1_id
rm /var/run/netns/$r1_id
rm /var/run/netns/$c2_id
```



# Conclusion

I really enjoyed writing this blog post, I wanted to do this for a while but never found the time. Now that I know this is possible there might be more related posts in the future. Now I can create more complex topologies and mix it up a bit using routing protocols. I can think of plenty of use cases for this blog post from testing routing protocols to playing or testing iptables rules or playing with tc rules for traffic control between containers.  
Best of all this is pretty lightweight and easy to clean up leaving my laptop (or the raspberry pi) very clean after I finish up.

