---
layout: post
title: Network Tuning for Pulpos
tags: [Linux, Network]
---

In the post, we document the network tuning we've done for the nodes in the [Pulpos]({{ site.baseurl }}{% post_url 2017-2-9-pulpos %}) cluster.<!-- more -->

* Table of Contents
{:toc}

## Default Settings
Default settings on CentOS 7 hosts with 32GB RAM (such as *pulpo-admin*):
{% highlight conf %}
net.core.rmem_default = 212992
net.core.rmem_max = 212992
net.core.wmem_default = 212992
net.core.wmem_max = 212992
net.core.netdev_max_backlog = 1000
net.ipv4.tcp_rmem = 4096        87380   6291456
net.ipv4.tcp_wmem = 4096        16384   4194304
net.ipv4.tcp_mem = 766212       1021616 1532424
net.ipv4.udp_mem = 768948       1025267 1537896
net.ipv4.udp_rmem_min = 4096
net.ipv4.udp_wmem_min = 4096
net.ipv4.tcp_congestion_control = cubic
net.ipv4.tcp_mtu_probing = 0
{% endhighlight %}

Default settings on CentOS 7 hosts with 64GB RAM (such as *pulpo-mds01*):
{% highlight conf %}
net.core.rmem_default = 212992
net.core.rmem_max = 212992
net.core.wmem_default = 212992
net.core.wmem_max = 212992
net.core.netdev_max_backlog = 1000
net.ipv4.tcp_rmem = 4096        87380   6291456
net.ipv4.tcp_wmem = 4096        16384   4194304
net.ipv4.tcp_mem = 1540329      2053772 3080658
net.ipv4.udp_mem = 1543059      2057414 3086118
net.ipv4.udp_rmem_min = 4096
net.ipv4.udp_wmem_min = 4096
net.ipv4.tcp_congestion_control = cubic
net.ipv4.tcp_mtu_probing = 0
{% endhighlight %}

And the default *Transmit Queue Length (txqueuelen)* is 1000:
{% highlight shell_session %}
[root@pulpo-admin ~]# ifconfig ens1f0 | grep txqueuelen
        ether 90:e2:ba:6f:83:58  txqueuelen 1000  (Ethernet)
{% endhighlight %}

## Install network test tools
`iperf3` is available at the base repo & `nuttcp` at EPEL:
{% highlight shell_session %}
[root@pulpo-admin ~]# yum -y install iperf3 nuttcp
[root@pulpo-admin ~]# tentakel yum -y install iperf3 nuttcp
{% endhighlight %}

### iperf3
ESnet maintains an excellent [guide on iperf3](http://fasterdata.es.net/performance-testing/network-troubleshooting-tools/iperf/). The default port for iper3 is 5201.

The default port for iper3 is 5201.

1) First we'll perform a memory to memory test between two 10G hosts: venadi (a FIONA box) and pulpo-admin. On venadi, all those ports are open:
{% highlight shell_session %}
[root@venadi ~]# firewall-cmd --list-all
public (default, active)
  interfaces: enp5s0
  sources:
  services: dhcpv6-client http ssh
  ports: 6001-6200/tcp 50000-51000/udp 4823/tcp 5001-5900/tcp 6001-6200/udp 2223/tcp 5001-5900/udp 50000-51000/tcp
  masquerade: no
  forward-ports:
  icmp-blocks:
  rich rules:
{% endhighlight %}

Start *iperf3* server on venadi:
{% highlight shell_session %}
[root@venadi ~]# iperf3 -s
{% endhighlight %}

Start *iperf3* client on pulpo-admin:
{% highlight shell_session %}
[root@pulpo-admin ~]# iperf3 -c venadi.ucsc.edu -i 1 -t 10 -V
iperf 3.1.7
Linux pulpo-admin.ucsc.edu 3.10.0-693.2.2.el7.x86_64 #1 SMP Tue Sep 12 22:26:13 UTC 2017 x86_64
Control connection MSS 8948
Time: Tue, 10 Oct 2017 21:05:46 GMT
Connecting to host venadi.ucsc.edu, port 5201
      Cookie: pulpo-admin.ucsc.edu.1507669546.2008
      TCP MSS: 8948 (default)
[  4] local 128.114.86.2 port 51346 connected to 128.114.109.74 port 5201
Starting Test: protocol: TCP, 1 streams, 131072 byte blocks, omitting 0 seconds, 10 second test
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec  1.15 GBytes  9.91 Gbits/sec    0    830 KBytes
[  4]   1.00-2.00   sec  1.15 GBytes  9.89 Gbits/sec    0    830 KBytes
[  4]   2.00-3.00   sec  1.15 GBytes  9.90 Gbits/sec    0    830 KBytes
[  4]   3.00-4.00   sec  1.15 GBytes  9.91 Gbits/sec    0    830 KBytes
[  4]   4.00-5.00   sec  1.15 GBytes  9.90 Gbits/sec    0    830 KBytes
[  4]   5.00-6.00   sec  1.15 GBytes  9.90 Gbits/sec    0    830 KBytes
[  4]   6.00-7.00   sec  1.15 GBytes  9.90 Gbits/sec    0    830 KBytes
[  4]   7.00-8.00   sec  1.15 GBytes  9.90 Gbits/sec    0    830 KBytes
[  4]   8.00-9.00   sec  1.14 GBytes  9.82 Gbits/sec    0    830 KBytes
[  4]   9.00-10.00  sec  1.15 GBytes  9.90 Gbits/sec    0    961 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
Test Complete. Summary Results:
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  11.5 GBytes  9.89 Gbits/sec    0             sender
[  4]   0.00-10.00  sec  11.5 GBytes  9.89 Gbits/sec                  receiver
CPU Utilization: local/sender 26.9% (0.3%u/26.7%s), remote/receiver 10.5% (0.5%u/10.0%s)
snd_tcp_congestion cubic

iperf Done.
{% endhighlight %}

2) Next we'll perform a memory to memory test between two 40G hosts: pulpo-osd01 & pulpo-osd02.

Start *iperf3* server on pulpo-osd01:
{% highlight shell_session %}
[root@pulpo-osd01 ~]# iperf3 -s
{% endhighlight %}

Start *iperf3* client on pulpo-osd02:
{% highlight shell_session %}
[root@pulpo-osd02 ~]# iperf3 -c pulpo-osd01.cluster -i 1 -t 10 -V
iperf 3.1.7
Linux pulpo-osd02.ucsc.edu 3.10.0-693.2.2.el7.x86_64 #1 SMP Tue Sep 12 22:26:13 UTC 2017 x86_64
Control connection MSS 8948
Time: Tue, 10 Oct 2017 21:47:44 GMT
Connecting to host pulpo-osd01.cluster, port 5201
      Cookie: pulpo-osd02.ucsc.edu.1507672064.6398
      TCP MSS: 8948 (default)
[  4] local 192.168.40.6 port 57414 connected to 192.168.40.5 port 5201
Starting Test: protocol: TCP, 1 streams, 131072 byte blocks, omitting 0 seconds, 10 second test
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec  2.62 GBytes  22.5 Gbits/sec    0    717 KBytes
[  4]   1.00-2.00   sec  2.78 GBytes  23.9 Gbits/sec    0    717 KBytes
[  4]   2.00-3.00   sec  2.93 GBytes  25.2 Gbits/sec    0    717 KBytes
[  4]   3.00-4.00   sec  2.94 GBytes  25.2 Gbits/sec   20    507 KBytes
[  4]   4.00-5.00   sec  2.87 GBytes  24.7 Gbits/sec    0    507 KBytes
[  4]   5.00-6.00   sec  2.90 GBytes  24.9 Gbits/sec    0    533 KBytes
[  4]   6.00-7.00   sec  2.90 GBytes  24.9 Gbits/sec    0    533 KBytes
[  4]   7.00-8.00   sec  2.93 GBytes  25.2 Gbits/sec    0    568 KBytes
[  4]   8.00-9.00   sec  2.85 GBytes  24.5 Gbits/sec    2    428 KBytes
[  4]   9.00-10.00  sec  2.84 GBytes  24.4 Gbits/sec    0    472 KBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
Test Complete. Summary Results:
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  28.6 GBytes  24.5 Gbits/sec   22             sender
[  4]   0.00-10.00  sec  28.6 GBytes  24.5 Gbits/sec                  receiver
CPU Utilization: local/sender 34.3% (0.4%u/34.0%s), remote/receiver 15.2% (0.1%u/15.2%s)
snd_tcp_congestion cubic
rcv_tcp_congestion cubic

iperf Done.
{% endhighlight %}
Before network tuning, we only got about 25 Gbps between 40G hosts!

### nuttcp
ESnet also maintains an excellent [guide on nuttcp](http://fasterdata.es.net/performance-testing/network-troubleshooting-tools/nuttcp/).

By default, the nuttcp server listens for commands on port 5000, and the actual nuttcp data transfers take place on port 5001.

1) First we’ll perform a memory to memory test between two 10G hosts: venadi (a FIONA box) and pulpo-admin. On venadi, Open TCP port 5000 on the server (venadi):
{% highlight shell_session %}
[root@venadi ~]# firewall-cmd --zone=public --add-port=5000/tcp
success
{% endhighlight %}

Open TCP port 5001 on the client (pulpo-admin):
{% highlight shell_session %}
[root@pulpo-admin ~]#  firewall-cmd --zone=public --add-port=5000-5100/tcp
success
{% endhighlight %}

Start *nuttcp* server on venadi:
{% highlight shell_session %}
[root@venadi ~]# nuttcp -S
{% endhighlight %}

Start *nuttcp* client on pulpo-admin::
{% highlight shell_session %}
[root@pulpo-admin ~]# nuttcp -i1 -r venadi.ucsc.edu
 1178.5000 MB /   1.00 sec = 9885.4901 Mbps    17 retrans
 1179.9375 MB /   1.00 sec = 9898.3697 Mbps    27 retrans
 1180.0000 MB /   1.00 sec = 9898.5376 Mbps    34 retrans
 1180.0000 MB /   1.00 sec = 9898.4585 Mbps    29 retrans
 1179.4375 MB /   1.00 sec = 9893.8883 Mbps    40 retrans
 1179.6250 MB /   1.00 sec = 9895.1248 Mbps    30 retrans
 1179.4375 MB /   1.00 sec = 9894.0763 Mbps    38 retrans
 1179.8125 MB /   1.00 sec = 9896.8658 Mbps    24 retrans
 1179.5000 MB /   1.00 sec = 9894.6006 Mbps    33 retrans
 1179.3750 MB /   1.00 sec = 9893.3245 Mbps    29 retrans

11801.2792 MB /  10.01 sec = 9894.3578 Mbps 11 %TX 49 %RX 301 retrans 0.18 msRTT
{% endhighlight %}
Almost at line rate despite the retrans!

Kill the server:
{% highlight shell_session %}
[root@venadi ~]# pkill nuttcp
{% endhighlight %}

Remove the TCP ports on both server and client:
{% highlight shell_session %}
[root@venadi ~]# firewall-cmd --zone=public --remove-port=5000/tcp

[root@pulpo-admin ~]# firewall-cmd --zone=public --remove-port=5000-5100/tcp
{% endhighlight %}

2) Next we’ll perform a memory to memory test between two 40G hosts: pulpo-osd01 & pulpo-osd01.

Start *nuttcp* server (one shot) on pulpo-osd01:
{% highlight shell_session %}
[root@pulpo-osd01 ~]# nuttcp -1
{% endhighlight %}

Start *nuttcp* client on pulpo-osd02:
{% highlight shell_session %}
[root@pulpo-osd02 ~]# nuttcp -i1 pulpo-osd01.cluster
 2729.2500 MB /   1.00 sec = 22892.2734 Mbps     0 retrans
 2761.8125 MB /   1.00 sec = 23169.7550 Mbps     0 retrans
 2797.0000 MB /   1.00 sec = 23463.2181 Mbps     0 retrans
 2765.2500 MB /   1.00 sec = 23196.5287 Mbps     0 retrans
 2758.5625 MB /   1.00 sec = 23140.5689 Mbps     0 retrans
 2744.1875 MB /   1.00 sec = 23019.5449 Mbps     0 retrans
 2824.8125 MB /   1.00 sec = 23696.5291 Mbps     0 retrans
 2870.1875 MB /   1.00 sec = 24077.1427 Mbps     0 retrans
 2919.0000 MB /   1.00 sec = 24486.1264 Mbps     0 retrans
 2784.3125 MB /   1.00 sec = 23355.1515 Mbps     0 retrans

27960.6875 MB /  10.00 sec = 23452.7184 Mbps 42 %TX 99 %RX 0 retrans 0.21 msRTT
{% endhighlight %}
Before network tuning, we only got about 24 Gbps between 40G hosts, with a lot of retrans!


## sysctl
### FIONA settings
FIONA boxes use the following kernel parameters (`/etc/sysctl.d/prp.conf`) on 40G hosts:
{% highlight conf %}
# buffers up to 512MB
net.core.rmem_max=536870912
net.core.wmem_max=536870912
# increase Linux autotuning TCP buffer limit to 256MB
net.ipv4.tcp_rmem=4096 87380 268435456
net.ipv4.tcp_wmem=4096 65536 268435456
net.core.netdev_max_backlog=250000
net.ipv4.tcp_congestion_control=cubic
net.ipv4.tcp_mtu_probing=1
net.ipv4.tcp_no_metrics_save = 1
{% endhighlight %}

### ESnet recommendations
ESnet maintains excellent [guide on how to tune Linux, Mac OSX, and FreeBSD hosts connected at speeds of 1Gbps or higher for maximum I/O performance for wide area network transfers](http://fasterdata.es.net/host-tuning/).

### Pulpos settings
We use the following kernel parameters (`/etc/sysctl.d/10g.conf`) on 10G hosts, which are based upon [ESnet's recommendations for a host with a 10G NIC optimized for network paths up to 200ms RTT](http://fasterdata.es.net/host-tuning/linux/):
{% highlight conf %}
# http://fasterdata.es.net/host-tuning/linux/
# allow testing with buffers up to 128MB
net.core.rmem_max = 134217728
net.core.wmem_max = 134217728
# increase Linux autotuning TCP buffer limit to 64MB
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
# recommended default congestion control is htcp
net.ipv4.tcp_congestion_control=htcp
# recommended for hosts with jumbo frames enabled
net.ipv4.tcp_mtu_probing=1
# recommended for CentOS7/Debian8 hosts
net.core.default_qdisc = fq
{% endhighlight %}

We use the following kernel parameters (`/etc/sysctl.d/40g.conf`) on 40G hosts, which are based upon [ESnet's recommendations for a host with a Mellanox ConnectX-3 NIC](http://fasterdata.es.net/host-tuning/nic-tuning/mellanox-connectx-3/):
{% highlight conf %}
# http://fasterdata.es.net/host-tuning/nic-tuning/mellanox-connectx-3/
# allow testing with buffers up to 256MB
net.core.rmem_max = 268435456
net.core.wmem_max = 268435456
# increase Linux autotuning TCP buffer limit to 128MB
net.ipv4.tcp_rmem = 4096 87380 134217728
net.ipv4.tcp_wmem = 4096 65536 134217728
# increase the length of the processor input queue
net.core.netdev_max_backlog = 250000
# recommended default congestion control is htcp
net.ipv4.tcp_congestion_control=htcp
# recommended for hosts with jumbo frames enabled
net.ipv4.tcp_mtu_probing=1
# recommended for CentOS7/Debian8 hosts
net.core.default_qdisc = fq
{% endhighlight %}

To load the configuration file, execute:
{% highlight shell_session %}
# sysctl -p /etc/sysctl.d/40g.conf
{% endhighlight %}

## Mellanox ConnectX-3
Each OSD node has a single-port [Mellanox ConnectX-3 Pro 10/40/56GbE Adapter](http://www.mellanox.com/page/products_dyn?product_family=162), showing up as `ens2` in CentOS 7.
{% highlight shell_session %}
# lspci | grep -i Mellanox
02:00.0 Ethernet controller: Mellanox Technologies MT27520 Family [ConnectX-3 Pro]
{% endhighlight %}

Esnet recommends using the [latest device driver from Mellanox](http://www.mellanox.com/page/products_dyn?product_family=27) rather than the one that comes with CentOS 7.

Install pre-requisites:
{% highlight shell_session %}
# tentakel -g osd yum -y install lsof libxml2-python
{% endhighlight %}

Install Mellanox EN Driver for Linux on the OSD nodes:
{% highlight shell_session %}
# tar xvfz mlnx-en-4.1-1.0.2.0-rhel7.4-x86_64.tgz
# cd mlnx-en-4.1-1.0.2.0-rhel7.4-x86_64
# ./install
{% endhighlight %}

Contrary to what is said in [ESnet article on Mellanox ConnectX-3](http://fasterdata.es.net/host-tuning/nic-tuning/mellanox-connectx-3/), the driver installation script didn't add anything to `/etc/sysctl.conf`, nor to `/etc/sysctl.d/`.

Furthermore, modify `/etc/modprobe.d/mlx4.conf` & `/etc/rc.d/rc.local` according to [Mellanox guide on ConnectX-3/Pro Tuning For Linux](https://community.mellanox.com/docs/DOC-2819):
{% highlight shell_session %}
# cat /sys/class/net/ens2/device/numa_node
0
# set_irq_affinity_bynode.sh 0 ens2
{% endhighlight %}

### Speed tests
**1) iperf3**

Start server:
{% highlight shell_session %}
[root@pulpo-osd01 ~]# iperf3 -s
{% endhighlight %}

Start client:
{% highlight shell_session %}
[root@pulpo-osd02 ~]# iperf3 -c pulpo-osd01.cluster -i 1 -t 10 -V
iperf 3.1.7
Linux pulpo-osd02.ucsc.edu 3.10.0-693.2.2.el7.x86_64 #1 SMP Tue Sep 12 22:26:13 UTC 2017 x86_64
Control connection MSS 8948
Time: Wed, 11 Oct 2017 22:57:37 GMT
Connecting to host pulpo-osd01.cluster, port 5201
      Cookie: pulpo-osd02.ucsc.edu.1507762657.0888
      TCP MSS: 8948 (default)
[  4] local 192.168.40.6 port 48812 connected to 192.168.40.5 port 5201
Starting Test: protocol: TCP, 1 streams, 131072 byte blocks, omitting 0 seconds, 10 second test
[ ID] Interval           Transfer     Bandwidth       Retr  Cwnd
[  4]   0.00-1.00   sec  3.13 GBytes  26.9 Gbits/sec    0   1.17 MBytes
[  4]   1.00-2.00   sec  3.25 GBytes  27.9 Gbits/sec    0   1.31 MBytes
[  4]   2.00-3.00   sec  3.24 GBytes  27.8 Gbits/sec    0   1.31 MBytes
[  4]   3.00-4.00   sec  3.24 GBytes  27.8 Gbits/sec    0   1.32 MBytes
[  4]   4.00-5.00   sec  3.25 GBytes  27.9 Gbits/sec    0   1.35 MBytes
[  4]   5.00-6.00   sec  3.27 GBytes  28.1 Gbits/sec    0   1.41 MBytes
[  4]   6.00-7.00   sec  3.23 GBytes  27.7 Gbits/sec    0   1.41 MBytes
[  4]   7.00-8.00   sec  3.19 GBytes  27.4 Gbits/sec    0   1.43 MBytes
[  4]   8.00-9.00   sec  3.25 GBytes  27.9 Gbits/sec    0   1.43 MBytes
[  4]   9.00-10.00  sec  3.24 GBytes  27.9 Gbits/sec    0   1.43 MBytes
- - - - - - - - - - - - - - - - - - - - - - - - -
Test Complete. Summary Results:
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  32.3 GBytes  27.7 Gbits/sec    0             sender
[  4]   0.00-10.00  sec  32.3 GBytes  27.7 Gbits/sec                  receiver
CPU Utilization: local/sender 36.3% (0.5%u/35.9%s), remote/receiver 11.8% (0.2%u/11.6%s)
snd_tcp_congestion htcp
rcv_tcp_congestion htcp

iperf Done.
{% endhighlight %}
Better; but far from line rate!

**2) nuttcp**

Start server (one shot):
{% highlight shell_session %}
[root@pulpo-osd01 ~]# nuttcp -1
{% endhighlight %}

Start client:
{% highlight shell_session %}
[root@pulpo-osd02 ~]# nuttcp -i1 pulpo-osd01.cluster
 3209.6875 MB /   1.00 sec = 26924.4871 Mbps     0 retrans
 3305.1250 MB /   1.00 sec = 27724.8712 Mbps     0 retrans
 3368.3750 MB /   1.00 sec = 28256.3448 Mbps     0 retrans
 3052.5625 MB /   1.00 sec = 25607.1599 Mbps     0 retrans
 3052.6875 MB /   1.00 sec = 25607.6195 Mbps     0 retrans
 3053.5000 MB /   1.00 sec = 25614.4608 Mbps     0 retrans
 3049.8750 MB /   1.00 sec = 25584.6663 Mbps     0 retrans
 3108.3750 MB /   1.00 sec = 26074.8612 Mbps     0 retrans
 3125.7500 MB /   1.00 sec = 26220.5341 Mbps     0 retrans
 3091.7500 MB /   1.00 sec = 25935.3750 Mbps     0 retrans

31443.2500 MB /  10.01 sec = 26360.8014 Mbps 43 %TX 89 %RX 0 retrans 0.21 msRTT
{% endhighlight %}
Also better; but far from line rate!

**Note**
1. I tried to change core affinity of the iperf3/nuttcp processes, but didn't observe an improved performance;
2. I set *cpufreq governor* to *performance* (see [40G/100G Tuning](http://fasterdata.es.net/host-tuning/100g-tuning/)), but didn't observe an improved performance either:
{% highlight shell_session %}
# cpupower frequency-set -g performance
{% endhighlight %}

### weak_module?
Philip Papadopoulos suggested the lackluster performance might be due to the fact that driver had been installed as a weak module:
{% highlight shell_session %}
[root@pulpo-osd01 ~]# modinfo mlx4_core
filename:       /lib/modules/3.10.0-693.2.2.el7.x86_64/weak-updates/mlnx-en/drivers/net/ethernet/mellanox/mlx4/mlx4_core.ko
version:        4.1-1.0.2
license:        Dual BSD/GPL
description:    Mellanox ConnectX HCA low-level driver
author:         Roland Dreier
rhelversion:    7.4
{% endhighlight %}
**weak-updates** in the driver path is the smoking gun. So let's [build Mellanox EN Driver for the current kernel](https://community.mellanox.com/docs/DOC-2434?forceNoRedirect=true).

Install the prerequisite packages:
{% highlight shell_session %}
# yum -y groupinstall "Development Tools"
# yum -y install createrepo
# yum -y install kernel-devel
{% endhighlight %}

Uninstall Mellanox EN Driver:
{% highlight shell_session %}
# ./uninstall.sh
{% endhighlight %}

Compile and install Mellanox EN Driver with kernel support:
{% highlight shell_session %}
# ./install --add-kernel-support
{% endhighlight %}

Load the new driver:
{% highlight shell_session %}
# /etc/init.d/mlnx-en.d restart

# modinfo mlx4_core
filename:       /lib/modules/3.10.0-693.2.2.el7.x86_64/extra/mlnx-en/drivers/net/ethernet/mellanox/mlx4/mlx4_core.ko
version:        4.1-1.0.2
license:        Dual BSD/GPL
description:    Mellanox ConnectX HCA low-level driver
author:         Roland Dreier
rhelversion:    7.4
{% endhighlight %}
The driver was successfully installed. However, it doesn't appear to improve performance either! I'll look further into this when I get a chance.
