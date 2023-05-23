---
title: "Highly Available Home Automation with Kubernetes"
date: 2023-03-27T15:47:17-03:00
draft: true
---

As my home page states I am a big fan of home automation and DIY electronic projects. Since I come from the automation and SRE world I like things to run always. My home automation started with a few smart bulbs, then Alexa integration for voice, then got more light bulbs form a different brand, then got a smart remote control and well I will write a blog post about that in the future as there is a lot to talk about that. I had like 10 different apps to control all the devices I had so I wanted an integrated solution. The Alexa app originally gave me that, All devices where integrated in 1 app and I could writ routines (automations) for them. But I realized Some things I wanted to do fell short, plus it all depended on being connected to the internet, so I decided to integrate something that will give me local control and still be running if internet was down. THen I found the open source project [Home Assistant](https://www.home-assistant.io/).  
At fist I ran home assistant standalone in a raspberry pi and it ran well for a long time. I had it backed up and such until the sd card died. Then I had to reconfigure it all over, not that hard with proper backups, but still I had to go to the trouble of reinstalling and restoring it. At the same time I added a lot of zigbee devices and integrated them to Home Assistant with [zigbee2mqtt](https://www.zigbee2mqtt.io/) and that allowed me to add multiple Raspberry Pis (3 actually) to extend the zigbee network and used mosquito as the mqtt broker.  
All of the sudden I found myself with a distributed system at my home. And since I have little time for fixing the automation things needed to be easy to recover and reliable. So I decided to create a more sophisticated system than a bunch of rpis doing standalone work.  
This is when I decided to add k8s, and make my system more reliable and easy to recover. So one of the main focus was the MTTR (mean time to recover) so I applied the same ideas and architecture I was already doing at work.  

# Requirements

Before jumping into things I decided to gather requirements for my HA home automation. Even if some components were not prepared for HA at least I made things much easier. So here are the requirements I thought.

#### Energy efficient
First of all I wanted to run a cluster, I have decided on k8s, but I needed something with low power and even smallest mini PCs can consume up to 23-30 watts so that, not to mention small server hardware. For an HA cluster I needed at least 3 nodes, and probably with a spare. So even running 4 or 5 nodes I would have about 150 Watts consumption.  
I have been playing with DIY and raspberry pis for a long time and with the appearance of the rpi4 with 4GB of ram a k8s cluster seemed feasible. SO I was getting addicted to get raspberry pis. So at the time of this writing, even though this started over 4 years ago, I have 3 raspberry pi 3B, 1 raspberry pi 3B+, 8 Raspberry pi 4 with 4GB ram and 1 Raspberry pi 4 with 8GB ram. I wish I could get more 8GB ones but that is when the shortage started to happen. Anyway, the idea was that RPIs use power supply of 5v 3A which gives a total of 15 watts at full blast. So I decided for the arm based solution and I prioritized more having power efficiency and sacrificed a bit more performance. Most applications I run don't need high performance anyway but I will talk about that later.

#### Fully automated base setup

Another requirement was automation. I wanted the setup to be fully automated. I knew many things were going to be trial and error and for a project I do and my little free time I had to automate from scratch, this seems counter intuitive but it was the right call.  
First I worked on just setting up the raspberry pi's with all the settings I needed for my network and software I needed like monitoring software, etc.  
I choose ubuntu as the main distro because that was something I am familiar with and I have been using it in x86_64 systems as well so most of automation I had works seamlessly in both platforms x86_64 and arm.
The requirement here was to flash/write the standard ubuntu image and have playbook (ansible) run and have the node set up with no human intervention.  
This was a hard part, as I was eager to get the cluster going, but I knew that I would have to recreate the cluster multiple times until getting things right.

#### The cluster must withstand power outages

Where I live power outages are not uncommon, even short ones or when there are thunderstorms. I first thought of investing on a UPS since the cluster would draw very little power, however UPS don't last forever, are expensive and after a few years you need to change batteries, so I decided not to go for the UPS solution.  
This was pretty easy to test, just unplug everything and plug back in and the cluster should come back up by itself.


#### The cluster must withstand node failures

The main reason for the cluster was to run all my home automation software and metrics gathering (prometheus/thanos). Yes I am a freak of measuring things, network, temp, humidity, power consumption, motion. whatever you can measure with a sensor my house will measure it. 
So everything revolves around home assistant, prometheus and grafana. Prometheus and grafana (with HA postgres backend) were fine because they have a way of running in high availability but home assistant doesn't support running multiple instances. But anyway a kubernetes cluster would give some more reliability from what I had with a standalone raspberry pi that could die at any time.

#### Needed a storage layer easy to manage

Since I wanted to do high availability host-path provisioner was not in question. I had to find some option for the storage, Home assistant, prometheus and grafana needed some sort of storage in the end. I was tempted to get a QNAP or Synology NAS and use iSCSI to mount volumes, however that solution was in conflict with the previous section `The cluster must withstand node failures` and an HA solution from those would also be expensive and more configuration overhead.  
After some research and some work I was doing at my day job, I decided for an hyper-converged infrastructure which the storage layer and the compute layer are shared. At the moment seemed the most cost efficient option to do, just add a bunch of disks to the raspberry pis and that is it... well was not trivial and I will talk about that later.
I tested OpenEBS, rook ceph and longhorn. None of the solutions were trivial and required some skill knowing how storage systems work. However I decided to use longhorn. At the moment looked easier to set up and manage even with a GUI and had features I needed out of the box like backup up volumes to cloud storage.


#### Fast restore

Having a home automated for a while, you get used to the automations, so a fast recovery of the system was a must. Otherwise I would get an angry wife because lights are not working or the garage door is not opening. LoL.  
So if something went sideways I needed to restore it in a quick fashion.



# The cluster

So now that I presented the requirements and some context I can talk about the cluster.  

So as mentioned before the cluster is an arm based architecture and the hardware are raspberry pi

- 1 raspberry pi 4B with 8GB RAM
- 8 Raspberry pi 4B with 4GB RAM

That gives me a 9 node cluster.

On the storage side it is a mix drives of whatever I had dangling around I would add it to the storage layer. So I have a couple of mSata ssd with an USB mSata shield with 256GB storage, then I have 4 2.5" sata ssd drives with USB adapter case, and even and old 64GB sata SSD I had around with a USB adapter case.

### Choosing the k8s distribution

I have been playing around with k8s distros for a while, I even have older posts on how to run them locally in your machine. So distros I tested originally as proof of concepts were k3s, microk8s and k0s. Spoiler... I actually created everything so distro didn't matter and all workloads run unmodified in all 3 distros.  
The decision was driven a bit because of affinity to Cannonical open source products plus some small testing and requirements I have.
Main requirement I had is that control plane of k8s should be highly available. Spolier... I went with microk8s.  
All 3 support high availability in some way, but the one that makes it very seamless for the user is microk8s you just create the first node and add nodes to it and after the 3rd node HA is up and running.... cool right and you can even add a spare node just in case one dies, all of it with a few command line statements.  
On the other hand k3s at the moment I made this decision the embedded HA etc solution was still in beta and it wasn't stable enough for my requirements. So even though I tested everything with k3s it wasn't quite ready for what I wanted, and the other HA solutions for k3s was to use an external database which required more involvement.  
And lastly I tested k8s which they claimed that it was the k8s distro with the lowest memory footprint. And based on my testing indeed was the one that required less memory... but... there is always a but. When running HA on a rpi with 4G the control plane was using around 1GB, pretty low while microk8s and k3s near 2GB. But there was a catch. When I looked at the CPU usage the control plane was taking 75% of the cores on each of the master nodes. So I had little room for running anything else because of CPU constraint.  
Off course this is not an exhaustive comparison of the 3, I chose what I knew at the moment was best suited for my needs.


### Setting up the cluster

Actually the first time I ran the cluster was pretty straightforward. I followed the microk8s guide and everything worked out of the box. I am kind of old fashioned and I need to run things by hand just to know the inner workings of the what I am installing and then automate it. Then I went with ansible for the automation as I have rebuild my cluster over 30 times until I got things right. Unfortunately most of my code is not open sourced yet as it needs some cleaning before someone else can use it. But there are some automations and ansible roles I use for the raspberry pi's that ware open sourced [here](https://github.com/fcastello). 
In this post I am not focusing on the little details of configuring everything, but focusing about the story and the lessons learnt and hope that someone else reads them and gets ahead of the issues I had to get a fully Highly Available cluster at home.  
The cluster setup is pretty simple. I have 3 master nodes 1 8GB ram and 2 with 4GB ram. And then 6 nodes for the worker nodes. The master nodes with 4GB ram are tainted so no other workloads run on them while the 8GB node can run workloads. I will talk about that in a while, but that setup can change at any time.  
As mentioned before storage layer is served by longhorn and configured to save the critical volumes to cloud storage. The setup is pretty straight forward and default values are enough for getting started. Then I found some tweaks that helped me automation in this case was easy as installation are plain k8s manifests or helm chart.


# Challenges

Issues I faced while getting the clusters ready were mostly related to hardware issues rather than cluster itself. This lightweight flavors of k8s are doing a great job at handling most of the things for you. The most common problems were loosing a few nodes and loosing quorum in the masters or replicated volumes. But the issues were not k8s itself they were more lower level.  
So turned out that setting a raspberry pi k8s cluster


