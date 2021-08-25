---
title: "Rpi Cloud Init Ubuntu"
date: 2021-05-14T20:28:48-03:00
draft: true
---



Create partition for the cidata partition on USB flash drive. Make sure you delet all partitions before running fdisk. This is an exmaple hot to partition it but in other cases device names and sizes will vary
```bash
# Commmands
$ fdisk /dev/sdb

Welcome to fdisk (util-linux 2.31.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): p
Disk /dev/sdb: 3.8 GiB, 4034969600 bytes, 7880800 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x20ab893b

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First sector (2048-7880799, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-7880799, default 7880799): +32M

Created a new partition 1 of type 'Linux' and of size 32 MiB.

Command (m for help): p
Disk /dev/sdb: 3.8 GiB, 4034969600 bytes, 7880800 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x20ab893b

Device     Boot Start   End Sectors Size Id Type
/dev/sdb1        2048 67583   65536  32M 83 Linux

Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): l

 0  Empty           24  NEC DOS         81  Minix / old Lin bf  Solaris
 1  FAT12           27  Hidden NTFS Win 82  Linux swap / So c1  DRDOS/sec (FAT-
 2  XENIX root      39  Plan 9          83  Linux           c4  DRDOS/sec (FAT-
 3  XENIX usr       3c  PartitionMagic  84  OS/2 hidden or  c6  DRDOS/sec (FAT-
 4  FAT16 <32M      40  Venix 80286     85  Linux extended  c7  Syrinx
 5  Extended        41  PPC PReP Boot   86  NTFS volume set da  Non-FS data
 6  FAT16           42  SFS             87  NTFS volume set db  CP/M / CTOS / .
 7  HPFS/NTFS/exFAT 4d  QNX4.x          88  Linux plaintext de  Dell Utility
 8  AIX             4e  QNX4.x 2nd part 8e  Linux LVM       df  BootIt
 9  AIX bootable    4f  QNX4.x 3rd part 93  Amoeba          e1  DOS access
 a  OS/2 Boot Manag 50  OnTrack DM      94  Amoeba BBT      e3  DOS R/O
 b  W95 FAT32       51  OnTrack DM6 Aux 9f  BSD/OS          e4  SpeedStor
 c  W95 FAT32 (LBA) 52  CP/M            a0  IBM Thinkpad hi ea  Rufus alignment
 e  W95 FAT16 (LBA) 53  OnTrack DM6 Aux a5  FreeBSD         eb  BeOS fs
 f  W95 Ext d (LBA) 54  OnTrackDM6      a6  OpenBSD         ee  GPT
10  OPUS            55  EZ-Drive        a7  NeXTSTEP        ef  EFI (FAT-12/16/
11  Hidden FAT12    56  Golden Bow      a8  Darwin UFS      f0  Linux/PA-RISC b
12  Compaq diagnost 5c  Priam Edisk     a9  NetBSD          f1  SpeedStor
14  Hidden FAT16 <3 61  SpeedStor       ab  Darwin boot     f4  SpeedStor
16  Hidden FAT16    63  GNU HURD or Sys af  HFS / HFS+      f2  DOS secondary
17  Hidden HPFS/NTF 64  Novell Netware  b7  BSDI fs         fb  VMware VMFS
18  AST SmartSleep  65  Novell Netware  b8  BSDI swap       fc  VMware VMKCORE
1b  Hidden W95 FAT3 70  DiskSecure Mult bb  Boot Wizard hid fd  Linux raid auto
1c  Hidden W95 FAT3 75  PC/IX           bc  Acronis FAT32 L fe  LANstep
1e  Hidden W95 FAT1 80  Old Minix       be  Solaris boot    ff  BBT
Hex code (type L to list all codes): b
Changed type of partition 'Linux' to 'W95 FAT32'.

Command (m for help): p
Disk /dev/sdb: 3.8 GiB, 4034969600 bytes, 7880800 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x20ab893b

Device     Boot Start   End Sectors Size Id Type
/dev/sdb1        2048 67583   65536  32M  b W95 FAT32

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.

# Formate as vfat with cidata label
$  mkfs.vfat -n CIDATA /dev/sdb1
mkfs.fat 4.1 (2017-01-24)



$ mount /dev/sdb1 /mnt/

```


on the USB flash drive create a file names `meta-data` with the following contents

```yaml
instance_id: ubuntu20210412
local-hostname: ubuntupi
```
- instance_id: should be what identifies you instance (this is notmally for cloud providers autimation) for home purposes you this value will not be used
- local-hostname: will set the instance hostname. I Normally recomend seting this to a default value and then override with other automation tools like ansible, puppet or cheff


Then create a file named `user-data` which will contain custom user configurations. This is usefull for setting usernames, copying files, installing basic software and many more options. Check cloud-init modules to see options about what can be set from this file.

```yaml
#cloud-config
users:
  - default
  - name: francisco
    primary_group: francisco
    home: /home/francisco
    lock_passwd: True
    groups: [adm, audio, cdrom, dialout, floppy, video, plugdev, dip, netdev]
    ssh_authorized_keys:
        - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDhrezi2yRjI6h25Zj+3boHomi7osKP9HDvG3xQseXYZmBf3m7VNhFrJy3YbirmZzOrpB2q9+M0pLn1NYGP0yBfCAVo9HXoPw+qkYCd/Kb2JHUNjBbDZP9ICx+loxv5FQ6C2nCjqlXdMzuuCxvGjhl/4JNycVxWnWpgZnOzgLr81KLjdnICwFY4+r7Yv3Y07iRgDxLSqEXBh2xTd576krDdAolcQ1vD/bEbYgaFHvLYhZjC4MiYnm3m59o+aNdwJPAJvi5g2F05VWLiyVM5dni/n25Vuow+TweqAwxhn3Lj070Ggh+0Uul7zgcuPEFVwiAWCaMBQsJNpd7bVMUM8htG0lRWUV6OG74DyK2yCNL11k2mIpwTy4tVRah+RmgIIEll2G6OIo5/XEra4/4Jg5USM7Ntm2zNe5WtdW57gdORCYjUWNNSoQ/FwoPAE7blV7F26kVF2EYmq6giwvvkuFoIZXz0Pd37AKEP6vu1vRYRBkakVjusiX6pShIToq4xYpE= francisco


```

`network-config` file
```yaml
version: 2
ethernets:
  # opaque ID for physical interfaces, only referred to by other stanzas
  eth0:
    dhcp4: no
    addresses:
      - 192.168.2.4/24  
    gateway4: 192.168.2.1
    nameservers:
      addresses: [1.1.1.1,8.8.8.8]
```