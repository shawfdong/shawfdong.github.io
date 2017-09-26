---
layout: post
title: Installing Ceph v11.2 (Kraken) on Pulpos
tags: [Ceph, Storage, Provisioning]
---

In this post, we describe how we installed Ceph v11.2 (codename **Kraken**) on the [Pulpos]({{ site.baseurl }}{% post_url 2017-2-9-pulpos %}) cluster.<!-- more -->

As of this writing, the current stable [release of Ceph](http://docs.ceph.com/docs/master/releases/) is Kraken (Ceph v11.2). Kraken, however, is not an LTS (Long Term Stable) release. So Kraken will only be maintained with bugfixes and backports until the next stable release, Luminous, is completed in the Spring of 2017. Every other stable release of Ceph is a LTS (Long Term Stable) and will receive updates until two LTS are published. The current Ceph LTS is Jewel (Ceph v10.2); and the next stable release, Luminous (Ceph v12.2), will be an LTS as well.

* Table of Contents
{:toc}

## Purging old Ceph installation
In order to start with a clean plate, we ran the following Bash script (`start-over.sh`) on the admin node (**pulpo-admin**) to [purge old Ceph packages, and erase old Ceph data and configuration](http://docs.ceph.com/docs/master/start/quick-ceph-deploy/#starting-over):
```shell
#!/bin/bash

cd ~/Pulpos

# Uninstall ceph-fuse on the client
ssh pulpo-dtn yum erase -y ceph-fuse

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

## Installing Kraken packages
We use a simple [Ansible playbook](http://docs.ansible.com/ansible/latest/playbooks.html) to install Ceph v11.2 (Kraken) packages on all nodes in Pulpos, performing the following tasks:

1)  Add a [Yum repository for Kraken](http://docs.ceph.com/docs/master/install/get-packages/#rpm-packages) (`/etc/yum.repos.d/ceph.repo`) on all the nodes:
```ini
[Ceph]
name=Ceph packages for $basearch
baseurl=https://download.ceph.com/rpm-kraken/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=2

[Ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-kraken/el7/noarch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=2

[ceph-source]
name=Ceph source packages
baseurl=https://download.ceph.com/rpm-kraken/el7/SRPMS
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=2
```

2) Install Ceph RPM packages on all the nodes;

3) Install `ceph-fuse` on the client node (**pulpo-dtn**);

4) Install `ceph-deploy` on the admin node (**pulpo-admin**).

Let's verify that Kraken is installed on all the nodes:
```shell
[root@pulpo-admin ~]# ansible -m command -a "ceph --version" all
pulpo-admin.local | SUCCESS | rc=0 >>
ceph version 11.2.1 (e0354f9d3b1eea1d75a7dd487ba8098311be38a7)

pulpo-mon01.local | SUCCESS | rc=0 >>
ceph version 11.2.1 (e0354f9d3b1eea1d75a7dd487ba8098311be38a7)

pulpo-dtn.local | SUCCESS | rc=0 >>
ceph version 11.2.1 (e0354f9d3b1eea1d75a7dd487ba8098311be38a7)

pulpo-osd01.local | SUCCESS | rc=0 >>
ceph version 11.2.1 (e0354f9d3b1eea1d75a7dd487ba8098311be38a7)

pulpo-mds01.local | SUCCESS | rc=0 >>
ceph version 11.2.1 (e0354f9d3b1eea1d75a7dd487ba8098311be38a7)

pulpo-osd02.local | SUCCESS | rc=0 >>
ceph version 11.2.1 (e0354f9d3b1eea1d75a7dd487ba8098311be38a7)

pulpo-osd03.local | SUCCESS | rc=0 >>
ceph version 11.2.1 (e0354f9d3b1eea1d75a7dd487ba8098311be38a7)
```

## ceph-deploy
We use [ceph-deploy](http://docs.ceph.com/docs/kraken/man/8/ceph-deploy/) to [deploy Kraken](http://docs.ceph.com/docs/kraken/start/quick-ceph-deploy/) on the Pulpos cluster. `ceph-deploy` is a easy and quick tool to set up and take down a Ceph cluster. It uses ssh to gain access to other Ceph nodes from the admin node (**pulpo-admin**), and then uses the underlying Python scripts to automate the manual process of Ceph installation on each node. One can also use a generic deployment system, such as Puppet, Chef or Ansible, to deploy Ceph. I am particularly interested in [ceph-ansible](http://docs.ceph.com/ceph-ansible/master/), the Ansible playbook for Ceph; and may try it in the near future.

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

## Adding OSDs
1) [List the disks](http://docs.ceph.com/docs/kraken/rados/deployment/ceph-deploy-osd/#list-disks) on the OSD nodes:
```shell
[root@pulpo-admin Pulpos]# ssh pulpo-osd01 ceph-disk list
```
or
```shell
[root@pulpo-admin Pulpos]# ssh pulpo-osd01 lsblk
```

2) We use the following Bash script (`zap-disks.sh`) to [zap the disks](http://docs.ceph.com/docs/kraken/rados/deployment/ceph-deploy-osd/#zap-disks) on the OSD nodes (**Caution**: device names *can* and *do* change!):
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

3) We then use the following Bash script (`create-osd.sh`) to [create OSDs](http://docs.ceph.com/docs/kraken/rados/deployment/ceph-deploy-osd/#prepare-osds) on the OSD nodes:
```shell
#!/bin/bash

### HDDs
for i in {1..3}
do
  j=1
  for x in {a..l}
  do
     ceph-deploy osd prepare pulpo-osd0${i}:sd${x}:/dev/nvme0n1
     ceph-deploy osd activate pulpo-osd0${i}:sd${x}1:/dev/nvme0n1p${j}
     j=$[j+1]
     sleep 10
  done
done

### NVMe
for i in {1..3}
do
  ceph-deploy osd prepare pulpo-osd0${i}:nvme1n1
  ceph-deploy osd activate pulpo-osd0${i}:nvme1n1p1
  sleep 10
done
```
The goals were to (on each of the OSD nodes):
* Create an OSD on each of the 8TB SATA HDDs, using the default **filestore** backend;
* Use a partition on the first NVMe SSD (`/dev/nvme1n0`) as the journal for each of the OSDs on the HDDs;
* Create an OSD on the second NVMe SSD (`/dev/nvme1n1`), using the default **filestore** backend.

Let's see if we have achieved our goals:
```shell
[root@pulpo-admin Pulpos]# ssh pulpo-osd01 ceph-disk list
/dev/nvme0n1 :
 /dev/nvme0n1p1 ceph journal, for /dev/sda1
 /dev/nvme0n1p10 ceph journal, for /dev/sdj1
 /dev/nvme0n1p11 ceph journal, for /dev/sdk1
 /dev/nvme0n1p12 ceph journal, for /dev/sdl1
 /dev/nvme0n1p2 ceph journal, for /dev/sdb1
 /dev/nvme0n1p3 ceph journal, for /dev/sdc1
 /dev/nvme0n1p4 ceph journal, for /dev/sdd1
 /dev/nvme0n1p5 ceph journal, for /dev/sde1
 /dev/nvme0n1p6 ceph journal, for /dev/sdf1
 /dev/nvme0n1p7 ceph journal, for /dev/sdg1
 /dev/nvme0n1p8 ceph journal, for /dev/sdh1
 /dev/nvme0n1p9 ceph journal, for /dev/sdi1
/dev/nvme1n1 :
 /dev/nvme1n1p1 ceph data, active, cluster ceph, osd.36, journal /dev/nvme1n1p2
 /dev/nvme1n1p2 ceph journal, for /dev/nvme1n1p1
/dev/sda :
 /dev/sda1 ceph data, active, cluster ceph, osd.0, journal /dev/nvme0n1p1
/dev/sdb :
 /dev/sdb1 ceph data, active, cluster ceph, osd.1, journal /dev/nvme0n1p2
/dev/sdc :
 /dev/sdc1 ceph data, active, cluster ceph, osd.2, journal /dev/nvme0n1p3
/dev/sdd :
 /dev/sdd1 ceph data, active, cluster ceph, osd.3, journal /dev/nvme0n1p4
/dev/sde :
 /dev/sde1 ceph data, active, cluster ceph, osd.4, journal /dev/nvme0n1p5
/dev/sdf :
 /dev/sdf1 ceph data, active, cluster ceph, osd.5, journal /dev/nvme0n1p6
/dev/sdg :
 /dev/sdg1 ceph data, active, cluster ceph, osd.6, journal /dev/nvme0n1p7
/dev/sdh :
 /dev/sdh1 ceph data, active, cluster ceph, osd.7, journal /dev/nvme0n1p8
/dev/sdi :
 /dev/sdi1 ceph data, active, cluster ceph, osd.8, journal /dev/nvme0n1p9
/dev/sdj :
 /dev/sdj1 ceph data, active, cluster ceph, osd.9, journal /dev/nvme0n1p10
/dev/sdk :
 /dev/sdk1 ceph data, active, cluster ceph, osd.10, journal /dev/nvme0n1p11
/dev/sdl :
 /dev/sdl1 ceph data, active, cluster ceph, osd.11, journal /dev/nvme0n1p12
```
It looks about right!

<p class="note">Each journal partition is only 5GB in size. So there is plenty of space left on the first NVMe SSD (the total capacity is 1.1TB). We may create a new partition there to benchmark the NVMe SSD in the near future.</p>

## Changing pg_num
Let's check the health of the Ceph cluster:
```shell
[root@pulpo-admin Pulpos]# ceph -s
    cluster ba892c66-7666-4957-b096-92a92bb87282
     health HEALTH_WARN
            too few PGs per OSD (4 < min 30)
     monmap e4: 3 mons at {pulpo-admin=128.114.86.2:6789/0,pulpo-mds01=128.114.86.5:6789/0,pulpo-mon01=128.114.86.4:6789/0}
            election epoch 12, quorum 0,1,2 pulpo-admin,pulpo-mon01,pulpo-mds01
        mgr active: pulpo-mon01 standbys: pulpo-mds01, pulpo-admin
     osdmap e182: 39 osds: 39 up, 39 in
            flags sortbitwise,require_jewel_osds,require_kraken_osds
      pgmap v793: 64 pgs, 1 pools, 0 bytes data, 0 objects
            1423 MB used, 265 TB / 265 TB avail
                  64 active+clean
```

At this point, only one default pool, `rbd`, exists. But the default `pg_num` is too small!
```shell
[root@pulpo-admin Pulpos]# ceph osd lspools
0 rbd,
[root@pulpo-admin Pulpos]# ceph osd pool get rbd pg_num
pg_num: 64
[root@pulpo-admin Pulpos]# ceph osd pool get rbd pgp_num
pgp_num: 64
```

The [recommended pg_num](http://docs.ceph.com/docs/kraken/rados/operations/placement-groups/) for a Ceph cluster of Pulpos' size is 1024. Let's change both `pg_num` and `pgp_num`:
```shell
[root@pulpo-admin Pulpos]# ceph osd pool set rbd pg_num 1024
set pool 0 pg_num to 1024
[root@pulpo-admin Pulpos]# ceph osd pool set rbd pgp_num 1024
set pool 0 pgp_num to 1024
```

Wait for a couple of minutes; then check the health again:
```shell
[root@pulpo-admin Pulpos]# ceph -s
    cluster ba892c66-7666-4957-b096-92a92bb87282
     health HEALTH_OK
     monmap e4: 3 mons at {pulpo-admin=128.114.86.2:6789/0,pulpo-mds01=128.114.86.5:6789/0,pulpo-mon01=128.114.86.4:6789/0}
            election epoch 12, quorum 0,1,2 pulpo-admin,pulpo-mon01,pulpo-mds01
        mgr active: pulpo-mon01 standbys: pulpo-mds01, pulpo-admin
     osdmap e190: 39 osds: 39 up, 39 in
            flags sortbitwise,require_jewel_osds,require_kraken_osds
      pgmap v861: 1024 pgs, 1 pools, 0 bytes data, 0 objects
            1471 MB used, 265 TB / 265 TB avail
                1024 active+clean
```
Healthy now!

## Modifying CRUSH map
Unlike [Luminous]({{ site.baseurl }}{% post_url 2017-8-30-luminous-on-pulpos %}), Kraken has no concept of `crsuh device class`, so it doesn't differentiate between OSDs backed by HDDs and an OSD backed by NVMes.
```shell
[root@pulpo-admin Pulpos]# ceph osd tree
ID WEIGHT    TYPE NAME            UP/DOWN REWEIGHT PRIMARY-AFFINITY
-1 265.17267 root default
-2  88.39088     host pulpo-osd01
 0   7.27539         osd.0             up  1.00000          1.00000
 1   7.27539         osd.1             up  1.00000          1.00000
 2   7.27539         osd.2             up  1.00000          1.00000
 3   7.27539         osd.3             up  1.00000          1.00000
 4   7.27539         osd.4             up  1.00000          1.00000
 5   7.27539         osd.5             up  1.00000          1.00000
 6   7.27539         osd.6             up  1.00000          1.00000
 7   7.27539         osd.7             up  1.00000          1.00000
 8   7.27539         osd.8             up  1.00000          1.00000
 9   7.27539         osd.9             up  1.00000          1.00000
10   7.27539         osd.10            up  1.00000          1.00000
11   7.27539         osd.11            up  1.00000          1.00000
36   1.08620         osd.36            up  1.00000          1.00000
-3  88.39088     host pulpo-osd02
12   7.27539         osd.12            up  1.00000          1.00000
13   7.27539         osd.13            up  1.00000          1.00000
14   7.27539         osd.14            up  1.00000          1.00000
15   7.27539         osd.15            up  1.00000          1.00000
16   7.27539         osd.16            up  1.00000          1.00000
17   7.27539         osd.17            up  1.00000          1.00000
18   7.27539         osd.18            up  1.00000          1.00000
19   7.27539         osd.19            up  1.00000          1.00000
20   7.27539         osd.20            up  1.00000          1.00000
21   7.27539         osd.21            up  1.00000          1.00000
22   7.27539         osd.22            up  1.00000          1.00000
23   7.27539         osd.23            up  1.00000          1.00000
37   1.08620         osd.37            up  1.00000          1.00000
-4  88.39088     host pulpo-osd03
24   7.27539         osd.24            up  1.00000          1.00000
25   7.27539         osd.25            up  1.00000          1.00000
26   7.27539         osd.26            up  1.00000          1.00000
27   7.27539         osd.27            up  1.00000          1.00000
28   7.27539         osd.28            up  1.00000          1.00000
29   7.27539         osd.29            up  1.00000          1.00000
30   7.27539         osd.30            up  1.00000          1.00000
31   7.27539         osd.31            up  1.00000          1.00000
32   7.27539         osd.32            up  1.00000          1.00000
33   7.27539         osd.33            up  1.00000          1.00000
34   7.27539         osd.34            up  1.00000          1.00000
35   7.27539         osd.35            up  1.00000          1.00000
38   1.08620         osd.38            up  1.00000          1.00000
```
Thus, by default, the [CRUSH](http://docs.ceph.com/docs/kraken/rados/operations/crush-map/) algorithm will pseudo-randomly store data of a pool in OSDs across the cluster, including both OSDs backed by HDDs and OSDs backed by NVMes! Because of significant difference in speed between HDDs and NVMes, this will result in imbalance. A better way is to [place different pools on different OSDs](http://docs.ceph.com/docs/kraken/rados/operations/crush-map/#placing-different-pools-on-different-osds). In order to do that with kraken, one must manually [edit the CRUSH map](http://docs.ceph.com/docs/kraken/rados/operations/crush-map/#editing-a-crush-map).  

1) Get the current CRUSH map in compiled form (`crushmap-0.bin`):
```shell
[root@pulpo-admin Pulpos]# ceph osd getcrushmap -o crushmap-0.bin
got crush map from osdmap epoch 191
```

2) Decompile the CRUSH map to a text file (`crushmap-0.txt`):
```shell
[root@pulpo-admin Pulpos]# crushtool -d crushmap-0.bin -o crushmap-0.txt
```

Here is the decompiled CRUSH map (`crushmap-0.txt`):
```
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable chooseleaf_vary_r 1
tunable straw_calc_version 1

# devices
device 0 osd.0
device 1 osd.1
device 2 osd.2
device 3 osd.3
device 4 osd.4
device 5 osd.5
device 6 osd.6
device 7 osd.7
device 8 osd.8
device 9 osd.9
device 10 osd.10
device 11 osd.11
device 12 osd.12
device 13 osd.13
device 14 osd.14
device 15 osd.15
device 16 osd.16
device 17 osd.17
device 18 osd.18
device 19 osd.19
device 20 osd.20
device 21 osd.21
device 22 osd.22
device 23 osd.23
device 24 osd.24
device 25 osd.25
device 26 osd.26
device 27 osd.27
device 28 osd.28
device 29 osd.29
device 30 osd.30
device 31 osd.31
device 32 osd.32
device 33 osd.33
device 34 osd.34
device 35 osd.35
device 36 osd.36
device 37 osd.37
device 38 osd.38

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root

# buckets
host pulpo-osd01 {
        id -2           # do not change unnecessarily
        # weight 88.391
        alg straw
        hash 0  # rjenkins1
        item osd.0 weight 7.275
        item osd.1 weight 7.275
        item osd.2 weight 7.275
        item osd.3 weight 7.275
        item osd.4 weight 7.275
        item osd.5 weight 7.275
        item osd.6 weight 7.275
        item osd.7 weight 7.275
        item osd.8 weight 7.275
        item osd.9 weight 7.275
        item osd.10 weight 7.275
        item osd.11 weight 7.275
        item osd.36 weight 1.086
}
host pulpo-osd02 {
        id -3           # do not change unnecessarily
        # weight 88.391
        alg straw
        hash 0  # rjenkins1
        item osd.12 weight 7.275
        item osd.13 weight 7.275
        item osd.14 weight 7.275
        item osd.15 weight 7.275
        item osd.16 weight 7.275
        item osd.17 weight 7.275
        item osd.18 weight 7.275
        item osd.19 weight 7.275
        item osd.20 weight 7.275
        item osd.21 weight 7.275
        item osd.22 weight 7.275
        item osd.23 weight 7.275
        item osd.37 weight 1.086
}
host pulpo-osd03 {
        id -4           # do not change unnecessarily
        # weight 88.391
        alg straw
        hash 0  # rjenkins1
        item osd.24 weight 7.275
        item osd.25 weight 7.275
        item osd.26 weight 7.275
        item osd.27 weight 7.275
        item osd.28 weight 7.275
        item osd.29 weight 7.275
        item osd.30 weight 7.275
        item osd.31 weight 7.275
        item osd.32 weight 7.275
        item osd.33 weight 7.275
        item osd.34 weight 7.275
        item osd.35 weight 7.275
        item osd.38 weight 1.086
}
root default {
        id -1           # do not change unnecessarily
        # weight 265.173
        alg straw
        hash 0  # rjenkins1
        item pulpo-osd01 weight 88.391
        item pulpo-osd02 weight 88.391
        item pulpo-osd03 weight 88.391
}

# rules
rule replicated_ruleset {
        ruleset 0
        type replicated
        min_size 1
        max_size 10
        step take default
        step chooseleaf firstn 0 type host
        step emit
}

# end crush map
```

We can see that at this point, only default CRUSH ruleset `0` exists. We can verify that the default pool `rbd` uses the default ruleset `0`:  
```shell
[root@pulpo-admin Pulpos]# ceph osd pool get rbd crush_ruleset
crush_ruleset: 0
```

3) Edit the CRUSH map. Here is the new CRUSH map in decompiled form (`crushmap-2.txt`):
```
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable chooseleaf_vary_r 1
tunable straw_calc_version 1

# devices
device 0 osd.0
device 1 osd.1
device 2 osd.2
device 3 osd.3
device 4 osd.4
device 5 osd.5
device 6 osd.6
device 7 osd.7
device 8 osd.8
device 9 osd.9
device 10 osd.10
device 11 osd.11
device 12 osd.12
device 13 osd.13
device 14 osd.14
device 15 osd.15
device 16 osd.16
device 17 osd.17
device 18 osd.18
device 19 osd.19
device 20 osd.20
device 21 osd.21
device 22 osd.22
device 23 osd.23
device 24 osd.24
device 25 osd.25
device 26 osd.26
device 27 osd.27
device 28 osd.28
device 29 osd.29
device 30 osd.30
device 31 osd.31
device 32 osd.32
device 33 osd.33
device 34 osd.34
device 35 osd.35
device 36 osd.36
device 37 osd.37
device 38 osd.38

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root

# buckets
host pulpo-osd01-hdd {
        id -2           # do not change unnecessarily
        # weight 87.3
        alg straw
        hash 0  # rjenkins1
        item osd.0 weight 7.275
        item osd.1 weight 7.275
        item osd.2 weight 7.275
        item osd.3 weight 7.275
        item osd.4 weight 7.275
        item osd.5 weight 7.275
        item osd.6 weight 7.275
        item osd.7 weight 7.275
        item osd.8 weight 7.275
        item osd.9 weight 7.275
        item osd.10 weight 7.275
        item osd.11 weight 7.275
}
host pulpo-osd02-hdd {
        id -3           # do not change unnecessarily
        # weight 87.3
        alg straw
        hash 0  # rjenkins1
        item osd.12 weight 7.275
        item osd.13 weight 7.275
        item osd.14 weight 7.275
        item osd.15 weight 7.275
        item osd.16 weight 7.275
        item osd.17 weight 7.275
        item osd.18 weight 7.275
        item osd.19 weight 7.275
        item osd.20 weight 7.275
        item osd.21 weight 7.275
        item osd.22 weight 7.275
        item osd.23 weight 7.275
}
host pulpo-osd03-hdd {
        id -4           # do not change unnecessarily
        # weight 87.3
        alg straw
        hash 0  # rjenkins1
        item osd.24 weight 7.275
        item osd.25 weight 7.275
        item osd.26 weight 7.275
        item osd.27 weight 7.275
        item osd.28 weight 7.275
        item osd.29 weight 7.275
        item osd.30 weight 7.275
        item osd.31 weight 7.275
        item osd.32 weight 7.275
        item osd.33 weight 7.275
        item osd.34 weight 7.275
        item osd.35 weight 7.275
}
host pulpo-osd01-nvme {
        id -5           # do not change unnecessarily
        # weight 1.086
        alg straw
        hash 0  # rjenkins1
        item osd.36 weight 1.086
}
host pulpo-osd02-nvme {
        id -6           # do not change unnecessarily
        # weight 1.086
        alg straw
        hash 0  # rjenkins1
        item osd.37 weight 1.086
}
host pulpo-osd03-nvme {
        id -7           # do not change unnecessarily
        # weight 1.086
        alg straw
        hash 0  # rjenkins1
        item osd.38 weight 1.086
}
root hdd {
        id -1           # do not change unnecessarily
        # weight 261.9
        alg straw
        hash 0  # rjenkins1
        item pulpo-osd01-hdd weight 87.3
        item pulpo-osd02-hdd weight 87.3
        item pulpo-osd03-hdd weight 87.3
}
root nvme {
        id -8           # do not change unnecessarily
        # weight 3.258
        alg straw
        hash 0  # rjenkins1
        item pulpo-osd01-nvme weight 1.086
        item pulpo-osd02-nvme weight 1.086
        item pulpo-osd03-nvme weight 1.086
}

# rules
rule replicated_ruleset {
        ruleset 0
        type replicated
        min_size 1
        max_size 10
        step take hdd
        step chooseleaf firstn 0 type host
        step emit
}

# end crush map
```

A quick summary of the modifications:
1. We replace the old host bucket `pulpo-osd01` (which contained both HDD OSDs and NVMe OSD) with `pulpo-osd01-hdd` (which only contains HDD OSDs) and `pulpo-osd01-nvme` (which contains the single NVMe OSD in the node);
2. We make similar modifications for pulpo-osd02 & pulpo-osd03;
3. We remove the old root bucket `default`;
4. We add a new root bucket `hdd`, which contains all OSDs backed by HDDs
5. We add a new root bucket `nvme`, which contains all OSDs backed by NVMes
6. We modify the default `replicated_ruleset` to take the root bucket `hdd` (so it'll only use OSDs backed by HDDs).

4) Compile the new CRUSH map:
```shell
[root@pulpo-admin Pulpos]# crushtool -c crushmap-2.txt -o crushmap-2.bin
```

5) Set the new CRUSH map for the cluster:
```shell
[root@pulpo-admin Pulpos]# ceph osd setcrushmap -i crushmap-2.bin
set crush map
```

Let's verify the new CRUSH tree:
```shell
[root@pulpo-admin Pulpos]# ceph osd tree
ID WEIGHT    TYPE NAME                 UP/DOWN REWEIGHT PRIMARY-AFFINITY
-8   3.25800 root nvme
-5   1.08600     host pulpo-osd01-nvme
36   1.08600         osd.36                 up  1.00000          1.00000
-6   1.08600     host pulpo-osd02-nvme
37   1.08600         osd.37                 up  1.00000          1.00000
-7   1.08600     host pulpo-osd03-nvme
38   1.08600         osd.38                 up  1.00000          1.00000
-1 261.90002 root hdd
-2  87.30000     host pulpo-osd01-hdd
 0   7.27499         osd.0                  up  1.00000          1.00000
 1   7.27499         osd.1                  up  1.00000          1.00000
 2   7.27499         osd.2                  up  1.00000          1.00000
 3   7.27499         osd.3                  up  1.00000          1.00000
 4   7.27499         osd.4                  up  1.00000          1.00000
 5   7.27499         osd.5                  up  1.00000          1.00000
 6   7.27499         osd.6                  up  1.00000          1.00000
 7   7.27499         osd.7                  up  1.00000          1.00000
 8   7.27499         osd.8                  up  1.00000          1.00000
 9   7.27499         osd.9                  up  1.00000          1.00000
10   7.27499         osd.10                 up  1.00000          1.00000
11   7.27499         osd.11                 up  1.00000          1.00000
-3  87.30000     host pulpo-osd02-hdd
12   7.27499         osd.12                 up  1.00000          1.00000
13   7.27499         osd.13                 up  1.00000          1.00000
14   7.27499         osd.14                 up  1.00000          1.00000
15   7.27499         osd.15                 up  1.00000          1.00000
16   7.27499         osd.16                 up  1.00000          1.00000
17   7.27499         osd.17                 up  1.00000          1.00000
18   7.27499         osd.18                 up  1.00000          1.00000
19   7.27499         osd.19                 up  1.00000          1.00000
20   7.27499         osd.20                 up  1.00000          1.00000
21   7.27499         osd.21                 up  1.00000          1.00000
22   7.27499         osd.22                 up  1.00000          1.00000
23   7.27499         osd.23                 up  1.00000          1.00000
-4  87.30000     host pulpo-osd03-hdd
24   7.27499         osd.24                 up  1.00000          1.00000
25   7.27499         osd.25                 up  1.00000          1.00000
26   7.27499         osd.26                 up  1.00000          1.00000
27   7.27499         osd.27                 up  1.00000          1.00000
28   7.27499         osd.28                 up  1.00000          1.00000
29   7.27499         osd.29                 up  1.00000          1.00000
30   7.27499         osd.30                 up  1.00000          1.00000
31   7.27499         osd.31                 up  1.00000          1.00000
32   7.27499         osd.32                 up  1.00000          1.00000
33   7.27499         osd.33                 up  1.00000          1.00000
34   7.27499         osd.34                 up  1.00000          1.00000
35   7.27499         osd.35                 up  1.00000          1.00000
```
and verify the ruleset `replicated_ruleset`:
```shell
[root@pulpo-admin Pulpos]# ceph osd crush rule dump replicated_ruleset
{
    "rule_id": 0,
    "rule_name": "replicated_ruleset",
    "ruleset": 0,
    "type": 1,
    "min_size": 1,
    "max_size": 10,
    "steps": [
        {
            "op": "take",
            "item": -1,
            "item_name": "hdd"
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
They both look about right!

6) Disable OSD CRUSH update on start:

Add the following 2 lines to `ceph.conf`:
```ini
[osd]
        osd_crush_update_on_start = false
```
Send the updated `ceph.conf` to all hosts:
```shell
[root@pulpo-admin Pulpos]# ceph-deploy --overwrite-conf config push pulpo-admin pulpo-dtn pulpo-mon01 pulpo-mds01 pulpo-osd01 pulpo-osd02 pulpo-osd03
```

7) Further readings:
* [Ceph: managing CRUSH with the CLI](https://www.sebastien-han.fr/blog/2014/01/13/ceph-managing-crush-with-the-cli/)
* [Ceph: mix SATA and SSD within the same box](https://www.sebastien-han.fr/blog/2014/08/25/ceph-mix-sata-and-ssd-within-the-same-box/)

## Adding an MDS
Add a Metadata server:
```shell
[root@pulpo-admin Pulpos]# ceph-deploy mds create pulpo-mds01
```
The goal is to create a [Ceph Filesystem (CephFS)](http://docs.ceph.com/docs/kraken/cephfs/), using 3 RADOS pools:
1. an Erasure Code data pool on the OSDs backed by HDDs
2. a replicated metadata pool on the OSDs backed by HDDs
3. a replicated pool on the OSDs backed by NVMes, as the cache tier of the Erasure Code data pool


## Creating an Erasure Code data pool
The default [erasure code](http://docs.ceph.com/docs/kraken/rados/operations/erasure-code/) profile sustains the loss of a single OSD.
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
[root@pulpo-admin Pulpos]# ceph osd erasure-code-profile set pulpo_ec k=2 m=1 ruleset-root=hdd plugin=jerasure technique=reed_sol_van

[root@pulpo-admin Pulpos]# ceph osd erasure-code-profile ls
default
pulpo_ec

[root@pulpo-admin Pulpos]# ceph osd erasure-code-profile get pulpo_ec
jerasure-per-chunk-alignment=false
k=2
m=1
plugin=jerasure
ruleset-failure-domain=host
ruleset-root=hdd
technique=reed_sol_van
w=8
```
The important parameter is `ruleset-root=hdd`, which set `hdd` as the root bucket for the CRUSH ruleset. So a pool created with this profile will only use the OSDs backed by HDDs.

Create the *Erasure Code* data pool for CephFS, with the `pulpo_ec` profile:
```shell
[root@pulpo-admin Pulpos]# ceph osd pool create cephfs_data 1024 1024 erasure pulpo_ec
pool 'cephfs_data' created
```
which also generates a new CRUSH ruleset with the same name `cephfs_data`.

Let's check the ruleset for pool `cephfs_data`:
```shell
[root@pulpo-admin Pulpos]# ceph osd pool get cephfs_data crush_ruleset
crush_ruleset: 1

[root@pulpo-admin Pulpos]# ceph osd dump | grep "^pool" | grep "crush_ruleset 1"
pool 1 'cephfs_data' erasure size 3 min_size 3 crush_ruleset 1 object_hash rjenkins pg_num 1024 pgp_num 1024 last_change 200 flags hashpspool stripe_width 4096

[root@pulpo-admin Pulpos]# ceph osd crush rule ls
[
    "replicated_ruleset",
    "cephfs_data"
]

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
            "item": -1,
            "item_name": "hdd"
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

## Creating a replicated metadata pool
Create the *replicated* metadata pool for CephFS, using the default CRUSH ruleset `replicated_ruleset`:
```shell
[root@pulpo-admin Pulpos]# ceph osd pool create cephfs_metadata 1024 1024 replicated
pool 'cephfs_metadata' created
```

Let’s verify the ruleset for pool `cephfs_metadata`:
```shell
[root@pulpo-admin Pulpos]# ceph osd pool get cephfs_metadata crush_ruleset
crush_ruleset: 0

[root@pulpo-admin Pulpos]# ceph osd dump | grep "^pool" | grep "crush_ruleset 0"
pool 0 'rbd' replicated size 3 min_size 2 crush_ruleset 0 object_hash rjenkins pg_num 1024 pgp_num 1024 last_change 185 flags hashpspool stripe_width 0
pool 2 'cephfs_metadata' replicated size 3 min_size 2 crush_ruleset 0 object_hash rjenkins pg_num 1024 pgp_num 1024 last_change 202 flags hashpspool stripe_width 0

[root@pulpo-admin Pulpos]# ceph osd crush rule dump replicated_ruleset
{
    "rule_id": 0,
    "rule_name": "replicated_ruleset",
    "ruleset": 0,
    "type": 1,
    "min_size": 1,
    "max_size": 10,
    "steps": [
        {
            "op": "take",
            "item": -1,
            "item_name": "hdd"
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

**NOTE** at this point, if we try to create a CephFS, we'll get an error!
```shell
[root@pulpo-admin Pulpos]# ceph fs new pulpos cephfs_metadata cephfs_data
Error EINVAL: pool 'cephfs_data' (id '1') is an erasure-code pool
```
So in Kraken, if the data pool is erasure-coded, it is required to add a 'cache tier' to the data pool for CephFS to work!

## Adding Cache Tiering to the data pool
The goal is to create a replicated pool on the OSDs backed by the NVMes, as the [cache tier](http://docs.ceph.com/docs/kraken/rados/operations/cache-tiering/) of the Erasure Code data pool for the CephFS.

1) Create a new CRUSH ruleset for the cache pool:
```shell
[root@pulpo-admin Pulpos]# ceph osd crush rule create-simple replicated_nvme nvme host
[root@pulpo-admin Pulpos]# ceph osd crush rule list
[
    "replicated_ruleset",
    "cephfs_data",
    "replicated_nvme"
]

[root@pulpo-admin Pulpos]# ceph osd crush rule dump replicated_nvme
{
    "rule_id": 2,
    "rule_name": "replicated_nvme",
    "ruleset": 2,
    "type": 1,
    "min_size": 1,
    "max_size": 10,
    "steps": [
        {
            "op": "take",
            "item": -8,
            "item_name": "nvme"
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
[root@pulpo-admin Pulpos]# ceph osd pool create cephfs_cache 128 128 replicated replicated_nvme
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
osdmap e215 pg 3.5c (3.5c) -> up [38,37] acting [38,37]
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
Now we can create the Ceph Filesystem:
```shell
[root@pulpo-admin Pulpos]# ceph fs new pulpos cephfs_metadata cephfs_data
new fs with metadata pool 2 and data pool 1
```

## Mounting CephFS on clients
There are 2 ways to [mount CephFS](http://docs.ceph.com/docs/master/cephfs/best-practices/) on a client: using either the kernel CephFS driver, or ceph-fuse. The fuse client is the easiest way to get up to date code, while the kernel client will often give better performance.

On a client, e.g., *pulpo-dtn*, create the mount point:
```shell
[root@pulpo-dtn ~]# mkdir /mnt/pulpos
```

### Kernel CephFS driver
The Ceph Storage Cluster runs with authentication turned on by default. We need a file containing the secret key (i.e., not the keyring itself).

0) [Create the secret file](http://docs.ceph.com/docs/kraken/start/quick-cephfs/#create-a-secret-file), and save it as `/etc/ceph/admin.secret`.

1) We can use the `mount` command to [mount CephFS with kernel driver](http://docs.ceph.com/docs/kraken/cephfs/kernel/)
```shell
[root@pulpo-dtn ~]# mount -t ceph 128.114.86.4:6789:/ /mnt/pulpos -o name=admin,secretfile=/etc/ceph/admin.secret
```
or more redundantly
```shell
[root@pulpo-dtn ~]# mount -t ceph 128.114.86.4:6789,128.114.86.5:6789,128.114.86.2:6789:/ /mnt/pulpos -o name=admin,secretfile=/etc/ceph/admin.secret
```
With this method, we need to specify the monitor host IP address(es) and port number(s).

2) Or we can use the simple helper [mount.ceph](http://docs.ceph.com/docs/kraken/man/8/mount.ceph/), which resolve monitor hostname(s) into IP address(es):
```shell
[root@pulpo-dtn ~]# mount.ceph pulpo-mon01:/ /mnt/pulpos -o name=admin,secretfile=/etc/ceph/admin.secret
```
or more redundantly
```shell
[root@pulpo-dtn ~]# mount.ceph pulpo-mon01,pulpo-mds01,pulpo-admin:/ /mnt/pulpos -o name=admin,secretfile=/etc/ceph/admin.secret
```

3) To [mount CephFS automatically on startup](http://docs.ceph.com/docs/kraken/cephfs/fstab/#kernel-driver), we can add the following to `/etc/fstab`:
```conf
128.114.86.4:6789,128.114.86.5:6789,128.114.86.2:6789:/  /mnt/pulpos  ceph  name=admin,secretfile=/etc/ceph/admin.secret,noatime,_netdev  0  2
```

### ceph-fuse
Make sure the `ceph-fuse` package is installed. We've already installed the package on *pulpo-dtn*, using *Ansible*.

`cephx` authentication is on by default. Ensure that the client host has a copy of the Ceph configuration file and a keyring with CAPS for the Ceph metadata server. *pulpo-dtn* already has a copy of these 2 files. **NOTE** [ceph-fuse](http://docs.ceph.com/docs/kraken/cephfs/fuse/) uses the keyring rather than a secret file for authentication!

Then we can use the `ceph-fuse` command to mount the CephFS as a FUSE ([Filesystem in Userspace](https://en.wikipedia.org/wiki/Filesystem_in_Userspace)):
on pulpo-dtn:
```shell
[root@pulpo-dtn ~]# ceph-fuse -m 128.114.86.4:6789 /mnt/pulpos
ceph-fuse[11424]: starting ceph client2017-09-04 11:35:32.792972 7f0949207f00 -1 init, newargv = 0x7f09537e8c60 newargc=11

ceph-fuse[11424]: starting fuse
```
or more redundantly:
```shell
[root@pulpo-dtn ~]# ceph-fuse -m pulpo-mon01:6789,pulpo-mds01:6789,pulpo-admin:6789 /mnt/pulpos
```

There are 2 options to automate mounting ceph-fuse: `fstab` or `systemd`.

1) We can add the following to `/etc/fstab` (see [http://docs.ceph.com/docs/kraken/cephfs/fstab/#fuse](http://docs.ceph.com/docs/kraken/cephfs/fstab/#fuse)):
```conf
id=admin  /mnt/pulpos  fuse.ceph  defaults,_netdev  0  0
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
