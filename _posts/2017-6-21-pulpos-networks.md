---
layout: post
title: Pulpos Networks
tags: [Ceph, Storage, Network]
---

In this post, we describe the network specifications and configurations of the Ceph storage cluster **Pulpos**.<!-- more -->

## Network Topology
![Pulpos Network Topology](/images/pulpos_networks.png)

## IPMI Network
7 ports (7-13) on sw7175-f4-01
non-DCO console network
vlan802
128.114.87.128/25

gateway: 128.114.87.254

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

port 1-7 on the Pica8 switch, VLAN 436

128.114.86.0/24 subnet,

the default gateway:128.114.86.254

IPv6 Gateway is 2607:f5f0:100:1::1/64

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

Assigned IP range 172.16.60.0/24 - non-routable, VLAN 874 local to sw7175-f4-01, ports 1/1/14 - 20, enabled and configured.

| Node  | LAN1 MAC Address  | IP Address  | LAN2 MAC Address  |
| :---: |:-----------------:| :----------:| :---------------: |
| admin | 0C:C4:7A:92:98:D6 | 192.168.1.1 | 0C:C4:7A:92:98:D6 |
| dtn   | 0C:C4:7A:92:99:4A | 192.168.1.2 | 0C:C4:7A:92:99:4B |
| mon   | 0C:C4:7A:92:99:10 | 192.168.1.3 | 0C:C4:7A:92:99:11 |
| mds   | 0C:C4:7A:92:98:F2 | 192.168.1.4 | 0C:C4:7A:92:98:F3 |
| osd01 | 0C:C4:7A:28:44:5E | 192.168.1.5 | 0C:C4:7A:28:44:5F |
| osd02 | 0C:C4:7A:28:44:1E | 192.168.1.6 | 0C:C4:7A:28:44:1E |
| osd03 | 0C:C4:7A:28:4D:5A | 192.168.1.7 | 0C:C4:7A:28:4D:5A |

## Cluster Network

40g, port 50 - 52 on Pica8, VLAN 4000

| Node  | MAC Address       | IP Address   |
| :---: |:-----------------:| :-----------:|
| osd01 | 0C:C4:7A:28:44:?? | 192.168.40.5 |
| osd02 | 0C:C4:7A:28:44:?? | 192.168.40.6 |
| osd03 | 0C:C4:7A:28:4D:?? | 192.168.40.7 |
