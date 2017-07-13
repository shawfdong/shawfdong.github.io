---
layout: post
title: Pulpos Networks
tags: [Ceph, Storage, Network]
---

In this post, we describe the network specifications and configurations of the Ceph storage cluster **Pulpos**.<!-- more -->

## Network Topology
![Pulpos Network Topology](/images/pulpos_networks.png)

## IPMI Network

| Node  | Hostname                  | MAC Address       | IP Address     |
| :---: |:-------------------------:| :----------------:| :------------: |
| admin | pulpo-admin-ipmi.ucsc.edu | 0c:c4:7a:92:94:55 | 128.114.87.133 |
| dtn   | pulpo-dtn-ipmi.ucsc.edu   | | 128.114.87.134 |
| mon   | pulpo-mon01-ipmi.ucsc.edu | | 128.114.87.135 |
| mds   | pulpo-mds01-ipmi.ucsc.edu | | 128.114.87.136 |
| osd01 | pulpo-osd01-ipmi.ucsc.edu | | 128.114.87.137 |
| osd02 | pulpo-osd02-ipmi.ucsc.edu | | 128.114.87.138 |
| osd03 | pulpo-osd03-ipmi.ucsc.edu | | 128.114.87.139 |

## Public Network

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

| Node  | LAN1 MAC Address  | IP Address     | LAN2 MAC Address  |
| :---: |:-----------------:| :-------------:| :---------------: |
| admin | 0c:c4:7a:92:98:d6 | 128.114.87.??? | 0c:c4:7a:92:98:d7 |
| dtn   | pulpo-dtn-ipmi.ucsc.edu   | |
| mon   | pulpo-mon01-ipmi.ucsc.edu | |
| mds   | pulpo-mds01-ipmi.ucsc.edu |
| osd01 | pulpo-osd01-ipmi.ucsc.edu |
| osd02 | pulpo-osd02-ipmi.ucsc.edu |
| osd03 | pulpo-osd03-ipmi.ucsc.edu |

## Cluster Network
