---
layout: post
title: Pulpos Networks
tags: [Ceph, Storage, Network]
---

In this post, we describe the network specifications and configurations of the Ceph storage cluster **Pulpos**.<!-- more -->

## Network Topology
![Pulpos Network Topology](/images/pulpos_networks.png)

## IPMI Network
Each node in the **Pulpos** cluster has an on-board [Baseboard Management Controller](https://www.supermicro.com/products/nfo/IPMI.cfm) (BMC), with a dedicated 1G NIC. These NICs are connected to ports 7-13, respectively, of the top-of-the-rack switch `sw7175-f4-01`. The switch is a [Brocade ICX6610-24](http://www.brocade.com/en/products-services/switches/campus-network-switches/icx-6610-switch.html)), with 8x 10GE SFP+ ports and 24x 1000BASE-T ports. The VLAN ID for ports 7-13 is 802. The subnet for the IPMI network is 128.114.87.128/25; and the gateway IP is 128.114.87.254. The subnet is part of UCSC's non-DCO console network, and can only be accessed via the [Data Center VPN](https://its.ucsc.edu/vpn/).

| Node  | Hostname                  | MAC Address       | IP Address     |
| :---: |:-------------------------:| :----------------:| :------------: |
| admin | pulpo-admin-ipmi.ucsc.edu | 0C:C4:7A:92:94:55 | 128.114.87.133 |
| dtn   | pulpo-dtn-ipmi.ucsc.edu   | 0C:C4:7A:92:94:8F | 128.114.87.134 |
| mon   | pulpo-mon01-ipmi.ucsc.edu | 0C:C4:7A:92:94:72 | 128.114.87.135 |
| mds   | pulpo-mds01-ipmi.ucsc.edu | 0C:C4:7A:92:94:63 | 128.114.87.136 |
| osd01 | pulpo-osd01-ipmi.ucsc.edu | 0C:C4:7A:29:17:3A | 128.114.87.137 |
| osd02 | pulpo-osd02-ipmi.ucsc.edu | 0C:C4:7A:29:17:1A | 128.114.87.138 |
| osd03 | pulpo-osd03-ipmi.ucsc.edu | 0C:C4:7A:29:1B:B2 | 128.114.87.139 |

## Public Network
Each node has an [Intel X520-DA2 10GbE adapter](http://ark.intel.com/products/39776/Intel-Ethernet-Converged-Network-Adapter-X520-DA2), with 2 SFP+ ports. On the [2U TwinPro](https://www.supermicro.com/products/nfo/2UTwinPro.cfm) nodes (admin, dtn, mon & mds), the first port of the Intel X520-DA2 10GbE adapter shows up as `ens1f0`, and the second as `ens1f1`, in CentOS 7. By contrast, on OSD nodes (osd01, osd02 & osd03), the first port of the Intel X520-DA2 10GbE adapter shows up as `ens5f0`, and the second as `ens5f1`, in CentOS 7. Only the first ports of each adapter are used, as the public interfaces of the Pulpos cluster. These ports are connected to ports 1-7 of our whitebox switch `sw7175-100-pica8-1`, which is a [QuantaMesh BMS T3048-LY2](https://www.qct.io/product/index/Networking/Bare-Metal-Switch/Leaf-Switch/QuantaMesh-BMS-T3048-LY2) and runs Pica8 [PicOS](http://www.pica8.com/products/picos). The VLAN ID for ports 1-7 is 436. The IPv4 subnet for the public network is 128.114.86.0/24; and IPv6 subnet is 2607:f5f0:100:1::1/64. The IPv4 gateway is 128.114.86.254 and IPv6 Gateway is 2607:f5f0:100:1::1.

Below are the configurations for the first 10GE ports on each node:

| Node  | Hostname             | IPv4 Address | IPv6 Address        |
| :---: |:--------------------:| :----------: | :-----------------: |
| admin | pulpo-admin.ucsc.edu | 128.114.86.2 | 2607:f5f0:100:1::10 |
| dtn   | pulpo-dtn.ucsc.edu   | 128.114.86.3 | 2607:f5f0:100:1::11 |
| mon   | pulpo-mon01.ucsc.edu | 128.114.86.4 | 2607:f5f0:100:1::12 |
| mds   | pulpo-mds01.ucsc.edu | 128.114.86.5 | 2607:f5f0:100:1::13 |
| osd01 | pulpo-osd01.ucsc.edu | 128.114.86.6 | 2607:f5f0:100:1::14 |
| osd02 | pulpo-osd02.ucsc.edu | 128.114.86.7 | 2607:f5f0:100:1::15 |
| osd03 | pulpo-osd03.ucsc.edu | 128.114.86.8 | 2607:f5f0:100:1::16 |


## Control Network
Each node has two on-board Intel I350 GE NICs. On the [2U TwinPro](https://www.supermicro.com/products/nfo/2UTwinPro.cfm) nodes (admin, dtn, mon & mds), the first Intel I350 GE NIC shows up as `eno1`, and the second as `eno2`, in CentOS 7. By contrast, on OSD nodes (osd01, osd02 & osd03), the first Intel I350 GE NIC shows up as `enp4s0f0`, and the second as `enp4s0f1`, in CentOS 7. Only the first Intel I350 GE NICs are used, as the control interfaces of the Pulpos cluster. These ports are connected to ports 14-20 of the top-of-the-rack switch `sw7175-f4-01`. The VLAN ID for ports 14-20 is 874. The subnet for the control network is 192.168.1.0/24.

Below are the configurations for the first GE NIC on each node:

| Node  | LAN1 MAC Address  | IP Address  |
| :---: |:-----------------:| :----------:|
| admin | 0C:C4:7A:92:98:D6 | 192.168.1.1 |
| dtn   | 0C:C4:7A:92:99:4A | 192.168.1.2 |
| mon   | 0C:C4:7A:92:99:10 | 192.168.1.3 |
| mds   | 0C:C4:7A:92:98:F2 | 192.168.1.4 |
| osd01 | 0C:C4:7A:28:44:5E | 192.168.1.5 |
| osd02 | 0C:C4:7A:28:44:1E | 192.168.1.6 |
| osd03 | 0C:C4:7A:28:4D:5A | 192.168.1.7 |

## Cluster Network
Each OSD node has an additional single-port [Mellanox ConnectX-3 Pro 10/40/56GbE Adapter](http://www.mellanox.com/page/products_dyn?product_family=162), showing up as `ens2` in CentOS 7. These adapters are connected to the 40GbE ports 50-52 of our whitebox switch `sw7175-100-pica8-1`. The VLAN ID of those switch ports is 4000.

| Node  | MAC Address       | IP Address   |
| :---: |:-----------------:| :-----------:|
| osd01 | e4:1d:2d:17:8d:e0 | 192.168.40.5 |
| osd02 | e4:1d:2d:17:8d:10 | 192.168.40.6 |
| osd03 | f4:52:14:4c:74:30 | 192.168.40.7 |
