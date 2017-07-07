---
layout: post
title: Ubuntu in Hyper-V
tags: [Hyper-V, Windows 10, Linux]
---

In this post, I describe how I installed Ubuntu in [Hyper-V](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/index), on my Dell XPS 15 9560 laptop running Windows 10 Pro Creators Update as of this writing. <!-- more -->

## Enable Hyper-V
First we need to make sure Hyper-V is enabled. In my case, Hyper-V was enabled automatically when I [installed Docker for Windows](https://docs.docker.com/docker-for-windows/install/). Otherwise, it is easy to [enable Hyper-V](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v).

<p class="note">The Hyper-V role <em>cannot</em> be installed on Windows 10 Home.</p>

## Download Ubuntu ISO
Here I downloaded the ISO for [Ubuntu Desktop](https://www.ubuntu.com/download/desktop) 16.04.2 LTS.

## Create a Hyper-V Switch
<p class="note">When Docker for Windows was installed, it created an internal virtual switch named <strong>DockerNAT</strong>. However, it appears that <strong>DockerNAT</strong> was created for the exclusive use by the Docker virtual machine <strong>MobyLinuxVM</strong> and Docker containers. If I connect my own virtual machine to <strong>DockerNAT</strong>, I won't be able to get network connection in my virtual machine. So I had to create a new Hyper-V virtual switch. In this case, I opt to create another internal virtual switch, with network address translation (NAT).</p>

* Open **Hyper-V Manager**;
* In Hyper-V Manager, click **Virtual Switch Manager...** in the right hand **Actions** menu;
* In Virtual Switch Manager, select **Internal** as the type of virtual switch to create, then click the button `Create Virtual Switch`;
* Give the virtual switch a proper **name**&mdash;I used **InternalNAT**&mdash;then click the button `OK`.

The above steps created a new **Hyper-V Virtual Ethernet Adapter** (NIC) named **vEthernet (InternalNAT)** (see Control Panel > Network and Internet > Network Connections). Alternatively, one can create the internal virtual switch, in one single step, in a PowerShell console with elevated permission:
{% highlight powershell %}
PS C:\WINDOWS\system32> New-VMSwitch -SwitchName "InternalNAT" -SwitchType Internal
{% endhighlight %}

* Assign an IP address to the virtual NIC:
{% highlight powershell %}
PS C:\WINDOWS\system32> New-NetIPAddress -IPAddress 192.168.100.1 -PrefixLength 24 -InterfaceAlias "vEthernet (InternalNAT)"

IPAddress         : 192.168.100.1
InterfaceIndex    : 45
InterfaceAlias    : vEthernet (InternalNAT)
AddressFamily     : IPv4
Type              : Unicast
PrefixLength      : 24
PrefixOrigin      : Manual
SuffixOrigin      : Manual
AddressState      : Tentative
ValidLifetime     : Infinite ([TimeSpan]::MaxValue)
PreferredLifetime : Infinite ([TimeSpan]::MaxValue)
SkipAsSource      : False
PolicyStore       : ActiveStore

IPAddress         : 192.168.100.1
InterfaceIndex    : 45
InterfaceAlias    : vEthernet (InternalNAT)
AddressFamily     : IPv4
Type              : Unicast
PrefixLength      : 24
PrefixOrigin      : Manual
SuffixOrigin      : Manual
AddressState      : Invalid
ValidLifetime     : Infinite ([TimeSpan]::MaxValue)
PreferredLifetime : Infinite ([TimeSpan]::MaxValue)
SkipAsSource      : False
PolicyStore       : PersistentStore
{% endhighlight %}

* Configure [NAT on the virtual switch](https://www.petri.com/using-nat-virtual-switch-hyper-v):
{% highlight powershell %}
PS C:\WINDOWS\system32> New-NetNAT -Name "NATNetwork" -InternalIPInterfaceAddressPrefix 192.168.100.0/24

Name                             : NATNetwork
ExternalIPInterfaceAddressPrefix :
InternalIPInterfaceAddressPrefix : 192.168.100.0/24
IcmpQueryTimeout                 : 30
TcpEstablishedConnectionTimeout  : 1800
TcpTransientConnectionTimeout    : 120
TcpFilteringBehavior             : AddressDependentFiltering
UdpFilteringBehavior             : AddressDependentFiltering
UdpIdleSessionTimeout            : 120
UdpInboundRefresh                : False
Store                            : Local
Active                           : True
{% endhighlight %}

There is no native DHCP server on Windows 10&mdash;although one can install the freeware [DHCP Server for Windows](http://www.dhcpserver.de/cms/)&mdash;so I'll simply use static IP addresses on the Hyper-V virtual machines.

## Create an Ubuntu Virtual Machine with Hyper-V
1. Open Hyper-V Manager
2. In Hyper-V Manager, click **Quick Create...** in the right hand **Actions** menu. Quick Create was introduced in Windows 10 Creators Update.
3. In Create Virtual Machine, select **InternalNAT** for **Network**

The rest of the installation is pretty straightforward...

### Assign a static IP address
Once the installation is completed and the VM is rebooted, modify `/etc/network/interfaces` to assign 192.168.100.2/24 to **eth0**:
{% highlight conf %}
# interfaces(5) file used by ifup(8) and ifdown(8)
auto lo
iface lo inet loopback
auto eth0
iface eth0 inet static
    address 192.168.100.2
    netmask 255.255.255.0
    gateway 192.168.100.1
    dns-nameserver 8.8.8.8
{% endhighlight %}

then bring up **eth0**:
{% highlight shell %}
dong@ubuntu-16-04:~$ sudo ifup eth0
{% endhighlight %}

### Install OpenSSH server
{% highlight shell %}
dong@ubuntu-16-04:~$ sudo apt update
dong@ubuntu-16-04:~$ sudo apt install openssh-server
{% endhighlight %}

### Install the virtual HWE kernel
Virtual Hardware Enablement (HWE) kernel is a kernel suitable for use as a guest virtual machine. The virtual HWE kernel will load fewer drivers and may boot faster and have less memory overhead than a generic image (see [Supported Ubuntu virtual machines on Hyper-V](https://docs.microsoft.com/en-us/windows-server/virtualization/hyper-v/supported-ubuntu-virtual-machines-on-hyper-v)).
{% highlight shell %}
dong@ubuntu-16-04:~$ sudo apt install linux-virtual-lts-xenial
{% endhighlight %}

### Install tools and daemons for use with VMs
{% highlight shell %}
dong@ubuntu-16-04:~$ sudo apt install linux-tools-virtual-lts-xenial linux-cloud-tools-virtual-lts-xenial
{% endhighlight %}

### Change the resolution
The resolution is set at 1152x864 (4:3), and can't be changed in **Settings** > **Displays**. To change the resolution, we have to modify `/etc/default/grub`.

Change the following line:
{% highlight conf %}
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
{% endhighlight %}
to:
{% highlight conf %}
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash video=hyperv_fb:1600x900"
{% endhighlight %}

Then in a terminal, run:
{% highlight shell %}
dong@ubuntu-16-04:~$ sudo update-grub
{% endhighlight %}

Reboot. The resolution will be changed to 1600x900 (16:9).
