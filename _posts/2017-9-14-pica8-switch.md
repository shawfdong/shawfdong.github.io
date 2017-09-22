---
layout: post
title: Pica8 Switch
tags: [Network, Linux, Security]
---

A [white box switch](http://searchsdn.techtarget.com/definition/white-box-switch), [QuantaMesh BMS T3048-LY2R](https://www.qct.io/product/index/Networking/Bare-Metal-Switch/Leaf-Switch/QuantaMesh-BMS-T3048-LY2R), provides networking connectivity to the Ceph storage cluster **Pulpos**. We runs Pica8 [PicOS](http://www.pica8.com/products/picos) on the switch.<!-- more --> QuantaMesh BMS switches can also run other Network OSes, such as [Cumulus](http://cumulusnetworks.com/cumulus-linux/overview/#cl-architecture).

## Hardware Specifications
QuantaMesh BMS T3048-LY2R is a 1U top top-of-rack switch, based on the Broadcom [Trident+](https://people.ucsc.edu/~warner/Bufs/trident) ([BCM56840 Series](https://www.broadcom.com/products/Switching/Carrier-and-Service-Provider/BCM56840-Series)) switching fabric, with:
* 48x 10GbE SFP+ ports
* 4x 40GbE QSFP+ ports
* Out-of-band management port (RJ-45, 10/100/1000Base-T)

The CPU is a dual-core 32-bit [PowerPC e500v2](https://en.wikipedia.org/wiki/PowerPC_e500) @ 1.2GHz, from Freescale Semiconductor:
```shell
shaw@sw7175-100-pica8-1:~$ cat /proc/cpuinfo
processor       : 0
cpu             : e500v2
clock           : 1200.000000MHz
revision        : 5.1 (pvr 8021 1051)
bogomips        : 150.00

processor       : 1
cpu             : e500v2
clock           : 1200.000000MHz
revision        : 5.1 (pvr 8021 1051)
bogomips        : 150.00

total bogomips  : 300.00
timebase        : 75000000
platform        : P2020 RDB
model           : fsl,P2020
Memory          : 2048 MB
```
As one can see from above, total memory is 2GB.

## PicOS
As of this writing, the switch runs PicOS *2.9.1.2*, which seems to be based on Debian 7.0:
```shell
shaw@sw7175-100-pica8-1:~$ cat /etc/debian_version
7.0
```
It uses an ancient Linux kernel *2.6.32.69*:
```shell
shaw@sw7175-100-pica8-1:~$ uname -a
Linux sw7175-100-pica8-1 2.6.32.69 #1 SMP Tue May 9 06:06:19 CST 2017 ppc GNU/Linux
```

PicOS can run in 2 different modes of operation:
* **Open vSwitch (OVS) mode**: In this mode, PicOS is dedicated and optimized for [OpenFlow](https://www.opennetworking.org/sdn-resources/openflow) applications
* **Layer 2 / Layer 3 (L2/L3) mode**: In this mode, PicOS can run both switching and routing protocols (using [XORP](http://en.wikipedia.org/wiki/XORP)) and OpenFlow applications

References:
* [PicOS Overview](http://www.pica8.com/wp-content/uploads/2015/08/pica8-whitepaper-picos-overview.pdf)
* [PicOS Routing And Switching Configuration Guide](http://www.pica8.com/wp-content/uploads/2015/09/v2.9/html/picos-routing-and-switching-configuration-guide/)
* [PicOS Routing And Switching Commands Reference Guide](http://www.pica8.com/wp-content/uploads/2015/09/v2.9/html/picos-routing-and-switching-commands-reference-guide/)
* [PicOS System Configuration](http://www.pica8.com/wp-content/uploads/2015/09/v2.9/html/picos-system-configuration/)
* [PicOS OVS Configuration Guide](http://www.pica8.com/wp-content/uploads/2015/09/v2.9/html/ovs-configuration-guide/)
* [PicOS Openflow Tutorials](http://www.pica8.com/wp-content/uploads/2015/09/v2.9/html/openflow-tutorials/)

## Accessing the Pica8 switch
The Pica8 switch is heavily guarded. In order to gain remote access to the switch, one needs to go through the following steps:

1) Enter [Data Center VPN](https://its.ucsc.edu/vpn/index.html) (vpn-dc.ucsc.edu), using [Cisco AnyConnect](https://its.ucsc.edu/vpn/installation.html). Data Center VPN requires [Multi-Factor Authentication (MFA)](https://its.ucsc.edu/mfa/index.html). To authenticate to Data Center VPN, one needs to provide both the CruzID **Gold** password and a One-Time Passcode (OTP). [Duo](https://duo.com/) is the vendor that facilitates the passcodes for MFA at UCSC.

2) Once inside Data Center VPN, ssh to one of the *sebastion* hosts, using MFA (CruzID **Blue** password + OTP). There are 2 *sebastion* hosts:
* noc-prod-sebastion-1.ucsc.edu (alias: *sebastion1.ucsc.edu*)
* noc-prod-sebastion-2.ucsc.edu (alias: *sebastion2.ucsc.edu*)

```shell
$ ssh -l shaw sebastion1.ucsc.edu
Password:
Duo two-factor login for shaw

Enter a passcode or select one of the following options:

 1. Duo Push to XXX-XXX-1855
 2. Phone call to XXX-XXX-1855
 3. SMS passcodes to XXX-XXX-1855

Passcode or option (1-3): 1
setsockopt IPV6_TCLASS 16: Operation not permitted:
Success. Logging you in...
```

3) Finally, one can ssh to the Pica8 switch, whose FQDN is *sw7175-100-ve435.ucsc.edu* and whose IP address is *128.114.109.93*.
```shell
[shaw@noc-prod-sebastion-1 ~]$ ssh -l shaw 128.114.109.93
```
Curiously, although PicOS says the hostname of the switch is *sw7175-100-pica8-1*:
```shell
shaw@sw7175-100-pica8-1:~$ hostname
sw7175-100-pica8-1
```
it is not the registered DNS name of the switch!

4) One can then enter CLI, by typing `cli`:
```shell
shaw@sw7175-100-pica8-1:~$ cli
Synchronizing configuration...OK.
Pica8 PicOS Version 2.9.1.2
Welcome to PicOS on sw7175-100-pica8-1
```

## Link Aggregation
My colleague George Peeks has configured [Link Aggregation Control Protocol (LACP)](https://en.wikipedia.org/wiki/Link_aggregation#Link_Aggregation_Control_Protocol) on the Pica8 switch, in order to support [bonding of two 10GbE interfaces on pulpo-dtn]({{ site.baseurl }}{% post_url 2017-9-12-bonding-on-pulpo-dtn %}).
```shell
shaw@sw7175-100-pica8-1> show running-config
    interface {
        cut-through-mode: true
        aggregate-ethernet ae1 {
            mtu: 9216
            aggregated-ether-options {
                lacp {
                    enable: true
                }
            }
            family {
                ethernet-switching {
                    native-vlan-id: 436
                }
            }
        }
        gigabit-ethernet "te-1/1/3" {
            description: "pulpo-dtn.ucsc.edu_LAG"
            mtu: 9216
            speed: "10000"
            ether-options {
                802.3ad: "ae1"
            }
            family {
                ethernet-switching {
                    native-vlan-id: 436
                }
            }
        }
        gigabit-ethernet "te-1/1/4" {
            description: "pulpo-dtn.ucsc.edu_LAG"
            mtu: 9216
            speed: "10000"
            ether-options {
                802.3ad: "ae1"
            }
            family {
                ethernet-switching {
                    native-vlan-id: 436
                    port-mode: "trunk"
                    vlan {
                        members 435
                    }
                }
            }
        }
    }

shaw@sw7175-100-pica8-1> show interface aggregate-ethernet ae1
Physical interface: ae1, Enabled, error-discard False, Physical link is Up
Interface index: 69, Mac Learning Enabled
Description:
Link-level type: Ethernet, MTU: 9216, Speed: 20Gb/s, Duplex: Auto
Source filtering: Disabled, Flow control: Disabled, Auto-negotiation: Disabled
Interface flags: Hardware-Down SNMP-Traps Internal: 0x0
Current address: 08:9e:01:f8:63:ac, Hardware address: 08:9e:01:f8:63:ac
Traffic statistics:
  5 sec input rate 88 bits/sec, 0 packets/sec
  5 sec output rate 1896 bits/sec, 3 packets/sec
  Input Packets............................75551404
  Output Packets...........................124005461
  Input Octets.............................601186683571
  Output Octets............................1023826723563
Hash-mapping: ethernet-source-destination
Aggregated link protocol: LACP
Minimum number of selected ports: 1
  Members        Status          Port Speed
  ---------      ----------      ----------
  te-1/1/3       Up(active)      10Gb/s
  te-1/1/4       Up(active)      10Gb/s
```
