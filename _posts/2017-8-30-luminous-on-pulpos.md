---
layout: post
title: Installing Ceph v12.2 (Luminous) on Pulpos
tags: [Ceph, Storage, Provisioning]
---

In this post, we describe how we installed Ceph v12.2 (codename **Luminous**) on the [Pulpos]({{ site.baseurl }}{% post_url 2017-2-9-pulpos %}) cluster.<!-- more -->

In a surprising move, Red Hat released Ceph 12.2.0 on August 29, 2017, way ahead of their original schedule &mdash; Luminous was originally planned for release in Spring 2018! Luminous is the current [Long Term Stable (LTS) release](http://docs.ceph.com/docs/master/releases/) of Ceph, replacing both previous stable release *Kraken* (Ceph v11.2) and previous LTS release Jewel (Ceph v10.2). Luminous has [introduced many major](http://docs.ceph.com/docs/master/release-notes/#v12-2-0-luminous) changes from Kraken and Jewel; upgrade from earlier release is non-trivial. So we'll perform a clean re-installation of Luminous on Pulpos.

* Table of Contents
{:toc}

## Purging old Ceph installation
In order to start with a clean plate, we ran the following Bash script (`start-over.sh`) on the admin node (**pulpo-admin**) to [purge old Ceph packages, and erase old Ceph data and configuration](http://docs.ceph.com/docs/master/start/quick-ceph-deploy/#starting-over):
```shell
#!/bin/bash

cd ~/Pulpos

# Uninstall ceph-fuse on the client
ssh pulpo-dtn "umount /mnt/pulpos; yum erase -y ceph-fuse"

# http://docs.ceph.com/docs/master/start/quick-ceph-deploy/#starting-over
ceph-deploy purge pulpo-admin pulpo-dtn pulpo-mon01 pulpo-mds01 pulpo-osd01 pulpo-osd02 pulpo-osd03
ceph-deploy purgedata pulpo-admin pulpo-dtn pulpo-mon01 pulpo-mds01 pulpo-osd01 pulpo-osd02 pulpo-osd03
ceph-deploy forgetkeys

# More cleanups
ansible -m command -a "yum erase -y libcephfs2 python-cephfs librados2 python-rados librbd1 python-rbd librgw2 python-rgw" all
ansible -m shell -a "rm -rf /etc/systemd/system/ceph*.target.wants" all
yum erase -y ceph-deploy
rm -f ceph*
```
We then rebooted all the nodes.

## Installing Luminous packages
We use a simple [Ansible playbook](http://docs.ansible.com/ansible/latest/playbooks.html) to install Ceph v12.2 (Luminous) packages on all nodes in Pulpos, performing the following tasks:

1)  Add a [Yum repository for Luminous](http://docs.ceph.com/docs/master/install/get-packages/#rpm-packages) (`/etc/yum.repos.d/ceph.repo`) on all the nodes:
```ini
[Ceph]
name=Ceph packages for $basearch
baseurl=https://download.ceph.com/rpm-luminous/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=2

[Ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-luminous/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=2

[ceph-source]
name=Ceph source packages
baseurl=https://download.ceph.com/rpm-luminous/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=2
```

2) Install Ceph RPM packages on all the nodes;

3) Install `ceph-fuse` on the client node (**pulpo-dtn**);

4) Install `ceph-deploy` on the admin node (**pulpo-admin**).

Let's verify that Luminous is installed on all the nodes:
```shell
[root@pulpo-admin ~]# ansible -m command -a "ceph --version" all
pulpo-admin.local | SUCCESS | rc=0 >>
ceph version 12.2.0 (32ce2a3ae5239ee33d6150705cdb24d43bab910c) luminous (rc)

pulpo-mon01.local | SUCCESS | rc=0 >>
ceph version 12.2.0 (32ce2a3ae5239ee33d6150705cdb24d43bab910c) luminous (rc)

pulpo-dtn.local | SUCCESS | rc=0 >>
ceph version 12.2.0 (32ce2a3ae5239ee33d6150705cdb24d43bab910c) luminous (rc)

pulpo-mds01.local | SUCCESS | rc=0 >>
ceph version 12.2.0 (32ce2a3ae5239ee33d6150705cdb24d43bab910c) luminous (rc)

pulpo-osd01.local | SUCCESS | rc=0 >>
ceph version 12.2.0 (32ce2a3ae5239ee33d6150705cdb24d43bab910c) luminous (rc)

pulpo-osd02.local | SUCCESS | rc=0 >>
ceph version 12.2.0 (32ce2a3ae5239ee33d6150705cdb24d43bab910c) luminous (rc)

pulpo-osd03.local | SUCCESS | rc=0 >>
ceph version 12.2.0 (32ce2a3ae5239ee33d6150705cdb24d43bab910c) luminous (rc)
```

## Ceph-Deploy
We use [ceph-deploy](http://docs.ceph.com/docs/luminous/man/8/ceph-deploy/) to [deploy Luminous](http://docs.ceph.com/docs/luminous/start/quick-ceph-deploy/) on the Pulpos cluster. `ceph-deploy` is a easy and quick tool to set up and take down a Ceph cluster. It uses ssh to gain access to other Ceph nodes from the admin node (**pulpo-admin**), and then uses the underlying Python scripts to automate the manual process of Ceph installation on each node. One can also use a generic deployment system, such as Puppet, Chef or Ansible, to deploy Ceph. I am particularly interested in [ceph-ansible](http://docs.ceph.com/ceph-ansible/master/), the Ansible playbook for Ceph; and may try it in the near future.

1) We use the directory `/root/Pulpos` on the admin node to maintain the configuration files and keys
```shell
[root@pulpo-admin ~]# cd ~/Pulpos/
```

2) Create a cluster, with `pulpo-mon01` as the initial monitor node (We'll add 2 monitors shortly):
```shell
[root@pulpo-admin Pulpos]# ceph-deploy new pulpo-mon01
```
which generates `ceph.conf` & `ceph.mon.keyring` in the directory.

3) Append the following 2 lines to `ceph.conf`
```ini
public_network = 128.114.86.0/24
cluster_network = 192.168.40.0/24
```
The `public_network` is 10 Gb/s and `cluster_network` 40 Gb/s (see [Pulpos Networks]({{ site.baseurl }}{% post_url 2017-6-21-pulpos-networks %}))

4) Append the following 2 lines to `ceph.conf` (to allow deletion of pools):
```ini
[mon]
mon_allow_pool_delete = true
```

5) Deploy the initial monitor(s) and gather the keys:
```shell
[root@pulpo-admin Pulpos]# ceph-deploy mon create-initial
```
which generated `ceph.client.admin.keyring`, `ceph.bootstrap-osd.keyring`, `ceph.bootstrap-mds.keyring` & `ceph.bootstrap-rgw.keyringmon.keyring` in the directory.

6) Copy the configuration file and admin key to all the nodes
```shell
[root@pulpo-admin Pulpos]# ceph-deploy admin pulpo-admin pulpo-dtn pulpo-mon01 pulpo-mds01 pulpo-osd01 pulpo-osd02 pulpo-osd03
```
which copies `ceph.client.admin.keyring` & `ceph.conf` to the directory `/etc/ceph` on all the nodes.

7) Add 2 more monitors, on **pulpo-mds01** & **pulpo-admin**, respectively:
```shell
[root@pulpo-admin Pulpos]# ceph-deploy mon add pulpo-mds01
[root@pulpo-admin Pulpos]# ceph-deploy mon add pulpo-admin
```
It seems that we can only add one at a time.

8) Deploy a manager daemon on each of the monitor nodes:
```shell
[root@pulpo-admin Pulpos]# ceph-deploy mgr create pulpo-mon01 pulpo-mds01 pulpo-admin
```
<p class="note"><em>ceph-mgr</em> is new daemon introduced in Luminous, and is a required part of any Luminous deployment.</p>

## Adding OSDs
1) [List the disks](http://docs.ceph.com/docs/luminous/rados/deployment/ceph-deploy-osd/#list-disks) on the OSD nodes:
```shell
[root@pulpo-admin Pulpos]# ssh pulpo-osd01 ceph-disk list
```
or
```shell
[root@pulpo-admin Pulpos]# ssh pulpo-osd01 lsblk
```

2) We use the following Bash script (`zap-disks.sh`) to [zap the disks](http://docs.ceph.com/docs/luminous/rados/deployment/ceph-deploy-osd/#zap-disks) on the OSD nodes (**Caution**: device names *can* and *do* change!):
```shell
#!/bin/bash

for i in {1..3}
do
  for x in {a..l}
  do
     # zap the twelve 8TB SATA drives
     ceph-deploy disk zap pulpo-osd0${i}:sd${x}
  done
  for j in {0..1}
  do
     # zap the two NVMe SSDs
     ceph-deploy disk zap pulpo-osd0${i}:nvme${j}n1
  done
done
```

3) We then use the following Bash script (`create-osd.sh`) to [create OSDs](http://docs.ceph.com/docs/luminous/rados/deployment/ceph-deploy-osd/#prepare-osds) on the OSD nodes:
```shell
#!/bin/bash

### HDDs
for i in {1..3}
do
  for x in {a..l}
  do
     ceph-deploy osd prepare --bluestore --block-db /dev/nvme0n1 --block-wal /dev/nvme0n1 pulpo-osd0${i}:sd${x}
     ceph-deploy osd activate pulpo-osd0${i}:sd${x}1
     sleep 10
  done
done

### NVMe
for i in {1..3}
do
  ceph-deploy osd prepare --bluestore pulpo-osd0${i}:nvme1n1
  ceph-deploy osd activate pulpo-osd0${i}:nvme1n1p1
  sleep 10
done
```
The goals were to (on each of the OSD nodes):
* Create an OSD on each of the 8TB SATA HDDs, using the new [bluestore](http://docs.ceph.com/docs/luminous/rados/configuration/storage-devices/#bluestore) backend;
* Use a partition on the first NVMe SSD (`/dev/nvme1n0`) as the *WAL device* and another partition as the *DB device* for each of the OSDs on the HDDs;
* Create an OSD on the second NVMe SSD (`/dev/nvme1n1`), using the new **bluetore** backend.

Let's verify we have achieved our goals:
```shell
[root@pulpo-admin Pulpos]# ssh pulpo-osd01 ceph-disk list
[root@pulpo-admin Pulpos]# ssh pulpo-osd01 ceph-disk list
/dev/nvme0n1 :
 /dev/nvme0n1p1 ceph block.db, for /dev/sda1
 /dev/nvme0n1p10 ceph block.wal, for /dev/sde1
 /dev/nvme0n1p11 ceph block.db, for /dev/sdf1
 /dev/nvme0n1p12 ceph block.wal, for /dev/sdf1
 /dev/nvme0n1p13 ceph block.db, for /dev/sdg1
 /dev/nvme0n1p14 ceph block.wal, for /dev/sdg1
 /dev/nvme0n1p15 ceph block.db, for /dev/sdh1
 /dev/nvme0n1p16 ceph block.wal, for /dev/sdh1
 /dev/nvme0n1p17 ceph block.db, for /dev/sdi1
 /dev/nvme0n1p18 ceph block.wal, for /dev/sdi1
 /dev/nvme0n1p19 ceph block.db, for /dev/sdj1
 /dev/nvme0n1p2 ceph block.wal, for /dev/sda1
 /dev/nvme0n1p20 ceph block.wal, for /dev/sdj1
 /dev/nvme0n1p21 ceph block.db, for /dev/sdk1
 /dev/nvme0n1p22 ceph block.wal, for /dev/sdk1
 /dev/nvme0n1p23 ceph block.db, for /dev/sdl1
 /dev/nvme0n1p24 ceph block.wal, for /dev/sdl1
 /dev/nvme0n1p3 ceph block.db, for /dev/sdb1
 /dev/nvme0n1p4 ceph block.wal, for /dev/sdb1
 /dev/nvme0n1p5 ceph block.db, for /dev/sdc1
 /dev/nvme0n1p6 ceph block.wal, for /dev/sdc1
 /dev/nvme0n1p7 ceph block.db, for /dev/sdd1
 /dev/nvme0n1p8 ceph block.wal, for /dev/sdd1
 /dev/nvme0n1p9 ceph block.db, for /dev/sde1
/dev/nvme1n1 :
 /dev/nvme1n1p1 ceph data, active, cluster ceph, osd.36, block /dev/nvme1n1p2
 /dev/nvme1n1p2 ceph block, for /dev/nvme1n1p1
/dev/sda :
 /dev/sda1 ceph data, active, cluster ceph, osd.0, block /dev/sda2, block.db /dev/nvme0n1p1, block.wal /dev/nvme0n1p2
 /dev/sda2 ceph block, for /dev/sda1
/dev/sdb :
 /dev/sdb1 ceph data, active, cluster ceph, osd.1, block /dev/sdb2, block.db /dev/nvme0n1p3, block.wal /dev/nvme0n1p4
 /dev/sdb2 ceph block, for /dev/sdb1
/dev/sdc :
 /dev/sdc1 ceph data, active, cluster ceph, osd.2, block /dev/sdc2, block.db /dev/nvme0n1p5, block.wal /dev/nvme0n1p6
 /dev/sdc2 ceph block, for /dev/sdc1
/dev/sdd :
 /dev/sdd1 ceph data, active, cluster ceph, osd.3, block /dev/sdd2, block.db /dev/nvme0n1p7, block.wal /dev/nvme0n1p8
 /dev/sdd2 ceph block, for /dev/sdd1
/dev/sde :
 /dev/sde1 ceph data, active, cluster ceph, osd.4, block /dev/sde2, block.db /dev/nvme0n1p9, block.wal /dev/nvme0n1p10
 /dev/sde2 ceph block, for /dev/sde1
/dev/sdf :
 /dev/sdf1 ceph data, active, cluster ceph, osd.5, block /dev/sdf2, block.db /dev/nvme0n1p11, block.wal /dev/nvme0n1p12
 /dev/sdf2 ceph block, for /dev/sdf1
/dev/sdg :
 /dev/sdg1 ceph data, active, cluster ceph, osd.6, block /dev/sdg2, block.db /dev/nvme0n1p13, block.wal /dev/nvme0n1p14
 /dev/sdg2 ceph block, for /dev/sdg1
/dev/sdh :
 /dev/sdh1 ceph data, active, cluster ceph, osd.7, block /dev/sdh2, block.db /dev/nvme0n1p15, block.wal /dev/nvme0n1p16
 /dev/sdh2 ceph block, for /dev/sdh1
/dev/sdi :
 /dev/sdi1 ceph data, active, cluster ceph, osd.8, block /dev/sdi2, block.db /dev/nvme0n1p17, block.wal /dev/nvme0n1p18
 /dev/sdi2 ceph block, for /dev/sdi1
/dev/sdj :
 /dev/sdj1 ceph data, active, cluster ceph, osd.9, block /dev/sdj2, block.db /dev/nvme0n1p19, block.wal /dev/nvme0n1p20
 /dev/sdj2 ceph block, for /dev/sdj1
/dev/sdk :
 /dev/sdk1 ceph data, active, cluster ceph, osd.10, block /dev/sdk2, block.db /dev/nvme0n1p21, block.wal /dev/nvme0n1p22
 /dev/sdk2 ceph block, for /dev/sdk1
/dev/sdl :
 /dev/sdl1 ceph data, active, cluster ceph, osd.11, block /dev/sdl2, block.db /dev/nvme0n1p23, block.wal /dev/nvme0n1p24
 /dev/sdl2 ceph block, for /dev/sdl1
```
<p class="note">Each DB partition is only 1GB in size and each WAL partition is only 576MB. So there is plenty of space left on the first NVMe SSD (the total capacity is 1.1TB). We may create a new partition there to benchmark the NVMe SSD in the near future.</p>

Let's chech the cluster status:
```shell
[root@pulpo-admin Pulpos]# ceph -s
  cluster:
    id:     e18516bf-39cb-4670-9f13-88ccb7d19769
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum pulpo-admin,pulpo-mon01,pulpo-mds01
    mgr: pulpo-mon01(active), standbys: pulpo-mds01, pulpo-admin
    osd: 39 osds: 39 up, 39 in

  data:
    pools:   0 pools, 0 pgs
    objects: 0 objects, 0 bytes
    usage:   0 kB used, 0 kB / 0 kB avail
    pgs:
```
Further readings on *BlueStore*:
* [BlueStore, A New Storage Backend for Ceph, One Year In](https://www.slideshare.net/sageweil1/bluestore-a-new-storage-backend-for-ceph-one-year-in)
* [Bluestore: A new storage engine for Ceph](https://www.socallinuxexpo.org/sites/default/files/presentations/allen-samuels-Scale%2015x.pdf)

## CRUSH device class
one nice new feature introduced in Luminous is CRUSH [device class](http://docs.ceph.com/docs/luminous/rados/operations/crush-map/#devices).  
```shell
[root@pulpo-admin Pulpos]# ceph osd tree
ID CLASS WEIGHT    TYPE NAME            STATUS REWEIGHT PRI-AFF
-1       265.29291 root default
-3        88.43097     host pulpo-osd01
 0   hdd   7.27829         osd.0            up  1.00000 1.00000
 1   hdd   7.27829         osd.1            up  1.00000 1.00000
 2   hdd   7.27829         osd.2            up  1.00000 1.00000
 3   hdd   7.27829         osd.3            up  1.00000 1.00000
 4   hdd   7.27829         osd.4            up  1.00000 1.00000
 5   hdd   7.27829         osd.5            up  1.00000 1.00000
 6   hdd   7.27829         osd.6            up  1.00000 1.00000
 7   hdd   7.27829         osd.7            up  1.00000 1.00000
 8   hdd   7.27829         osd.8            up  1.00000 1.00000
 9   hdd   7.27829         osd.9            up  1.00000 1.00000
10   hdd   7.27829         osd.10           up  1.00000 1.00000
11   hdd   7.27829         osd.11           up  1.00000 1.00000
36  nvme   1.09149         osd.36           up  1.00000 1.00000
-5        88.43097     host pulpo-osd02
12   hdd   7.27829         osd.12           up  1.00000 1.00000
13   hdd   7.27829         osd.13           up  1.00000 1.00000
14   hdd   7.27829         osd.14           up  1.00000 1.00000
15   hdd   7.27829         osd.15           up  1.00000 1.00000
16   hdd   7.27829         osd.16           up  1.00000 1.00000
17   hdd   7.27829         osd.17           up  1.00000 1.00000
18   hdd   7.27829         osd.18           up  1.00000 1.00000
19   hdd   7.27829         osd.19           up  1.00000 1.00000
20   hdd   7.27829         osd.20           up  1.00000 1.00000
21   hdd   7.27829         osd.21           up  1.00000 1.00000
22   hdd   7.27829         osd.22           up  1.00000 1.00000
23   hdd   7.27829         osd.23           up  1.00000 1.00000
37  nvme   1.09149         osd.37           up  1.00000 1.00000
-7        88.43097     host pulpo-osd03
24   hdd   7.27829         osd.24           up  1.00000 1.00000
25   hdd   7.27829         osd.25           up  1.00000 1.00000
26   hdd   7.27829         osd.26           up  1.00000 1.00000
27   hdd   7.27829         osd.27           up  1.00000 1.00000
28   hdd   7.27829         osd.28           up  1.00000 1.00000
29   hdd   7.27829         osd.29           up  1.00000 1.00000
30   hdd   7.27829         osd.30           up  1.00000 1.00000
31   hdd   7.27829         osd.31           up  1.00000 1.00000
32   hdd   7.27829         osd.32           up  1.00000 1.00000
33   hdd   7.27829         osd.33           up  1.00000 1.00000
34   hdd   7.27829         osd.34           up  1.00000 1.00000
35   hdd   7.27829         osd.35           up  1.00000 1.00000
38  nvme   1.09149         osd.38           up  1.00000 1.00000
```
Luminous automatically associate the OSDs backed by HDDs with the `hdd` device class; and the OSDs backed by NVMes with the `nvme` device class. So we no longer need to manually modify CRUSH map (as in kraken and earlier Ceph releases) in order to place different pools on different OSDs!
```shell
[root@pulpo-admin Pulpos]# ceph osd crush class ls
[
    "hdd",
    "nvme"
]
```

## Adding an MDS
Add a Metadata server:
```shell
[root@pulpo-admin Pulpos]# ceph-deploy mds create pulpo-mds01
```
The goal is to create a [Ceph Filesystem (CephFS)](http://docs.ceph.com/docs/luminous/cephfs/), using 3 RADOS pools:
1. an Erasure Code data pool on the OSDs backed by HDDs
2. a replicated metadata pool on the OSDs backed by HDDs
3. a replicated pool on the OSDs backed by NVMes, as the cache tier of the Erasure Code data pool

## Creating an Erasure Code data pool
The default [erasure code](http://docs.ceph.com/docs/luminous/rados/operations/erasure-code/) profile sustains the loss of a single OSD.
```shell
[root@pulpo-admin Pulpos]# ceph osd erasure-code-profile ls
default
[root@pulpo-admin Pulpos]# ceph osd erasure-code-profile get default
k=2
m=1
plugin=jerasure
technique=reed_sol_van
```

Let's create a new erasure code profile `pulpo_ec`:
```shell
[root@pulpo-admin Pulpos]# ceph osd erasure-code-profile set pulpo_ec k=2 m=1 crush-device-class=hdd plugin=jerasure technique=reed_sol_van

[root@pulpo-admin Pulpos]# ceph osd erasure-code-profile ls
default
pulpo_ec

[root@pulpo-admin Pulpos]# ceph osd erasure-code-profile get pulpo_ec
crush-device-class=hdd
crush-failure-domain=host
crush-root=default
jerasure-per-chunk-alignment=false
k=2
m=1
plugin=jerasure
technique=reed_sol_van
w=8
```
The important parameter is `crush-device-class=hdd`, which set `hdd` as the device class for the profile. So a pool created with this profile will only use the OSDs backed by HDDs.

Create the *Erasure Code* data pool for CephFS, with the `pulpo_ec` profile:
```shell
[root@pulpo-admin Pulpos]# ceph osd pool create cephfs_data 1024 1024 erasure pulpo_ec
pool 'cephfs_data' created
```
which also generates a new CRUSH rule with the same name `cephfs_data`. We note in passing a terminology change: what's called `CRUSH ruleset` in Kraken and earlier is now called `CRUSH rule`; and the parameter `crush_ruleset` in the old ceph command is now replaced with `crush_rule`!

Let's check the CRUSH rule for pool `cephfs_data`:
```shell
[root@pulpo-admin Pulpos]# ceph osd pool get cephfs_data crush_rule
crush_rule: cephfs_data

[root@pulpo-admin Pulpos]# ceph osd dump | grep "^pool" | grep "crush_rule 1"
pool 1 'cephfs_data' erasure size 3 min_size 3 crush_rule 1 object_hash rjenkins pg_num 1024 pgp_num 1024 last_change 160 flags hashpspool stripe_width 8192

[root@pulpo-admin Pulpos]# ceph osd crush rule ls
replicated_rule
cephfs_data

[root@pulpo-admin Pulpos]# ceph osd crush rule dump cephfs_data
{
    "rule_id": 1,
    "rule_name": "cephfs_data",
    "ruleset": 1,
    "type": 3,
    "min_size": 3,
    "max_size": 3,
    "steps": [
        {
            "op": "set_chooseleaf_tries",
            "num": 5
        },
        {
            "op": "set_choose_tries",
            "num": 100
        },
        {
            "op": "take",
            "item": -2,
            "item_name": "default~hdd"
        },
        {
            "op": "chooseleaf_indep",
            "num": 0,
            "type": "host"
        },
        {
            "op": "emit"
        }
    ]
}
```

By default, erasure coded pools only work with uses like RGW that perform full object writes and appends. A new feature introduced in Luminous allows [partial writes for an erasure coded pool](http://docs.ceph.com/docs/luminous/rados/operations/erasure-code/#erasure-coding-with-overwrites), which may be enabled with a per-pool setting. This lets RBD and CephFS store their data in an erasure coded pool! Let's enable *overwrites* for the pool `cephfs_data`:
```shell
[root@pulpo-admin Pulpos]# ceph osd pool set cephfs_data allow_ec_overwrites true
set pool 1 allow_ec_overwrites to true
```

## Creating a replicated metadata pool
As stated earlier, the goal is to create a replicated metadata pool for CephFS on the OSDs backed by HDDs.

However, the default CRUSH rule for replicated pool, `replicated_rule`, will use all types of OSDs, no matter whether they are backed by HDDs, or by NVMes:
```shell
[root@pulpo-admin Pulpos]# ceph osd crush rule dump replicated_rule
{
    "rule_id": 0,
    "rule_name": "replicated_rule",
    "ruleset": 0,
    "type": 1,
    "min_size": 1,
    "max_size": 10,
    "steps": [
        {
            "op": "take",
            "item": -1,
            "item_name": "default"
        },
        {
            "op": "chooseleaf_firstn",
            "num": 0,
            "type": "host"
        },
        {
            "op": "emit"
        }
    ]
}
```

Here is the syntax for creating a new replication rule:
```shell
osd crush rule create-replicated <name>      create crush rule <name> for replicated
 <root> <type> {<class>}                      pool to start from <root>, replicate
                                              across buckets of type <type>, using a
                                              choose mode of <firstn|indep> (default
                                              firstn; indep best for erasure pools)
```

Let's create a new replication rule, `pulpo_hdd`, that targets the `hdd` device class (*root* is `default` and *bucket* type `host`):
```shell
[root@pulpo-admin Pulpos]# ceph osd crush rule create-replicated pulpo_hdd default host hdd
```

Check the rule:
```shell
[root@pulpo-admin Pulpos]# ceph osd crush rule dump pulpo_hdd
{
    "rule_id": 2,
    "rule_name": "pulpo_hdd",
    "ruleset": 2,
    "type": 1,
    "min_size": 1,
    "max_size": 10,
    "steps": [
        {
            "op": "take",
            "item": -2,
            "item_name": "default~hdd"
        },
        {
            "op": "chooseleaf_firstn",
            "num": 0,
            "type": "host"
        },
        {
            "op": "emit"
        }
    ]
}
```

We can now create the metadata pool using the CRUSH rule `pulpo_hdd`:
```shell
[root@pulpo-admin Pulpos]# ceph osd pool create cephfs_metadata 1024 1024 replicated pulpo_hdd
pool 'cephfs_metadata' created
```

Let’s verify that pool `cephfs_metadata` indeed uses rule `pulpo_hdd`:
```shell
[root@pulpo-admin Pulpos]# ceph osd pool get cephfs_metadata crush_rule
crush_rule: pulpo_hdd
```

We note in passing that because we have enabled overwrites for the Erasure Code data pool, we could create a CephFS at this point:
```shell
# ceph fs new pulpos cephfs_metadata cephfs_data
```
which is a marked improvement over [Kraken]({{ site.baseurl }}{% post_url 2017-8-21-kraken-on-pulpos %}). We, however, will delay the creation of CephFS until we've added a *cache tier* to the data pool.

## Adding Cache Tiering to the data pool
The goal is to create a replicated pool on the OSDs backed by the NVMes, as the [cache tier](http://docs.ceph.com/docs/luminous/rados/operations/cache-tiering/) of the Erasure Code data pool for the CephFS.

1) Create a new replication rule, `pulpo_nvme`, that targets the `nvme` device class (*root* is `default` and *bucket* type `host`):
```shell
[root@pulpo-admin Pulpos]# ceph osd crush rule create-replicated pulpo_nvme default host nvme
```

Check the rule:
```shell
[root@pulpo-admin ~]# ceph osd crush rule ls
replicated_rule
cephfs_data
pulpo_hdd
pulpo_nvme

[root@pulpo-admin ~]# ceph osd crush rule dump pulpo_nvme
{
    "rule_id": 3,
    "rule_name": "pulpo_nvme",
    "ruleset": 3,
    "type": 1,
    "min_size": 1,
    "max_size": 10,
    "steps": [
        {
            "op": "take",
            "item": -12,
            "item_name": "default~nvme"
        },
        {
            "op": "chooseleaf_firstn",
            "num": 0,
            "type": "host"
        },
        {
            "op": "emit"
        }
    ]
}
```

2) Create the replicated cache pool:
```shell
[root@pulpo-admin Pulpos]# ceph osd pool create cephfs_cache 128 128 replicated pulpo_nvme
pool 'cephfs_cache' created
```

By default, the replication size is 3. But 2 is sufficient for the cache pool.
```shell
[root@pulpo-admin Pulpos]# ceph osd pool get cephfs_cache size
size: 3
[root@pulpo-admin Pulpos]# ceph osd pool set cephfs_cache size 2
set pool 3 size to 2
```

One can list all the [placement groups](http://docs.ceph.com/docs/master/rados/operations/placement-groups/) of the cache pool (pool 3):
```shell
# ceph pg dump | grep '^3\.'
```
and get the placement group map for a particular placement group:
```shell
#  ceph pg map 3.5c
osdmap e173 pg 3.5c (3.5c) -> up [36,38] acting [36,38]
```

3) Create the cache tier:
```shell
[root@pulpo-admin Pulpos]# ceph osd tier add cephfs_data cephfs_cache
pool 'cephfs_cache' is now (or already was) a tier of 'cephfs_data'

[root@pulpo-admin Pulpos]# ceph osd tier cache-mode cephfs_cache writeback
set cache-mode for pool 'cephfs_cache' to writeback
[root@pulpo-admin Pulpos]# ceph osd tier set-overlay cephfs_data cephfs_cache
overlay for 'cephfs_data' is now (or already was) 'cephfs_cache'
[root@pulpo-admin Pulpos]# ceph osd pool set cephfs_cache hit_set_type bloom
set pool 3 hit_set_type to bloom
[root@pulpo-admin Pulpos]# ceph osd pool set cephfs_cache hit_set_count 12
set pool 3 hit_set_count to 12
[root@pulpo-admin Pulpos]# ceph osd pool set cephfs_cache hit_set_period 14400
set pool 3 hit_set_period to 14400
```

## Creating CephFS
Now we are ready to create the Ceph Filesystem:
```shell
[root@pulpo-admin Pulpos]# ceph fs new pulpos cephfs_metadata cephfs_data
new fs with metadata pool 2 and data pool 1
```

## A serious bug!
Unfortunately, there is a serious bug lurking in the current version of Luminous (v12.2.0)! If we check the status of the Ceph cluster, we are told that all *placement groups* are both inactive and unclean!
```shell
[root@pulpo-admin Pulpos]# ceph -s
  cluster:
    id:     e18516bf-39cb-4670-9f13-88ccb7d19769
    health: HEALTH_WARN
            Reduced data availability: 2176 pgs inactive
            Degraded data redundancy: 2176 pgs unclean

  services:
    mon: 3 daemons, quorum pulpo-admin,pulpo-mon01,pulpo-mds01
    mgr: pulpo-mon01(active), standbys: pulpo-mds01, pulpo-admin
    mds: pulpos-1/1/1 up  {0=pulpo-mds01=up:active}
    osd: 39 osds: 39 up, 39 in

  data:
    pools:   3 pools, 2176 pgs
    objects: 0 objects, 0 bytes
    usage:   0 kB used, 0 kB / 0 kB avail
    pgs:     100.000% pgs unknown
             2176 unknown
```

Same with `ceph health`:
```shell
[root@pulpo-admin Pulpos]# ceph health detail
HEALTH_WARN Reduced data availability: 2176 pgs inactive; Degraded data redundancy: 2176 pgs unclean
PG_AVAILABILITY Reduced data availability: 2176 pgs inactive
    pg 1.31d is stuck inactive for 65861.646865, current state unknown, last acting []
    pg 1.31e is stuck inactive for 65861.646865, current state unknown, last acting []
    pg 1.31f is stuck inactive for 65861.646865, current state unknown, last acting []
...
PG_DEGRADED Degraded data redundancy: 2176 pgs unclean
    pg 1.31d is stuck unclean for 65861.646865, current state unknown, last acting []
    pg 1.31e is stuck unclean for 65861.646865, current state unknown, last acting []
    pg 1.31f is stuck unclean for 65861.646865, current state unknown, last acting []
...
```

However, if we query any *placement group* that is supposedly inactive and unclean, we find it to be actually both *active* and *clean*. Take, for example, pg 1.31d:
```shell
[root@pulpo-admin Pulpos]# ceph pg 1.31d query
{
    "state": "active+clean",
    "snap_trimq": "[]",
    "epoch": 180,
    "up": [
        17,
        25,
        6
    ],
    "acting": [
        17,
        25,
        6
    ],
    ...
}
```
We hope this bug will be fixed soon!

## Mounting CephFS on clients
There are 2 ways to [mount CephFS](http://docs.ceph.com/docs/master/cephfs/best-practices/) on a client: using either the kernel CephFS driver, or ceph-fuse. The fuse client is the easiest way to get up to date code, while the kernel client will often give better performance.

On a client, e.g., *pulpo-dtn*, create the mount point:
```shell
[root@pulpo-dtn ~]# mkdir /mnt/pulpos
```

### Kernel CephFS driver
The Ceph Storage Cluster runs with authentication turned on by default. We need a file containing the secret key (i.e., not the keyring itself).

0) [Create the secret file](http://docs.ceph.com/docs/luminous/start/quick-cephfs/#create-a-secret-file), and save it as `/etc/ceph/admin.secret`.

1) We can use the `mount` command to [mount CephFS with kernel driver](http://docs.ceph.com/docs/luminous/cephfs/kernel/)
```shell
[root@pulpo-dtn ~]# mount -t ceph 128.114.86.4:6789:/ /mnt/pulpos -o name=admin,secretfile=/etc/ceph/admin.secret
```
or more redundantly
```shell
[root@pulpo-dtn ~]# mount -t ceph 128.114.86.4:6789,128.114.86.5:6789,128.114.86.2:6789:/ /mnt/pulpos -o name=admin,secretfile=/etc/ceph/admin.secret
```
With this method, we need to specify the monitor host IP address(es) and port number(s).

2) Or we can use the simple helper [mount.ceph](http://docs.ceph.com/docs/luminous/man/8/mount.ceph/), which resolve monitor hostname(s) into IP address(es):
```shell
[root@pulpo-dtn ~]# mount.ceph pulpo-mon01:/ /mnt/pulpos -o name=admin,secretfile=/etc/ceph/admin.secret
```
or more redundantly
```shell
[root@pulpo-dtn ~]# mount.ceph pulpo-mon01,pulpo-mds01,pulpo-admin:/ /mnt/pulpos -o name=admin,secretfile=/etc/ceph/admin.secret
```

3) To [mount CephFS automatically on startup](http://docs.ceph.com/docs/luminous/cephfs/fstab/#kernel-driver), we can add the following to `/etc/fstab`:
```conf
128.114.86.4:6789,128.114.86.5:6789,128.114.86.2:6789:/  /mnt/pulpos  ceph  name=admin,secretfile=/etc/ceph/admin.secret,noatime,_netdev  0  2
```

And here is another bug in current version of Luminous (v12.2.0): when CephFS is mounted, the mount point doesn't show up in the output of `df`; and although we can list the mount point specifically with `df -h /mnt/pulpos`, the size of the filesystem is reported as 0!
```
[root@pulpo-dtn ~]# df -h /mnt/pulpos
Filesystem                                Size  Used Avail Use% Mounted on
128.114.86.4,128.114.86.5,128.114.86.2:/     0     0     0    - /mnt/pulpos
```
Nonetheless, we can read and write to the CephFS just fine!

### ceph-fuse
Make sure the `ceph-fuse` package is installed. We've already installed the package on *pulpo-dtn*, using *Ansible*.

`cephx` authentication is on by default. Ensure that the client host has a copy of the Ceph configuration file and a keyring with CAPS for the Ceph metadata server. *pulpo-dtn* already has a copy of these 2 files. **NOTE** [ceph-fuse](http://docs.ceph.com/docs/luminous/cephfs/fuse/) uses the keyring rather than a secret file for authentication!

Then we can use the `ceph-fuse` command to mount the CephFS as a FUSE ([Filesystem in Userspace](https://en.wikipedia.org/wiki/Filesystem_in_Userspace)):
on pulpo-dtn:
```shell
[root@pulpo-dtn ~]# ceph-fuse -m 128.114.86.4:6789 /mnt/pulpos
ceph-fuse[3699]: starting ceph client2017-09-05 11:13:45.398103 7f8891150040 -1 init, newargv = 0x7f889b82ee40 newargc=9

ceph-fuse[3699]: starting fuse
```
or more redundantly:
```shell
[root@pulpo-dtn ~]# ceph-fuse -m pulpo-mon01:6789,pulpo-mds01:6789,pulpo-admin:6789 /mnt/pulpos
```

There are 2 options to automate mounting ceph-fuse: `fstab` or `systemd`.

1) We can add the following to `/etc/fstab` (see [http://docs.ceph.com/docs/luminous/cephfs/fstab/#fuse](http://docs.ceph.com/docs/luminous/cephfs/fstab/#fuse)):
```conf
id=admin  /mnt/pulpos  fuse.ceph  defaults,_netdev  0  2
```

2) `ceph-fuse@.service` and `ceph-fuse.target` systemd units are available. To mount CephFS as a FUSE on `/mnt/pulpos`, using *systemctl*:
```shell
[root@pulpo-dtn ~]# systemctl start ceph-fuse@/mnt/pulpos.service
```
To create a persistent mount point:
```shell
[root@pulpo-dtn ~]# systemctl enable ceph-fuse.target
Created symlink from /etc/systemd/system/remote-fs.target.wants/ceph-fuse.target to /usr/lib/systemd/system/ceph-fuse.target.
Created symlink from /etc/systemd/system/ceph.target.wants/ceph-fuse.target to /usr/lib/systemd/system/ceph-fuse.target.

[root@pulpo-dtn ~]# systemctl enable ceph-fuse@-mnt-pulpos
Created symlink from /etc/systemd/system/ceph-fuse.target.wants/ceph-fuse@-mnt-pulpos.service to /usr/lib/systemd/system/ceph-fuse@.service.
```
**NOTE** here the command must be `systemctl enable ceph-fuse@-mnt-pulpos`. If we run `systemctl enable ceph-fuse@/mnt/pulpos` instead, we'll get an error "Failed to execute operation: Unit name pulpos is not valid." However, when starting the service, we can run either `systemctl start ceph-fuse@/mnt/pulpos` or `systemctl start ceph-fuse@-mnt-pulpos`!

Lastly, we note the same bug in current version of Luminous (v12.2.0): when CephFS is mounted using *ceph-fuse*, the mount point doesn't show up in the output of `df`; and although we can list the mount point specifically with `df -h /mnt/pulpos`, the size of the filesystem is reported as 0!
```
[root@pulpo-dtn ~]# df -h /mnt/pulpos
Filesystem      Size  Used Avail Use% Mounted on
ceph-fuse          0     0     0    - /mnt/pulpos
```
