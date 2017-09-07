---
layout: post
title: Automated Installation of CentOS 7
tags: [Linux, Provisioning]
---

In this post I describe how we set up a home-brew environment for automated installation of CentOS 7 on the nodes of our Ceph storage cluster [Pulpos]({{ site.baseurl }}{% post_url 2017-2-9-pulpos %}).<!-- more --> For bigger deployment, one might want to look into a more sophisticated provisioning system like [Cobbler](http://cobbler.github.io/).

## Apache HTTP server
First we install the Apache HTTP server on [pulpo-admin]({{ site.baseurl }}{% post_url 2017-7-10-pulpo-admin %}).
1. Install Apache HTTP server:
```shell
[root@pulpo-admin ~]# yum -y install http httpd-devel httpd-tools
```
2. Enable and start httpd:
```shell
[root@pulpo-admin ~]# systemctl enable httpd
[root@pulpo-admin ~]# systemctl start httpd
  ```

## CentOS 7 Local Mirror
Next we create a [local CentOS 7 mirror](https://wiki.centos.org/HowTos/CreateLocalMirror) for network installs.
```shell
[root@pulpo-admin ~]# yum -y install rsync

[root@pulpo-admin ~]# mkdir -p /var/www/html/centos

[root@pulpo-admin ~]# rsync -avSHP --delete --exclude "local*" \
--exclude "isos" linux.mirrors.es.net::centos/7.3.1611/ \
/var/www/html/centos/7.3.1611/

[root@pulpo-admin ~]# cd /var/www/html/centos/
[root@pulpo-admin centos]# ln -fs 7.3.1611 7
```

## DHCP server
1. Install DHCP server:
```shell
[root@pulpo-admin ~]# yum -y install dhcp
```
2. Edit `/etc/dhcp/dhcpd.conf` to configure DHCP server for the control network 192.168.1.0/24 (see [Pulpos Networks]({{ site.baseurl }}{% post_url 2017-6-21-pulpos-networks %})):
```conf
ddns-update-style none;
subnet 192.168.1.0 netmask 255.255.255.0 {
        default-lease-time 1200;
        max-lease-time 1200;
        option subnet-mask 255.255.255.0;
        option broadcast-address 192.168.1.255;
        filename "pxelinux.0";
        next-server 192.168.1.1;

        group "local" {
                host dtn {
                        hardware ethernet 0C:C4:7A:92:99:4A;
                        fixed-address 192.168.1.2;
                }
                host mon {
                        hardware ethernet 0C:C4:7A:92:99:10;
                        fixed-address 192.168.1.3;
                }
                host mds {
                        hardware ethernet 0C:C4:7A:92:98:F2;
                        fixed-address 192.168.1.4;
                }
                host osd01 {
                        hardware ethernet 0C:C4:7A:28:44:5E;
                        fixed-address 192.168.1.5;
                }
                host osd02 {
                        hardware ethernet 0C:C4:7A:28:44:1E;
                        fixed-address 192.168.1.6;
                }
                host osd03 {
                        hardware ethernet 0C:C4:7A:28:4D:5A;
                        fixed-address 192.168.1.7;
                }
        }
}
```
3. Enable and start **dhcpd**:
```shell
[root@pulpo-admin ~]# systemctl enable dhcpd
[root@pulpo-admin ~]# systemctl start dhcpd
```

## TFTP server
1. Install TFTP server and [syslinux](http://www.syslinux.org/wiki/index.php?title=The_Syslinux_Project):
```shell
[root@pulpo-admin ~]# yum clean all
[root@pulpo-admin ~]# yum -y install xinetd tftp tftp-server syslinux
```
2. Configure TFTP server by editing `/etc/xinetd.d/tftp` (**Note** the standard TFTP directory is */var/lib/tftpboot*, but we'll use `/tftpboot` instead):
```conf
service tftp
{
        socket_type             = dgram
        protocol                = udp
        wait                    = yes
        user                    = root
        server                  = /usr/sbin/in.tftpd
        server_args             = -s /tftpboot
        disable                 = no
        per_source              = 11
        cps                     = 100 2
        flags                   = IPv4
}
```
3. Copy [PXELINUX](http://www.syslinux.org/wiki/index.php?title=PXELINUX) files:
```shell
[root@pulpo-admin ~]# mkdir /tftpboot
[root@pulpo-admin ~]# cp -v /usr/share/syslinux/pxelinux.0 /tftpboot/
[root@pulpo-admin ~]# cp -v /usr/share/syslinux/menu.c32 /tftpboot/
[root@pulpo-admin ~]# cp -v /usr/share/syslinux/memdisk /tftpboot
[root@pulpo-admin ~]# cp -v /usr/share/syslinux/mboot.c32 /tftpboot
[root@pulpo-admin ~]# cp -v /usr/share/syslinux/chain.c32 /tftpboot
```
4. Copy `initrd.img` and `vmlinuz` from the CentOS 7 local mirror:
```shell
[root@pulpo-admin ~]# cp -v /var/www/html/centos/7/os/x86_64/images/pxeboot/initrd.img /tftpboot/
[root@pulpo-admin ~]# cp -v /var/www/html/centos/7/os/x86_64/images/pxeboot/vmlinuz /tftpboot/
```
5. Edit `/usr/lib/systemd/system/tftp.service` so that we'll use `/tftpboot` (instead of the default */var/lib/tftpboot*) as the TFTP directory:
```shell
[root@pulpo-admin ~]# sed -i 's=/var/lib/tftpboot=/tftpboot=' \
/usr/lib/systemd/system/tftp.service
```
6. Enable and start TFTP (**Note**: because we've made changes to the unit file for *tftp.service*, we need to reload systemd manager configuration with `systemctl daemon-reload`):
```shell
[root@pulpo-admin ~]# systemctl daemon-reload
[root@pulpo-admin ~]# systemctl enable tftp
[root@pulpo-admin ~]# systemctl start tftp
```
7. Test TFTP locally:
```shell
[root@pulpo-admin ~]# tftp
(to) localhost
tftp> get menu.c32
tftp> quit
[root@pulpo-admin ~]# ls -l menu.c32
-rw-r--r-- 1 root root 55012 Jul 17 15:19 menu.c32
```
So it works!

## PXELINUX
1. Create the directory `/tftpboot/pxelinux.cfg` where PXELINUX configuration files reside:
```shell
[root@pulpo-admin ~]# mkdir /tftpboot/pxelinux.cfg
```
2. Create a local boot configuration file (`/tftpboot/pxelinux.cfg/localboot`) with the following content:
```conf
default centos7
prompt 0
label centos7
        localboot 0
```
3. Make localboot the *default* boot configuration:
```shell
[root@pulpo-admin ~]# cd /tftpboot/pxelinux.cfg
[root@pulpo-admin pxelinux.cfg]# ln -s localboot default
```
4. Create a PXELINUX configuration file for each IP address in the control subnet 192.168.1.0/24. For example, the private IP address of *pulpo-dtn* is `192.168.1.2`, its [PXELINUX](http://www.syslinux.org/wiki/index.php?title=PXELINUX) configuration filename is thus `C0A80102` (uppercase hexadecimal value of the IP address); and the content of `/tftpboot/pxelinux.cfg\C0A80102` is (see [RHEL7 Boot Options](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Installation_Guide/chap-anaconda-boot-options.html)):
```conf
default centos7
prompt 0
label centos7
        kernel vmlinuz
        append initrd=initrd.img inst.ks=http://192.168.1.1/centos/ks/dtn.cfg devfs=nomount selinux=0
```

## Kickstart Installation
Lastly we create a [kickstart configuration file](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Installation_Guide/sect-kickstart-syntax.html) for each node. These configuration files are served by Apache HTTP server. For example, the kickstart configuration file for *pulpo-dtn* is `/var/www/html/centos/ks/dtn.cfg`, with the following content:
```conf
#version=CentOS7
# System authorization information
auth --enableshadow --passalgo=sha512
# Use network installation
url --url http://192.168.1.1/centos/7/os/x86_64/
# Use graphical install
graphical
# Initial Setup is not started the first time the system boots
firstboot --disabled
ignoredisk --only-use=sda
# Keyboard layouts
keyboard --vckeymap=us --xlayouts='us'
# System language
lang en_US.UTF-8

# Network information
network  --bootproto=static --device=eno1 --ip=192.168.1.2 --mtu=1500 --netmask=255.255.255.0 --ipv6=auto --activate
network  --bootproto=static --device=ens1f0 --gateway=128.114.86.254 --ip=128.114.86.3 --mtu=9000 --nameserver=8.8.8.8 --netmask=255.255.255.0 --ipv6=auto --activate
network  --hostname=pulpo-dtn.ucsc.edu

# Root password
rootpw --iscrypted $6$xxxxxxxx
# System services
services --enabled="chronyd"
# SELinux configuration
selinux --disabled
# System timezone
timezone America/Los_Angeles --isUtc
# System bootloader configuration
bootloader --location=mbr --boot-drive=sda
# Partition clearing information
zerombr
clearpart --all --initlabel --drives=sda
# Disk partitioning information
part /boot --fstype=xfs --size=256 --asprimary --ondisk=sda
part swap --fstype=swap --size=16384 --asprimary --ondisk=sda
part / --fstype=xfs --grow --size=8192 --asprimary --ondisk=sda

# Accept the End User License Agreement (EULA) without user interaction
eula --agreed
# Reboot after the installation is successfully completed
reboot

%packages
@^minimal
@core
chrony

%end

%post
mkdir -p /root/.ssh
chmod 700 /root/.ssh
echo "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIN9qCAv9Xbyyv+MO70+nGH3kmjTKw/ehdaneCdGztxF0 root@pulpos" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys

%end

%addon com_redhat_kdump --disable --reserve-mb='auto'

%end
```

**NOTES**

1) *pulpo-dtn* has a single SSD. On dual-SSD nodes such as *pulpo-mon01* and *pulpo-mds01*, we create software RAID 1 partitions on the 2 SSDs:
```conf
# System bootloader configuration
bootloader --location=mbr --boot-drive=sda
# Partition clearing information
zerombr
clearpart --all --initlabel --drives=sda,sdb
# Disk partitioning information
part raid.500 --fstype="mdmember" --ondisk=sda --size=256
part raid.506 --fstype="mdmember" --ondisk=sdb --size=256
part raid.1735 --fstype="mdmember" --ondisk=sda --size=16384
part raid.1741 --fstype="mdmember" --ondisk=sdb --size=16384
part raid.950 --fstype="mdmember" --ondisk=sda --grow --size=8192
part raid.956 --fstype="mdmember" --ondisk=sdb --grow --size=8192
raid /boot --device=boot --fstype="xfs" --level=RAID1 --label=boot raid.500 raid.506
raid swap --device=swap --fstype="swap" --level=RAID1 raid.1735 raid.1741
raid / --device=root --fstype="xfs" --level=RAID1 --label=root raid.950 raid.956
```

2) Each of the 3 OSD nodes has two 1TB SATA SSDs (which are the boot drives), twelve 8TB SATA HDDs, and two 1.2TB PCIe SSDs. Most of the time, the twelve 8TB SATA HDDs show up as `sda` - `sdl`, and the two 1TB SATA SSDs as `sdm` & `sdn`, respectively. However, device names do change from time to time! To avoid accidental installation of the OS onto some of the HDDs, one can either pull out all the HDD driver bays during network installation; or leave the *disk partitioning information* blank in the kickstart configuration file and manually configure the partitions during installation.

I took the latter approach. I didn't physically go to the Data Center to configure the partitions; instead, I used Supermicro's GUI tool [IPMIView](https://www.supermicro.com/solutions/SMS_IPMI.cfm) to redirect KVM console to my laptop and did the configuration there.     

3). Somehow, during the network installation, the Anaconda installer would send a BPDU packet to the top-of-the-rack Brocade switch, causing the ports to be err-disabled! Our network engineers had to disable *BPDU guard* on those ports!
