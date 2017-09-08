---
layout: post
title: Pulpo-Admin
tags: [Ceph, Storage, Linux]
---

The admin node of our Ceph storage cluster [Pulpos]({{ site.baseurl }}{% post_url 2017-2-9-pulpos %}) is **pulpo-admin**.<!-- more -->

## Hardware
* Two 8-core [Intel Xeon E5-2620 v4](https://ark.intel.com/products/92986/Intel-Xeon-Processor-E5-2620-v4-20M-Cache-2_10-GHz) processors @ 2.1 GHz
* 32GB DDR4 memory @ 2400 MHz
* 480GB [Intel DC S3500 Series](http://ark.intel.com/products/series/74935/Intel-SSD-DC-S3500-Series) SSD
* [Intel X520-DA2 10GbE adapter](http://ark.intel.com/products/39776/Intel-Ethernet-Converged-Network-Adapter-X520-DA2), with 2 SFP+ ports
* [Supermicro 6028TP-HTR](https://www.supermicro.com/products/system/2u/6028/sys-6028tp-htr.cfm) 2U Quad-Nodes chassis

## Software
1) Performe a minimal installation of CentOS 7 on *pulpo-admin*.

2) Disable Selinux by changing the following line in `/etc/selinux/config`:
```conf
SELINUX=enforcing
```
to:
```conf
SELINUX=disabled
```

3) After copying SSH keys to the host, disable password authentication of SSH by changing the following line in `/etc/ssh/sshd_config`:
```conf
PasswordAuthentication yes
```
to
```conf
PasswordAuthentication no
```

4) Disable GSSAPI authentication of SSH by changing the following line in `/etc/ssh/sshd_config`:
```conf
GSSAPIAuthentication yes
```
to:
```conf
GSSAPIAuthentication no
```

5) Update all packages:
```shell
[root@pulpo-admin ~]# yum -y update
```

6) Install the package `net-tools`, which contains basic networking tools, including *ifconfig*, *netstat*, *route*, and others:
```shell
[root@pulpo-admin ~]# yum install -y net-tools
```

7) Install the package `bind-utils`, which contains a collection of utilities for querying DNS servers, including *dig*, *nslookup*, and others:
```shell
[root@pulpo-admin ~]# yum install -y bind-utils
```

8) Reboot.

9) Remove the old kernel:
```shell
[root@pulpo-admin ~]# yum erase -y kernel-3.10.0-514.el7.x86_64
```

10) Create a pair of SSH keys of type `ed25519` for *root*:
```shell
[root@pulpo-admin ~]# cd ~/.ssh
[root@pulpo-admin .ssh]# ssh-keygen -t ed25519
```

<p class="note"><em>hyper-threading</em> is enabled on pulpo-admin, as well as on all the other nodes in the Pulpos cluster.</p>
