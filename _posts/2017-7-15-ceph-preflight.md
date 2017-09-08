---
layout: post
title: Ceph Preflight
tags: [Ceph, Storage, Provisioning]
---

In this post, we describe how we prepared the nodes in the [Pulpos]({{ site.baseurl }}{% post_url 2017-2-9-pulpos %}) cluster for Ceph installation.<!-- more -->

## Chrony
The clocks on Ceph nodes (especially Ceph Monitor nodes) should be synchronized to prevent issues arising from [clock](http://docs.ceph.com/docs/master/rados/configuration/mon-config-ref/#clock) drift. **Chrony** is an implementation of the Network Time Protocol (NTP); and is the [preferred NTD daemon on RHEL/CentOS 7](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/System_Administrators_Guide/ch-Configuring_NTP_Using_the_chrony_Suite.html). Hence we run **chronyd** rather than **ntpd** on Pulpos nodes.

The **chrony** suite has been installed on every Pulpos node during the [auto installation of CentOS 7]({{ site.baseurl }}{% post_url 2017-7-12-auto-install-of-centos %}).

## Password-less SSH access
The SSH public key for `root` has been copied to every Pulpos node during the [auto installation of CentOS 7]({{ site.baseurl }}{% post_url 2017-7-12-auto-install-of-centos %}).

## Firewall
1) [FirewallD]({{ site.baseurl }}{% post_url 2017-7-11-firewalld-on-centos %}) is installed by default with CentOS 7 on every Pulpos node;

2) We've used a simple [Ansible playbook](http://docs.ansible.com/ansible/latest/playbooks.html) to bind all 1GbE and 40GbE interfaces on every node to the `trusted` zone of FirewallD;

3) We'll run `mon` daemons on *pulpo-mon01*, *pulpo-mds01* & *pulpo-admin*; `osd` daemons on *pulpo-osd01*, *pulpo-osd02* & *pulpo-osd03*; and `mds` daemon on *pulpo-mds01*.  Let's [open the required ports](http://docs.ceph.com/docs/luminous/start/quick-start-preflight/#ensure-connectivity):
```shell
# ansible -m command -a "firewall-cmd --zone=public --add-service=ceph-mon --permanent" mons

# ansible -m command -a "firewall-cmd --zone=public --add-service=ceph --permanent" osds

# ssh pulpo-mds01.local "firewall-cmd --zone=public --add-service=ceph --permanent" osds"

# ansible -m command -a "firewall-cmd --reload" all
```

## SELINUX
SELINUX is doubly disabled, with both a kernel parameter `selinux=0` and the configuration file `/etc/selinux/config`.

## IPv6
Ceph doesn't support dual-stack yet! So we've only configured IPv4 addresses on the network interfaces, even though we do have [both IPv4 and IPv6 addresses available for all 10G public interfaces]({{ site.baseurl }}{% post_url 2017-6-21-pulpos-networks %}).  
