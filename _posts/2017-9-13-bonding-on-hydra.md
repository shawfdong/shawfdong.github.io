---
layout: post
title: Bonding on Hydra
tags: [Linux, Network]
---

In this post, we document how we aggregated two 1GbE network interfaces into a logical *bonded* interface, on the 4-GPU workstation [Hydra]({{ site.baseurl }}{% post_url 2017-7-28-hydra %}).<!-- more --> We'll use mode 6 (`balance-alb`) of [Linux Ethernet Bonding Driver](https://www.kernel.org/doc/Documentation/networking/bonding.txt), because UCSC does not support [Link Aggregation](https://en.wikipedia.org/wiki/Link_aggregation) for servers hosted outside the data center.

0) The two 1GbE network interfaces are `eno1` & `eno2`. It is prudent to back up the old configurations: */etc/sysconfig/network-scripts/ifcfg-eno1* & */etc/sysconfig/network-scripts/ifcfg-eno2*.

1) [RHEL 7 Networking Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/networking_guide/sec-network_bonding_using_the_command_line_interface) says the bonding module is not loaded by default in RHEL/CentOS 7 and one needs to load the module:
{% highlight shell_session %}
# modprobe --first-time bonding
{% endhighlight %}
but I find it unnecessary; because I didn't do it and everything worked fine.

2) Create configuration file `/etc/sysconfig/network-scripts/ifcfg-bond0` for the Channel Bonding Interface `bond0`:
{% highlight conf %}
DEVICE=bond0
NAME=bond0
TYPE=Bond
BONDING_MASTER=yes
BONDING_OPTS="mode=balance-alb miimon=100"
IPADDR=128.114.59.108
PREFIX=24
GATEWAY=128.114.59.1
DNS1=8.8.8.8
DOMAIN=soe.ucsc.edu
BOOTPROTO=none
ONBOOT=yes
IPV6INIT=yes
IPV6_AUTOCONF=yes
{% endhighlight %}
<p class="note">In this case we use mode 6 (<em>balance-alb</em>), which does not require any specific configuration of the switch.</p>

3a) Edit configuration file `/etc/sysconfig/network-scripts/ifcfg-eno1` for SLAVE interface `eno1`:
{% highlight conf %}
DEVICE=eno1
NAME=bond0-eno1
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
MASTER=bond0
SLAVE=yes
{% endhighlight %}

3b) Edit configuration file `/etc/sysconfig/network-scripts/ifcfg-eno2` for SLAVE interface `eno2`:
{% highlight conf %}
DEVICE=eno2
NAME=bond0-eno2
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
MASTER=bond0
SLAVE=yes
{% endhighlight %}

4) Activate the Channel Bond:
{% highlight shell_session %}
# ifdown eno1; ifdown eno2; nmcli con reload; ifup bond0; ifup eno1; ifup eno2
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/4)
{% endhighlight %}
Note that we were operating in-band, via an SSH session over *eno1*; so we chained the commands together. And it worked! My SSH connection was not even dropped!

5) View the status of the bond interface:
{% highlight shell_session %}
[root@hydra ~]# ip link
2: eno1: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc mq master bond0 state UP mode DEFAULT qlen 1000
    link/ether ac:1f:6b:21:d1:5a brd ff:ff:ff:ff:ff:ff
3: eno2: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc mq master bond0 state UP mode DEFAULT qlen 1000
    link/ether ac:1f:6b:21:d1:5b brd ff:ff:ff:ff:ff:ff
38: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT qlen 1000
    link/ether ac:1f:6b:21:d1:5a brd ff:ff:ff:ff:ff:ff

[root@hydra ~]# ip address
2: eno1: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc mq master bond0 state UP qlen 1000
    link/ether ac:1f:6b:21:d1:5a brd ff:ff:ff:ff:ff:ff
3: eno2: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 1500 qdisc mq master bond0 state UP qlen 1000
    link/ether ac:1f:6b:21:d1:5b brd ff:ff:ff:ff:ff:ff
38: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP qlen 1000
    link/ether ac:1f:6b:21:d1:5a brd ff:ff:ff:ff:ff:ff
    inet 128.114.59.108/24 brd 128.114.59.255 scope global bond0
       valid_lft forever preferred_lft forever
    inet6 fe80::ae1f:6bff:fe21:d15a/64 scope link
       valid_lft forever preferred_lft forever
{% endhighlight %}

**/proc/net/bonding/bond0**:
{% highlight conf %}
Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: adaptive load balancing
Primary Slave: None
Currently Active Slave: eno1
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

Slave Interface: eno1
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: ac:1f:6b:21:d1:5a
Slave queue ID: 0

Slave Interface: eno2
MII Status: up
Speed: 1000 Mbps
Duplex: full
Link Failure Count: 1
Permanent HW addr: ac:1f:6b:21:d1:5b
Slave queue ID: 0
{% endhighlight %}

**sysfs**:
{% highlight shell_session %}
[root@hydra ~]# cat /sys/class/net/bond0/bonding/slaves
eno1 eno2
{% endhighlight %}
