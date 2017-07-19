---
layout: post
title: Automated Installation of CentOS 7
tags: [Linux, Provisioning]
---

In this post I describe how we set up a home-brew environment for automated installation of CentOS 7 on the nodes of our Ceph storage cluster [Pulpos]({{ site.baseurl }}{% post_url 2017-2-9-pulpos %}).<!-- more --> For bigger deployment, one might want to look into a more sophisticated provisioning system like [Cobbler](http://cobbler.github.io/).

## Apache HTTP server
First we install the Apache HTTP server on [pulpo-admin]({{ site.baseurl }}{% post_url 2017-7-10-pulpo-admin %}).
* Install Apache HTTP server:
  ```shell
  [root@pulpo-admin ~]# yum -y install http httpd-devel httpd-tools
  ```
* Enable and start httpd:
  ```shell
  [root@pulpo-admin ~]# systemctl enable httpd
  [root@pulpo-admin ~]# systemctl start httpd
  ```

## CentOS 7 Local Mirror
Next we create a [local CentOS 7 mirror](https://wiki.centos.org/HowTos/CreateLocalMirror) for network installs.
```shell
[root@pulpo-admin ~]# yum install rsync

[root@pulpo-admin ~]# mkdir -p /var/www/html/centos

[root@pulpo-admin ~]# rsync -avSHP --delete --exclude "local*" \
--exclude "isos" linux.mirrors.es.net::centos/7.3.1611/ \
/var/www/html/centos/7.3.1611/

[root@pulpo-admin ~]# cd /var/www/html/centos/
[root@pulpo-admin centos]# ln -fs 7.3.1611 7
```

Edit `/etc/yum.repos.d/CentOS-Base.repo` so that **pulpo-admin** will use the local mirror:
```conf
[base]
name=CentOS-$releasever - Base
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
baseurl=file:///var/www/html/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#released updates
[updates]
name=CentOS-$releasever - Updates
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/updates/$basearch/
baseurl=file:///var/www/html/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/extras/$basearch/
baseurl=file:///var/www/html/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus
#mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/centosplus/$basearch/
baseurl=file:///var/www/html/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
```

## DHCP server
1. Install DHCP server:
```shell
[root@pulpo-admin ~]# yum -y install dhcp
```
2. Edit `/etc/dhcp/dhcpd.conf` to configure DHCP server for the control network 192.168.1.1/24 (see [Pulpos Networks]({{ site.baseurl }}{% post_url 2017-6-21-pulpos-networks %})):
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
4. Edit `/usr/lib/systemd/system/tftp.service` so that we'll use `/tftpboot` (instead of the default */var/lib/tftpboot*) as the TFTP directory:
```shell
[root@pulpo-admin ~]# sed -i 's=/var/lib/tftpboot=/tftpboot=' \
/usr/lib/systemd/system/tftp.service
```
5. Enable and start TFTP (**Note**: because we've made changes to the unit file for *tftp.service*, we need to reload systemd manager configuration with `systemctl daemon-reload`):
```shell
[root@pulpo-admin ~]# systemctl daemon-reload
[root@pulpo-admin ~]# systemctl enable tftp
[root@pulpo-admin ~]# systemctl start tftp
```
6. Test TFTP locally:
```shell
[root@pulpo-admin ~]# tftp
(to) localhost
tftp> get menu.c32
tftp> quit
[root@pulpo-admin ~]# ls -l menu.c32
-rw-r--r-- 1 root root 55012 Jul 17 15:19 menu.c32
```
So it works!
7. Copy `initrd.img` and `vmlinuz` from the CentOS 7 local mirror:
```shell
[root@pulpo-admin ~]# cp /var/www/html/centos/7/os/x86_64/images/pxeboot/initrd.img /tftpboot/
[root@pulpo-admin ~]# cp /var/www/html/centos/7/os/x86_64/images/pxeboot/vmlinuz /tftpboot/
```
8. Create a default PXELINUX configuration file (see: https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Installation_Guide/chap-anaconda-boot-options.html)

```
# mkdir /tftpboot/pxelinux.cfg

```

http://www.syslinux.org/wiki/index.php?title=PXELINUX

C0A80102 -> 192.168.1.2 (uppercase hexadecimal)


```
# cat /tftpboot/pxelinux.cfg/default
default centos7
prompt 0
label centos7
	kernel vmlinuz
	append initrd=initrd.img ks=http://192.168.100.1/centos/ks/ks.cfg method=http://192.168.100.1/centos/7/os/x86_64/  modprobe.blacklist=nouveau devfs=nomount
```

Somehow, when auto-installing a node, the installer would send a BPDU packet to the Brocade switch, causing the port to be err-disabled! 

```
[root@pulpo-admin pxelinux.cfg]# ln -s localboot default

[root@pulpo-admin pxelinux.cfg]# ln -s install-single-ssd C0A80102
```

9. Create a kickstart configuration file (see https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Installation_Guide/sect-kickstart-syntax.html):

Sample Kickstart Configuration Files
```
 # cat /var/www/html/centos/ks/ks.cfg
```



## References
