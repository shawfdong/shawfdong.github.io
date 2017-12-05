---
layout: post
title: Bonding on pulpo-dtn
tags: [Linux, Network]
---

In this post, we document how we aggregated two 10GbE network interfaces into a logical *bonded* interface, on on **pulpo-dtn**.<!-- more --> We'll use mode 4 (`802.3ad`) of [Linux Ethernet Bonding Driver](https://www.kernel.org/doc/Documentation/networking/bonding.txt), which requires a switch that supports IEEE 802.3ad dynamic [Link Aggregation](https://en.wikipedia.org/wiki/Link_aggregation). We've just configured the [Pica8 switch]({{ site.baseurl }}{% post_url 2017-9-14-pica8-switch %}) to support IEEE 802.3ad dynamic link aggregation.

0) The two *10GbE* network interfaces are `ens1f0` & `ens1f1`. It is prudent to back up the old configurations: */etc/sysconfig/network-scripts/ifcfg-ens1f0* & */etc/sysconfig/network-scripts/ifcfg-ens1f1*.

1) [RHEL 7 Networking Guide](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/networking_guide/sec-network_bonding_using_the_command_line_interface) says the bonding module is not loaded by default in RHEL/CentOS 7 and one needs to load the module:
{% highlight shell_session %}
[root@pulpo-dtn ~]# modprobe --first-time bonding
{% endhighlight %}
but that [may not be necessary]({{ site.baseurl }}{% post_url 2017-9-13-bonding-on-hydra %}).

2) Bring down the two *10GbE* interfaces:
{% highlight shell_session %}
# ifdown ens1f0
# ifdown ens1f1
{% endhighlight %}

3) Create configuration file `/etc/sysconfig/network-scripts/ifcfg-bond0` for the Channel Bonding Interface `bond0`:
{% highlight conf %}
DEVICE=bond0
NAME=bond0
TYPE=Bond
BONDING_MASTER=yes
BONDING_OPTS="mode=802.3ad miimon=100"
IPADDR=128.114.86.3
NETMASK=255.255.255.0
GATEWAY=128.114.86.254
DNS1=8.8.8.8
BOOTPROTO=none
ONBOOT=yes
MTU=9000
IPV6INIT=yes
IPV6_AUTOCONF=yes
{% endhighlight %}

4a) Edit configuration file `/etc/sysconfig/network-scripts/ifcfg-ens1f0` for SLAVE interface `ens1f0`:
{% highlight conf %}
DEVICE=ens1f0
NAME=bond0-ens1f0
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
MASTER=bond0
SLAVE=yes
{% endhighlight %}

4b) Edit configuration file `/etc/sysconfig/network-scripts/ifcfg-ens1f1` for SLAVE interface `ens1f1`:
{% highlight conf %}
DEVICE=ens1f1
NAME=bond0-ens1f1
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
MASTER=bond0
SLAVE=yes
{% endhighlight %}

5) Reload all interfaces, to make **NetworkManager** aware of the changes:
{% highlight shell_session %}
[root@pulpo-dtn ~]# nmcli con reload
{% endhighlight %}

6) Bring up the Channel Bonding interface:
{% highlight shell_session %}
[root@pulpo-dtn ~]# ifup bond0
[root@pulpo-dtn ~]# ifup ens1f0
[root@pulpo-dtn ~]# ifup ens1f1
{% endhighlight %}

7) View the status of the bond interface:
{% highlight shell_session %}
[root@pulpo-dtn ~]# ip link
4: ens1f0: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 9000 qdisc mq master bond0 state UP mode DEFAULT qlen 1000
    link/ether 90:e2:ba:85:59:a4 brd ff:ff:ff:ff:ff:ff
5: ens1f1: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 9000 qdisc mq master bond0 state UP mode DEFAULT qlen 1000
    link/ether 90:e2:ba:85:59:a4 brd ff:ff:ff:ff:ff:ff
6: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 9000 qdisc noqueue state UP mode DEFAULT qlen 1000
    link/ether 90:e2:ba:85:59:a4 brd ff:ff:ff:ff:ff:ff

[root@pulpo-dtn ~]# ip address
4: ens1f0: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 9000 qdisc mq master bond0 state UP qlen 1000
link/ether 90:e2:ba:85:59:a4 brd ff:ff:ff:ff:ff:ff
5: ens1f1: <BROADCAST,MULTICAST,SLAVE,UP,LOWER_UP> mtu 9000 qdisc mq master bond0 state UP qlen 1000
link/ether 90:e2:ba:85:59:a4 brd ff:ff:ff:ff:ff:ff
6: bond0: <BROADCAST,MULTICAST,MASTER,UP,LOWER_UP> mtu 9000 qdisc noqueue state UP qlen 1000
link/ether 90:e2:ba:85:59:a4 brd ff:ff:ff:ff:ff:ff
inet 128.114.86.3/24 brd 128.114.86.255 scope global bond0
   valid_lft forever preferred_lft forever
inet6 2607:f5f0:100:1:92e2:baff:fe85:59a4/64 scope global noprefixroute dynamic
   valid_lft 2591971sec preferred_lft 604771sec
inet6 fe80::92e2:baff:fe85:59a4/64 scope link
   valid_lft forever preferred_lft forever
{% endhighlight %}

**/proc/net/bonding/bond0**:
{% highlight conf %}
Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: IEEE 802.3ad Dynamic link aggregation
Transmit Hash Policy: layer2 (0)
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

802.3ad info
LACP rate: slow
Min links: 0
Aggregator selection policy (ad_select): stable
System priority: 65535
System MAC address: 90:e2:ba:85:59:a4
Active Aggregator Info:
        Aggregator ID: 1
        Number of ports: 2
        Actor Key: 13
        Partner Key: 69
        Partner Mac Address: 08:9e:01:f8:63:ac

Slave Interface: ens1f0
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 90:e2:ba:85:59:a4
Slave queue ID: 0
Aggregator ID: 1
Actor Churn State: none
Partner Churn State: none
Actor Churned Count: 0
Partner Churned Count: 0
details actor lacp pdu:
    system priority: 65535
    system mac address: 90:e2:ba:85:59:a4
    port key: 13
    port priority: 255
    port number: 1
    port state: 61
details partner lacp pdu:
    system priority: 32768
    system mac address: 08:9e:01:f8:63:ac
    oper key: 69
    port priority: 32768
    port number: 4
    port state: 61

Slave Interface: ens1f1
MII Status: up
Speed: 10000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 90:e2:ba:85:59:a5
Slave queue ID: 0
Aggregator ID: 1
Actor Churn State: none
Partner Churn State: none
Actor Churned Count: 0
Partner Churned Count: 0
details actor lacp pdu:
    system priority: 65535
    system mac address: 90:e2:ba:85:59:a4
    port key: 13
    port priority: 255
    port number: 2
    port state: 61
details partner lacp pdu:
    system priority: 32768
    system mac address: 08:9e:01:f8:63:ac
    oper key: 69
    port priority: 32768
    port number: 3
    port state: 61
{% endhighlight %}

**sysfs**:
{% highlight shell_session %}
[root@pulpo-dtn ~]# cat /sys/class/net/bond0/bonding/slaves
ens1f0 ens1f1
{% endhighlight %}    

8) Check Firewall:
{% highlight shell_session %}
[root@pulpo-dtn ~]# firewall-cmd --get-active-zones
public
  interfaces: bond0 ens1f0 ens1f1
trusted
  interfaces: eno1
{% endhighlight %}
