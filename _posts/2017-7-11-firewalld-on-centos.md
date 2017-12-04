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

## firewall-cmd
**firewall-cmd** is the command line client of the FirewallD. It provides interface to manage runtime and permanent configuration.
{% highlight shell_session %}
[root@pulpo-admin ~]# firewall-cmd --state
running
{% endhighlight %}
<p class="note">Firewalld uses two configuration sets: Runtime and Permanent. Runtime configuration changes are not retained on reboot or upon restarting FirewallD whereas permanent changes are not applied to a running system.</p>

## Zones
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

## Fail2ban
[Fail2ban](https://www.fail2ban.org/wiki/index.php/Main_Page) scans log files and bans IP addresses that makes too many password failures. It updates firewall rules to reject the IP address. Fail2Ban can read multiple log files such as sshd or Apache web server ones.

Fail2ban packages are available from the EPEL repository. To install Fail2ban with support for manipulating firewalld rules (Ref: [Install Fail2ban on Centos 7 to Protect SSH via firewalld](https://devops.profitbricks.com/tutorials/install-fail2ban-on-centos-7-to-protect-ssh-via-firewalld/)):
{% highlight shell_session %}
# yum install -y fail2ban-firewalld
{% endhighlight %}

To configure Fail2ban, add a file `/etc/fail2ban/jail.local` with the following contents:
{% highlight conf %}
[sshd]
enabled = true
maxretry = 3
bantime = 86400
{% endhighlight %}

Enable and start the `fail2ban` service:
{% highlight shell_session %}
# systemctl enable fail2ban
Created symlink from /etc/systemd/system/multi-user.target.wants/fail2ban.service to /usr/lib/systemd/system/fail2ban.service.
# systemctl start fail2ban
{% endhighlight %}

Verify that the *sshd* jail has indeed been enabled:
{% highlight shell_session %}
# fail2ban-client status
Status
|- Number of jail:      1
`- Jail list:   sshd
{% endhighlight %}

Verify that Fail2ban has added a FirewallD rule to block IP addresses that are generating failed login attempts:
{% highlight shell_session %}
# firewall-cmd --direct --get-all-rules
ipv4 filter INPUT 0 -p tcp -m multiport --dports ssh -m set --match-set fail2ban-sshd src -j REJECT --reject-with icmp-port-unreachable
{% endhighlight %}
An *ipset* list called `fail2ban-sshd` has been created. We can list the current contents of the list:
{% highlight shell_session %}
# ipset list fail2ban-sshd
Name: fail2ban-sshd
Type: hash:ip
Revision: 1
Header: family inet hashsize 1024 maxelem 65536 timeout 86400
Size in memory: 16592
References: 1
Members:
201.249.207.212 timeout 86393
{% endhighlight %}
or
{% highlight shell_session %}
# fail2ban-client status sshd
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     1
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   201.249.207.212
{% endhighlight %}

## References
* [Introduction to FirewallD on CentOS](https://www.linode.com/docs/security/firewalls/introduction-to-firewalld-on-centos)
* [RHEL7: How to get started with Firewalld](https://www.certdepot.net/rhel7-get-started-firewalld/)
* [Install Fail2ban on Centos 7 to Protect SSH via firewalld](https://devops.profitbricks.com/tutorials/install-fail2ban-on-centos-7-to-protect-ssh-via-firewalld/)
