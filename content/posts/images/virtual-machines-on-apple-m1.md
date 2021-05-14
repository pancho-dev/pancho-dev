---
title: "Running virtual machins on apple M1"
date: 2021-03-16T15:11:30-03:00
draft: true
tags: ["vistualization","linux", "apple"]
---


```bash
#!/usr/bin/env bash
#Install brew and qemu + cloud init metadata dependencies
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)" 
brew install qemu
brew install cdrtools

rm -rf /tmp/ubuntuqemuboot

#download Ubuntu 20.04 Cloud Image and resize to 30 Gigs
mkdir -p /tmp/ubuntuqemuboot/images
cd /tmp/ubuntuqemuboot/images
curl https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img --output focal-server-cloudimg-amd64.img
qemu-img resize focal-server-cloudimg-amd64.img 30G

#create the cloud-init NoCloud metadata disk file
mkdir -p /tmp/ubuntuqemuboot/cloudinitmetadata
cd /tmp/ubuntuqemuboot/cloudinitmetadata
ssh-keygen -b 2048 -t rsa -f id_rsa_ubuntu2004boot -P ""
chmod 0600 /tmp/ubuntuqemuboot/cloudinitmetadata/id_rsa_ubuntu2004boot
PUBLIC_KEY=$(cat id_rsa_ubuntu2004boot.pub)

cat <<EOF >/tmp/ubuntuqemuboot/cloudinitmetadata/meta-data
instance-id: circle-the-wagons-local716
local-hostname: circle-the-wagons
EOF

cat <<EOF >/tmp/ubuntuqemuboot/cloudinitmetadata/user-data
#cloud-config
debug: true
disable_root: false
users:
  - name: root
    shell: /bin/bash
    ssh-authorized-keys:
      - ${PUBLIC_KEY}
EOF

#create the cloud-init optical drive
mkisofs -output cidata.iso -volid cidata -joliet -rock user-data meta-data

#boot the machine up
qemu-system-x86_64 -m 2048 -smp 4 -hda /tmp/ubuntuqemuboot/images/focal-server-cloudimg-amd64.img -cdrom /tmp/ubuntuqemuboot/cloudinitmetadata/cidata.iso -device e1000,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22 -nographic


```