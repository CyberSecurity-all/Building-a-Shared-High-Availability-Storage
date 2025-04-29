# Building-a-Shared-High-Availability-Storage
Building a shared storage high availability (HA) cluster with two nodes based on DRBD with RDMA-based network

 Building a shared storage high availability (HA) cluster with two nodes based on DRBD with RDMA-based network.
Live migration.
1.1. Beginning. Due to the reduction in the price of network cards (I bought a pair of dual-port ones with a pair of copper cables for $100 with delivery), there was a desire to build a cluster of two nodes with two points of failure, connected directly by three wired networks and one WiFi, each having one nvme disk for shared storage. This was supposed to be a cluster option for a home or a small company. The choice fell on drbd storage from rdma.
1.2. Implementation.
1.2.1. Hardware. System. Configuration.

Two nodes with Proxmox 8.3.2 installed:

root@pve1:~# pveversion 
pve-manager/8.3.2/3e76eec21c4a14a7 (running kernel: 6.8.12-5-pve)
[root@pve99 ~]$ pveversion
pve-manager/8.3.2/3e76eec21c4a14a7 (running kernel: 6.8.12-5-pve)

Mellanox Technologies ConnectX-3 Pro Stand-up dual-port 40GbE MCX314A-BCCT dual-port cards are connected directly via copper cables.

Node pve1: 10.10.1.1 Part of the command output:

lspci -vvv  
01:00.0 Ethernet controller: Mellanox Technologies MT27520 Family [ConnectX-3 Pro]  
Subsystem: Mellanox Technologies Mellanox Technologies ConnectX-3 Pro Stand-up dual-port 40GbE MCX314A-BCCT
...
Kernel driver in use: mlx4_core
Kernel modules: mlx4_core


Node pve99: 10.10.1.2 Part of the command output:

lspci -vvv
01:00.0 Ethernet controller: Mellanox Technologies MT27520 Family [ConnectX-3 Pro]
Subsystem: Mellanox Technologies Mellanox Technologies ConnectX-3 Pro Stand-up dual-port 40GbE MCX314A-BCCT 
...
Kernel driver in use: mlx4_core
Kernel modules: mlx4_core

1.2.2. Checking the functionality. (Installing drivers and settings are discussed separately at the link:

Let's install the rping package on both nodes:

apt install rdmacm-utils

On the server:

Node 1: pve1 (10.10.1.1):

root@pve1:~# rping -s -v 

On the client:

Node 2: pve99 (10.10.1.2):

[root@pve99 ~]$ rping -c -a 10.10.1.1 -v

We will see something like this output:

ping data: rdma-ping-54436: abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRST
ping data: rdma-ping-54437: bcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTU
ping data: rdma-ping-54438: cdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUV
ping data: rdma-ping-54439: defghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVW
ping data: rdma-ping-54440: efghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWX
ping data: rdma-ping-54441: fghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXY
ping data: rdma-ping-54442: ghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ
ping data: rdma-ping-54443: hijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ[
ping data: rdma-ping-54444: ijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ[\
ping data: rdma-ping-54445: jklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ[\]
ping data: rdma-ping-54446: klmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ[\]^
^C
[root@pve99 ~]$

Everything is fine.
4.2.3. On both nodes we install:

apt install drbd-utils
apt install build-essential flex bison libssl-dev libnl-3-dev libnl-genl-3-dev libxml2-dev xmlto xsltproc python3-pytest python3-sphinx python3-yaml python3-jinja2 dkms

We definitely need to stop the service.

systemctl list-units | grep drbd
drbdadm down all

Unloading modules is a must!!!:

lsmod | grep drbd
systemctl stop drbd
rmmod drbd

Download the latest version:

wget https://pkg.linbit.com//downloads/drbd/9/drbd-9.2.12.tar.gz

Unpacking:

tar xfz drbd-9.2.12.tar.gz

We go into the directory and compile and install:

cd drbd-9.2.12
make KVER=$(uname -r) all
make install

Installing the module

Copy the compiled module to the appropriate kernel directory:


cp  /lib/modules/$(uname -r)/updates/drbd..ko /lib/modules/$(uname -r)/kernel/drivers/block/
cp  /lib/modules/$(uname -r)/updates/drbd_transport_rdma.ko /lib/modules/$(uname -r)/kernel/drivers/block/
cp  /lib/modules/$(uname -r)/updates/drbd_transport_tcp.ko /lib/modules/$(uname -r)/kernel/drivers/block/
cp  /lib/modules/$(uname -r)/updates/drbd_transport_lb-tcp.ko /lib/modules/$(uname -r)/kernel/drivers/block/

Update the list of available modules:

depmod -a

Load modules manually:

modprobe drbd
modprobe drbd_transport_rdma
modprobe drbd_transport_lb-tcp
modprobe drbd_transport_tcp
modinfo drbd

First node pve1:

root@pve1:~# cat /proc/drbd
version: 9.2.12 (api:2/proto:118-122)
GIT-hash: 2da6f528dc4ab3fd25c511f7b03531100e54ab08 build by root@pve1, 2024-12-17 19:11:24
Transports (api:21): rdma (9.2.12) lb-tcp (9.2.12) tcp (9.2.12)

root@pve1:~#
root@pve1:~# modinfo drbd
filename:       /lib/modules/6.8.12-5-pve/updates/drbd.ko
softdep:        post: handshake
alias:          block-major-147-*
license:        GPL
version:        9.2.12
description:    drbd - Distributed Replicated Block Device v9.2.12
author:         Philipp Reisner <phil@linbit.com>, Lars Ellenberg <lars@linbit.com>
srcversion:     C0FA687B694B5082F797130
depends:        lru_cache,libcrc32c
retpoline:      Y
name:           drbd
vermagic:       6.8.12-5-pve SMP preempt mod_unload modversions 
parm:           enable_faults:int
parm:           fault_rate:int
parm:           fault_count:int
parm:           fault_devs:int
parm:           disable_sendpage:bool
parm:           allow_oos:DONT USE! (bool)
parm:           minor_count:Approximate number of drbd devices (1U-255U) (uint)
parm:           usermode_helper:string
parm:           protocol_version_min:
                Reject DRBD dialects older than this.
                Supported: DRBD 8 [86-101]; DRBD 9 [118-122].
                Default: 86 (drbd_protocol_version)
parm:           strict_names:restrict resource and connection names to ascii alnum and a subset of punct (drbd_strict_names)
root@pve1:~#

Second node pve99:

[root@pve99 ~]$ cat /proc/drbd
version: 9.2.12 (api:2/proto:118-122)
GIT-hash: 2da6f528dc4ab3fd25c511f7b03531100e54ab08 build by root@pve99, 2024-12-17 23:30:23
Transports (api:21): rdma (9.2.12) lb-tcp (9.2.12) tcp (9.2.12)
[root@pve99 ~]$

[root@pve99 ~]$ modinfo drbd
filename:       /lib/modules/6.8.12-5-pve/updates/drbd.ko
softdep:        post: handshake
alias:          block-major-147-*
license:        GPL
version:        9.2.12
description:    drbd - Distributed Replicated Block Device v9.2.12
author:         Philipp Reisner <phil@linbit.com>, Lars Ellenberg <lars@linbit.com>
srcversion:     45B06AA2C33AAFE56B535F5
depends:        lru_cache,libcrc32c
retpoline:      Y
name:           drbd
vermagic:       6.8.12-5-pve SMP preempt mod_unload modversions 
parm:           enable_faults:int
parm:           fault_rate:int
parm:           fault_count:int
parm:           fault_devs:int
parm:           disable_sendpage:bool
parm:           allow_oos:DONT USE! (bool)
parm:           minor_count:Approximate number of drbd devices (1U-255U) (uint)
parm:           usermode_helper:string
parm:           protocol_version_min:
                Reject DRBD dialects older than this.
                Supported: DRBD 8 [86-101]; DRBD 9 [118-122].
                Default: 86 (drbd_protocol_version)
parm:           strict_names:restrict resource and connection names to ascii alnum and a subset of punct (drbd_strict_names)
[root@pve99 ~]$

Let's add loading of modules to the file on both nodes:

nano /etc/modules

Example file:

# /etc/modules: kernel modules to load at boot time.
#
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.
# Parameters can be specified after the module name.

# Generated by sensors-detect on Sun Sep 29 22:12:00 2024
# Chip drivers
it87
vfio
vfio_iommu_type1
vfio_pci
#vfio_virqfd #not necessary if kernel 6.2

nvmet
nvmet-tcp
nvme-rdma
rdma_ucm
rdma_cm
ib_uverbs
mlx4_ib
drbd
drbd_transport_rdma
drbd_transport_lb-tcp
drbd_transport_tcp

1.3. Configuring DRBD devices.

There are two nodes. Each node has a disk: nvme0n1, by the way, when loading, the order can change: nvme0n1, nvme1n1… if there are several disks, so let's consider the case by label. When we pair disks, it is better to have them identical, identically labeled.
1.3.1 Preparing disks

Check the disk sector size. Set the same, change if necessary for maximum performance Node 1, pve1

root@pve1:~# nvme list
Node Generic SN Model Namespace Usage Format FW Rev
/dev/nvme1n1 /dev/ng1n1 S4EUNG0M328258D Samsung SSD 970 EVO Plus 250GB 1 214.99 GB / 250.06 GB 512 B + 0 B 1B2QEXM7
/dev/nvme0n1 /dev/ng0n1 50026B7282A726A4 KINGSTON SKC3000S512G 1 512.11 GB / 512.11 GB 4 KiB + 0 B EIFK31.6

Checking the block size

root@pve1:~# nvme id-ns /dev/nvme0 -n 1 -H | grep &quot;LBA Format&quot;
[6:5] : 0 Most significant 2 bits of Current LBA Format Selected
[3:0] : 0x1 Least significant 4 bits of Current LBA Format Selected
LBA Format 0 : Metadata Size: 0 bytes - Data Size: 512 bytes - Relative Performance: 0x2 Good
LBA Format 1 : Metadata Size: 0 bytes - Data Size: 4096 bytes - Relative Performance: 0x1 Better (in use)
root@pve1:~#

If necessary, change to 4k

root@pve1:~# nvme id-ns /dev/format --lbaf=1 /dev/nvme0n1

Similar to node 2, pve99
1.3.2. Using fdisk, we will first partition the first one as follows:

root@pve1:~# fdisk -l /dev/nvme0n1
Disk /dev/nvme0n1: 476.94 GiB, 512110190592 bytes, 125026902 sectors
Disk model: KINGSTON SKC3000S512G                   
Units: sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: A1F37274-73E6-864F-B0B6-9BDD551BBD45

Device             Start       End  Sectors  Size Type
/dev/nvme0n1p1      4096    266239   262144    1G Linux swap
/dev/nvme0n1p2    266240  21237759 20971520   80G Linux filesystem
/dev/nvme0n1p3 105123840 121901055 16777216   64G Linux filesystem
/dev/nvme0n1p4 121901056 125026815  3125760 11.9G Linux swap
/dev/nvme0n1p5  21237760  63180799 41943040  160G Linux filesystem
/dev/nvme0n1p6  63180800 105123839 41943040  160G Linux filesystem

Partition table entries are not in disk order.
root@pve1:~#

Let's save the disk layout to a file:

root@pve1:~# sfdisk -d /dev/nvme0n1 > nvmeKINGSTON512P6.dump

It will look something like this:

root@pve1:~# cat nvmeKINGSTON512P6.dump
label: gpt
label-id: A1F37274-73E6-864F-B0B6-9BDD551BBD45
device: /dev/nvme0n1
unit: sectors
first-lba: 256
last-lba: 125026896
sector-size: 4096  

/dev/nvme0n1p1 : start=        4096, size=      262144, type=0657FD6D-A4AB-43C4-84E5-0933C84B4F4F, uuid=B21C1B97-64EE-6948-AA0F-0BBA8797EB91
/dev/nvme0n1p2 : start=      266240, size=    20971520, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=8291281C-CA4C-0F4C-AD9E-C30C52FFBC04
/dev/nvme0n1p3 : start=   105123840, size=    16777216, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=47A78E4D-FC8E-F54B-8DAF-D52A7908590C
/dev/nvme0n1p4 : start=   121901056, size=     3125760, type=0657FD6D-A4AB-43C4-84E5-0933C84B4F4F, uuid=7F23E207-9E1A-1B42-962F-98BED3C1F479
/dev/nvme0n1p5 : start=    21237760, size=    41943040, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=A00C2C46-01F3-1B48-9AB2-B458FDADC3D7
/dev/nvme0n1p6 : start=    63180800, size=    41943040, type=0FC63DAF-8483-4772-8E79-3D69D8477DE4, uuid=207CEE45-7593-6441-9FAF-99E294452177
root@pve1:~#

Now, on another node, we will immediately save the layout to disk, and we will do the same when replacing a damaged disk:

sfdisk /dev/nvme0n1 < nvmeKINGSTON512P6.dump

Accordingly, we have:

[root@pve99 ~]$ fdisk -l /dev/nvme0n1
Disk /dev/nvme0n1: 476.94 GiB, 512110190592 bytes, 125026902 sectors
Disk model: KINGSTON SKC3000S512G                   
Units: sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: A1F37274-73E6-864F-B0B6-9BDD551BBD45

Device             Start       End  Sectors  Size Type
/dev/nvme0n1p1      4096    266239   262144    1G Linux swap
/dev/nvme0n1p2    266240  21237759 20971520   80G Linux filesystem
/dev/nvme0n1p3 105123840 121901055 16777216   64G Linux filesystem
/dev/nvme0n1p4 121901056 125026815  3125760 11.9G Linux swap
/dev/nvme0n1p5  21237760  63180799 41943040  160G Linux filesystem
/dev/nvme0n1p6  63180800 105123839 41943040  160G Linux filesystem

Partition table entries are not in disk order.
[root@pve99 ~]$

We also see identical disk partition labels, which we will use later:

[root@pve99 ~]$ blkid /dev/nvme0n1p5
/dev/nvme0n1p5: UUID="6b33cc5a02d7cc72" TYPE="drbd" PARTUUID="a00c2c46-01f3-1b48-9ab2-b458fdadc3d7"
[root@pve99 ~]$ blkid /dev/nvme0n1p6
/dev/nvme0n1p6: UUID="f0eb844c4d858ddc" TYPE="drbd" PARTUUID="207cee45-7593-6441-9faf-99e294452177"
[root@pve99 ~]$

1.3.3. Setting up LVM filters. If you are going to use lvm for DRBD devices. THIS IS VERY IMPORTANT! In order for lvm to work on top of the drbd device and not touch the corresponding physical devices, you need to set up the /etc/lvm/lvm.conf file.

Edit on both nodes:

nano /etc/lvm/lvm.conf

The type of part of a file, usually its end:

devices {
     # added by pve-manager to avoid scanning ZFS zvols and Ceph rbds
     filter=["r|/dev/zd.*|","r|/dev/rbd.*|",
#"r|/dev/mapper/vg_drbd.*|",
#"r|/dev/.*vg_drbd.*|",
"r|.*a00c2c46-01f3-1b48-9ab2-b458fdadc3d7.*|",
"r|.*207cee45-7593-6441-9faf-99e294452177.*|",
"a|/dev/drbd.*|"]
     global_filter=["r|/dev/zd.*|","r|/dev/rbd.*|",
"r|.*a00c2c46-01f3-1b48-9ab2-b458fdadc3d7.*|",
"r|.*207cee45-7593-6441-9faf-99e294452177.*|",
"a|/dev/drbd.*|"]
}

MANDATORY! To apply, you need to run the command on both nodes, after which a reboot is required: update-initramfs -u
1.3.3. Create drbd devices: drbd0 from the physical device: nvme0n1p5 (or more precisely with PARTUUID="a00c2c46-01f3-1b48-9ab2-b458fdadc3d7"), drbd1 from the physical device: /dev/nv5 1-9faf-99e294452177"). Do this on both nodes (make files on one node and copy them to the other node).

nano /etc/drbd.d/r0.res

File type:

resource r0 {
    protocol  C;
#    device    /dev/drbd0 minor 0;
#    meta-disk internal;
    on pve1 {
            address 10.10.2.1:7788;
                volume 0 {
                    device    /dev/drbd0 minor 0;
                    disk /dev/disk/by-partuuid/a00c2c46-01f3-1b48-9ab2-b458fdadc3d7;
                    #disk /dev/nvme0n1p5;
                    meta-disk internal;
                }
    }
    on pve99 {
            address 10.10.2.2:7788;
                volume 0 {
                    device    /dev/drbd0 minor 0;
                    disk /dev/disk/by-partuuid/a00c2c46-01f3-1b48-9ab2-b458fdadc3d7;
                    #disk /dev/nvme0n1p5;
                    meta-disk internal;
                }
    }
    startup {
        degr-wfc-timeout 60;
        become-primary-on both;
    }
    disk {
        on-io-error   detach;
        c-plan-ahead  10;
        c-fill-target 100K;
        c-min-rate    500M;
        c-max-rate    1000M;

        no-disk-flushes;
        no-disk-barrier;
    }
    net {
        transport   rdma;
        max-buffers 36k;
        sndbuf-size 10M;
        rcvbuf-size 10M;
        allow-two-primaries;
    }
}

File for the second resource:

nano /etc/drbd.d/r1.res

File type:

resource r1 {
    protocol  C;
#    device    /dev/drbd1 minor 1;
#    meta-disk internal;
    on pve1 {
            address 10.10.2.1:7789;
        volume 1 {
            # device name
            device /dev/drbd1  minor 1;
            # specify disk to be used for devide above
            disk /dev/disk/by-partuuid/207cee45-7593-6441-9faf-99e294452177;
            #disk /dev/nvme0n1p6;
            # where to create metadata
            # specify the block device name when using a different disk
            meta-disk internal;
        }
    }
    on pve99 {
             address 10.10.2.2:7789;
        volume 1 {
            device /dev/drbd1  minor 1;
            disk /dev/disk/by-partuuid/207cee45-7593-6441-9faf-99e294452177;
            #disk /dev/nvme0n1p6;
            meta-disk internal;
        }
    }
    startup {
        degr-wfc-timeout 60;
        become-primary-on both;
    }
    disk {
        on-io-error   detach;
        c-plan-ahead  10;
        c-fill-target 100K;
        c-min-rate    500M;
        c-max-rate    1000M;

        no-disk-flushes;
        no-disk-barrier;
    }
    net {
        transport   rdma;
        max-buffers 36k;
        sndbuf-size 10M;
        rcvbuf-size 10M;
        allow-two-primaries;
    }
}

1.3.4. We start the service and create resources, we do this on both nodes:

 # systemctl enable --now drbd

# systemctl restart drbd
# drbdadm create-md r{0,1}
# drbdadm up r{0,1}

Then, on just one node, we set the resources to their initial state and run the initial sync:

root@pve1:~# drbdadm primary --force r{0,1}

Waiting for synchronization.

We do the same on the second node.

root@pve99:~# drbdadm primary --force r{0,1}

Let's check:

[root@pve99 ~]$ drbdadm status

r0 role:Primary
  disk:UpToDate open:no
  pve1 role:Primary
    peer-disk:UpToDate

r1 role:Primary
  volume:1 disk:UpToDate open:no
  pve1 role:Primary
    volume:1 peer-disk:UpToDate

[root@pve99 ~]$

1.3.5 Next we create physical LVM DRBD devices on both nodes:

root@pve1:~# pvcreate /dev/drbd{0,1}
  Physical volume "/dev/drbd0" successfully created
  Physical volume "/dev/drbd1" successfully created
 
root@ve99:~# pvcreate /dev/drbd{0,1}
  Physical volume "/dev/drbd0" successfully created
  Physical volume "/dev/drbd1" successfully created

і створіть групи томів лише на одному з вузлів:
root@pve1:~# vgcreate vg_drbd0 /dev/drbd0
  Volume group "vg_drbd0" successfully created
 
root@pve1:~# vgcreate vg_drbd1 /dev/drbd1
  Volume group "vg_drbd1" successfully created

Now the groups can be seen on both nodes thanks to DRBD replication:

root@pve1:~# vgs
  VG       #PV #LV #SN Attr   VSize    VFree  
  os         1  17   0 wz--n- <465.76g  71.73g
  pve        1   8   0 wz--n-  231.88g  16.00g
  vg_drbd0   1   5   0 wz--n-  159.99g 106.99g
  vg_drbd1   1   2   0 wz--n-  159.99g 147.99g
root@pve1:~#

Second:

[root@pve99 ~]$ root@pve1:~pvs
  PV         VG       Fmt  Attr PSize   PFree  
  /dev/drbd0 vg_drbd0 lvm2 a--  159.99g 106.99g
  /dev/drbd1 vg_drbd1 lvm2 a--  159.99g 147.99g
  /dev/sda3  pve      lvm2 a--  <36.76g   4.50g
[root@pve99 ~]$

1.3.6. Shared storage.

We go to the PVE admin web console and add the LVM storage to Datacenter, select vg_drbd0 from the drop-down list and check the boxes for active and shared. In the Nodes drop-down list, we select both nodes pve1 and pve99 and click Add. Repeat the same for vg_drbd1.
1.3.7. Let's create a virtual machine with a disk on shared storage.

Let's run a test:

root@debvsan:/home/vov# fio --filename=/dev/sda1 --direct=1 --rw=read --bs=1m --size=20G --numjobs=200 --runtime=60 --group_reporting --name=file1 

file1: (g=0): rw=read, bs=(R) 1024KiB-1024KiB, (W) 1024KiB-1024KiB, (T) 1024KiB-1024KiB, ioengine=psync, iodepth=1
...
fio-3.33
Starting 200 processes
Jobs: 200 (f=200): [R(200)][100.0%][r=1368MiB/s][r=1368 IOPS][eta 00m:00s]
file1: (groupid=0, jobs=200): err= 0: pid=1092: Sun Dec 29 13:12:07 2024
  read: IOPS=1367, BW=1368MiB/s (1434MB/s)(80.3GiB/60130msec)
    clat (msec): min=2, max=1177, avg=145.78, stdev=97.77
     lat (msec): min=2, max=1177, avg=145.78, stdev=97.77
    clat percentiles (msec):
     |  1.00th=[   30],  5.00th=[   52], 10.00th=[   73], 20.00th=[   94],
     | 30.00th=[   97], 40.00th=[  103], 50.00th=[  113], 60.00th=[  131],
     | 70.00th=[  153], 80.00th=[  188], 90.00th=[  253], 95.00th=[  330],
     | 99.00th=[  514], 99.50th=[  592], 99.90th=[  936], 99.95th=[  995],
     | 99.99th=[ 1150]
   bw (  MiB/s): min=  399, max= 2673, per=100.00%, avg=1385.68, stdev= 2.71, samples=23547
   iops        : min=  202, max= 2648, avg=1308.01, stdev= 2.75, samples=23547
  lat (msec)   : 4=0.01%, 10=0.02%, 20=0.11%, 50=4.66%, 100=32.25%
  lat (msec)   : 250=52.77%, 500=9.10%, 750=0.77%, 1000=0.28%, 2000=0.05%
  cpu          : usr=0.00%, sys=0.04%, ctx=92967, majf=0, minf=53984
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued rwts: total=82238,0,0,0 short=0,0,0,0 dropped=0,0,0,0
     latency   : target=0, window=0, percentile=100.00%, depth=1

Run status group 0 (all jobs):
   READ: bw=1368MiB/s (1434MB/s), 1368MiB/s-1368MiB/s (1434MB/s-1434MB/s), io=80.3GiB (86.2GB), run=60130-60130msec

Disk stats (read/write):
  sda: ios=81909/617, merge=30/54, ticks=11925926/1508, in_queue=11927523, util=88.78%
root@debvsan:/home/vov#

Let's move a virtual machine disk from one storage to another.

root@pve1:~# qm move-disk 105 scsi0 vg_drbd0 

create full clone of drive scsi0 (vg_drbd1:vm-105-disk-0)
  Logical volume "vm-105-disk-2" created.
drive mirror is starting for drive-scsi0
drive-scsi0: transferred 6.0 MiB of 8.0 GiB (0.07%) in 0s
drive-scsi0: transferred 1013.0 MiB of 8.0 GiB (12.37%) in 1s
drive-scsi0: transferred 2.0 GiB of 8.0 GiB (24.40%) in 2s
drive-scsi0: transferred 2.9 GiB of 8.0 GiB (36.51%) in 3s
drive-scsi0: transferred 3.9 GiB of 8.0 GiB (48.63%) in 4s
drive-scsi0: transferred 4.9 GiB of 8.0 GiB (60.84%) in 5s
drive-scsi0: transferred 5.8 GiB of 8.0 GiB (72.97%) in 6s
drive-scsi0: transferred 6.8 GiB of 8.0 GiB (84.95%) in 7s
drive-scsi0: transferred 7.8 GiB of 8.0 GiB (97.08%) in 8s
drive-scsi0: transferred 8.0 GiB of 8.0 GiB (100.00%) in 9s, ready
all 'mirror' jobs are ready
drive-scsi0: Completing block job...
drive-scsi0: Completed successfully.
drive-scsi0: mirror-job finished
root@pve1:~#
