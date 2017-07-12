---
layout: post
title: FirewallD on CentOS
tags: [Security, Linux]
---

[FirewallD](http://www.firewalld.org/) is a frontend controller / wrapper for **iptables** used to implement persistent network traffic rules.<!-- more --> Working with FirewallD has two main differences compared to directly controlling iptables (Ref: [Introduction to FirewallD on CentOS](https://www.linode.com/docs/security/firewalls/introduction-to-firewalld-on-centos)):
1. FirewallD uses zones and services instead of chain and rules.
2. It manages rulesets dynamically, allowing updates without breaking existing sessions and connections.

FirewallD is included by default with CentOS 7 and is enabled by default.
{% highlight shell %}
[root@pulpo-admin ~]# systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
   Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; vendor preset: enabled)
   Active: active (running) since Tue 2017-07-11 14:12:36 PDT; 34min ago
     Docs: man:firewalld(1)
 Main PID: 10906 (firewalld)
   CGroup: /system.slice/firewalld.service
           └─10906 /usr/bin/python -Es /usr/sbin/firewalld --nofork --nopid

Jul 11 14:12:35 pulpo-admin.ucsc.edu systemd[1]: Starting firewalld - dynamic firewall daemon...
Jul 11 14:12:36 pulpo-admin.ucsc.edu systemd[1]: Started firewalld - dynamic firewall daemon.
{% endhighlight %}

**firewall-cmd** is the command line client of the FirewallD. It provides interface to manage runtime and permanent configuration.
{% highlight shell %}
[root@pulpo-admin ~]# firewall-cmd --state
running
{% endhighlight %}
<p class="note">Firewalld uses two configuration sets: Runtime and Permanent. Runtime configuration changes are not retained on reboot or upon restarting FirewallD whereas permanent changes are not applied to a running system.</p>

Right after the minimal install of CentOS on [pulpo-admin]({{ site.baseurl }}{% post_url 2017-7-10-pulpo-admin %}), the default zone is **public**:
{% highlight shell %}
[root@pulpo-admin ~]# firewall-cmd --get-default-zone
public
{% endhighlight %}
and all active network interfaces are bound to the **public** zone:
{% highlight shell %}
[root@pulpo-admin ~]# firewall-cmd --get-active-zones
public
  interfaces: eno1 ens1f0
{% endhighlight %}

To list all zones and their configurations:
{% highlight shell %}
[root@pulpo-admin ~]# firewall-cmd --get-zones
work drop internal external trusted home dmz public block

[root@pulpo-admin ~]# firewall-cmd --list-all-zones
{% endhighlight %}

Let's remove the GbE interface **eno1** from the **public** zone and bind it to the **trusted** zone (Ref: [RHEL7: How to get started with Firewalld](https://www.certdepot.net/rhel7-get-started-firewalld/)):
{% highlight shell %}
[root@pulpo-admin ~]# firewall-cmd --permanent --zone=trusted --change-interface=eno1
The interface is under control of NetworkManager, setting zone to 'trusted'.
success
{% endhighlight %}
But that is not enough. The interface is under control of **NetworkManager**. Be default, all interfaces are bound to the default zone, which is **public**. To really *permanently* bind **eno1** to the **trusted** zone, we need to modify NetworkManager configuration. We can use [nmcli](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Networking_Guide/sec-Using_the_NetworkManager_Command_Line_Tool_nmcli.html), Network Manager Command Line Interface, to do so.
{% highlight shell %}
[root@pulpo-admin ~]# nmcli con show
NAME    UUID                                  TYPE            DEVICE
eno1    1a641f99-21ff-4324-9166-963e063c2c7b  802-3-ethernet  eno1
ens1f0  bdc5d857-140d-48e2-a953-7d422e2405ba  802-3-ethernet  ens1f0
eno2    3daeaadd-aaab-4a04-89ad-5f41e2f4e859  802-3-ethernet  --
ens1f1  b8522386-019a-4463-83c2-67bbe6216243  802-3-ethernet  --

[root@pulpo-admin ~]# nmcli con mod eno1 connection.zone trusted

[root@pulpo-admin ~]# nmcli con up eno1
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/2)
{% endhighlight %}

<p class="tip">Alternatively, we can change the zone of <strong>eno1</strong> by editing the file <strong>/etc/sysconfig/network-scripts/ifcfg-eno1</strong>. Add <code class="highlighter-rouge">ZONE=trusted</code> to the file; then run <code class="highlighter-rouge">nmcli con reload</code>.</p>

Verify that **eno1** is now bound to **trusted**:
{% highlight shell %}
[root@pulpo-admin ~]# firewall-cmd --get-zone-of-interface=eno1
trusted
{% endhighlight %}

And we now have 2 active zones:
{% highlight shell %}
[root@pulpo-admin ~]# firewall-cmd --get-active-zones
public
  interfaces: ens1f0
trusted
  interfaces: eno1
{% endhighlight %}

## References
* [Introduction to FirewallD on CentOS](https://www.linode.com/docs/security/firewalls/introduction-to-firewalld-on-centos)
* [RHEL7: How to get started with Firewalld](https://www.certdepot.net/rhel7-get-started-firewalld/)
