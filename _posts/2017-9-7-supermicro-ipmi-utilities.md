---
layout: post
title: Supermicro IPMI Utilities
tags: [Provisioning, Security]
---

Supermicro provides a set of [utilities for configuring IPMI devices](https://www.supermicro.com/solutions/SMS_IPMI.cfm) on Supermicro servers.<!-- more -->

* Table of Contents
{:toc}

## IPMICFG
IPMICFG is an *in-band* utility for configuring IPMI devices. It is a command line tool providing standard IPMI and Supermicro proprietary OEM commands. This CLI-based utility can be executed on DOS, Windows, and Linux OS and does *not* require any installation procedures.

We can use *IPMICFG* (or the standard *ipmitool*) to mitigate the serious [Supermicro IPMI Vulnerability]({{ site.baseurl }}{% post_url 2017-9-7-supermicro-ipmi-vulnerability %}).
1. To disable DHCP for the IPMI interface:
{% highlight shell_session %}
./IPMICFG-Linux.x86_64 -dhcp off
{% endhighlight %}
2. To set the IPMI interface's IP address to *0.0.0.0*:
{% highlight shell_session %}
./IPMICFG-Linux.x86_64 -m 0.0.0.0
{% endhighlight %}

We can also [change the option of the IPMI interface](https://siliconmechanics.zendesk.com/hc/en-us/articles/201123119-Changing-NIC-failover-mode), using *raw* codes.
* The default is **failover**:
{% highlight shell_session %}
./IPMICFG-Linux.x86_64 -raw 0x30 0x70 0x0c 0
02
{% endhighlight %}
* Set it to **dedicated**:
{% highlight shell_session %}
./IPMICFG-Linux.x86_64 -raw 0x30 0x70 0x0c 1 0
{% endhighlight %}
* Verify it:
{% highlight shell_session %}
./IPMICFG-Linux.x86_64 -raw 0x30 0x70 0x0c 0
00
{% endhighlight %}

## SMCIPMITool
SMCIPMITool is an **out-of-band** Supermicro utility that allows a user to interface with SuperBlade systems and IPMI devices via CLI (Command Line Interface). Two kinds of user modes are provided when you start the SMCIMPITool: **Command Line** Mode and **Shell** Mode.

SMCIPMITool Key Features:
* Remote IPMI Management
* Remote NM (Node Manager) 2.0 Management
* Remote IPMI Sensor and Event Management
* Remote FRU Management
* Remote IPMI User/Group Management
* Remote Blade System Management
* IPMI Firmware Upgrade
* Virtual Media Management

Tips:
* To set boot device to be PXE in next boot:
{% highlight shell_session %}
ipmi power bootoption 1
{% endhighlight %}
* To reset the system and force PXE as the boot device in the next boot only:
{% highlight shell_session %}
ipmi power reset PXE
{% endhighlight %}

## IPMIView
IPMIView is a GUI-based software application that allows administrators to manage multiple target systems through BMC. IPMIView monitors and reports on the status of a SuperBlade system, including the blade server, power supply, gigabit switch, InfiniBand and CMM modules. IPMIView also supports remote KVM and Virtual Media.

IPMIView Key Features:
* IPMI System Management
* KVM Console Redirection
* Text Console Redirection
* Virtual Media Management
* IPMI User/Group Management
* Trap Receiver
* Mobile App (Android, iOS)

Tips:
* On Windows, run IPMIView as *administrator*;
* To have a better experience running IPMIView on a Windows 10 system with a High DPI display, right click IPMIView20 icon, choose **Properties** from the menu, go to the **Compatibility** tab, check "Override high DPI scaling behavior. Scaling performed by:", and select **System**.
