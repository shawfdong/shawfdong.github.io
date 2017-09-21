---
layout: post
title: Installing Globus Connect Server on pulpo-dtn
tags: [Network, Storage, Linux]
---

In this post, we document how we installed [Globus Connect Server](https://www.globus.org/globus-connect-server) on **pulpo-dtn**.<!-- more --> We mostly followed the instructions in [Globus Connect Server Installation Guide](https://docs.globus.org/globus-connect-server-installation-guide/).

* Table of Contents
{:toc}

## Configuring Firewall
1) Open inbound TCP port `2811` from `184.73.189.163` and `174.129.226.69`, which is used for GridFTP control channel traffic:
```shell
[root@pulpo-dtn ~]# firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="184.73.189.163" port protocol="tcp" port="2811" accept'
success

[root@pulpo-dtn ~]# firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="174.129.226.69" port protocol="tcp" port="2811" accept'
success
```

2) Open inbound TCP ports `50000 - 51000` from any, which are used for GridFTP data channel traffic:
```shell
[root@pulpo-dtn ~]# firewall-cmd --permanent --zone=public --add-port=50000-51000/tcp
success
```

<p class="note">We don't need to open ports for outbound traffics, because they are allowed by default. And we won't run <em>MyProxy</em> service, nor <em>OAuth</em> service, on pulpo-dtn.</p>

3) Reload firewall rules:
```shell
[root@pulpo-dtn ~]# firewall-cmd --reload
success
```

4) Verify firewall rules:
```shell
[root@pulpo-dtn ~]# firewall-cmd --list-rich-rules
rule family="ipv4" source address="174.129.226.69" port port="2811" protocol="tcp" accept
rule family="ipv4" source address="184.73.189.163" port port="2811" protocol="tcp" accept

[root@pulpo-dtn ~]# firewall-cmd --list-port
50000-51000/tcp

[root@pulpo-dtn ~]# firewall-cmd --list-all
public (active)
  target: default
  icmp-block-inversion: no
  interfaces: bond0 ens1f0 ens1f1
  sources:
  services: dhcpv6-client ssh
  ports: 50000-51000/tcp
  protocols:
  masquerade: no
  forward-ports:
  sourceports:
  icmp-blocks:
  rich rules:
        rule family="ipv4" source address="174.129.226.69" port port="2811" protocol="tcp" accept
        rule family="ipv4" source address="184.73.189.163" port port="2811" protocol="tcp" accept
```

## Installing Globus Connect Server
0) EPEL repository and *yum-plugin-priorities* have already been installed.

1) Install the Globus Connect Server repository:
```shell
[root@pulpo-dtn ~]# yum -y install https://downloads.globus.org/toolkit/globus-connect-server/globus-connect-server-repo-latest.noarch.rpm
```
which adds 3 repos to `/etc/yum.repos.d/`:
```shell
[root@pulpo-dtn ~]# ls -l /etc/yum.repos.d/globus-toolkit-6-*
-rw-r--r-- 1 root root 466 Sep 12 14:27 /etc/yum.repos.d/globus-toolkit-6-stable-el7.repo
-rw-r--r-- 1 root root 507 Sep 12 14:27 /etc/yum.repos.d/globus-toolkit-6-testing-el7.repo
-rw-r--r-- 1 root root 513 Sep 12 14:27 /etc/yum.repos.d/globus-toolkit-6-unstable-el7.repo
```

2) Install Globus Connect Server
```shell
[root@pulpo-dtn ~]# yum -y install globus-connect-server
```

## Creating a Globus Endpoint
We'll create a Globus Endpoint `ucsc#pulpo-dtn` for *pulpo-dtn*, using authentication method [CILogon](https://docs.globus.org/authorization-authentication-guide/#transfer_to_from_an_endpoint_using_cilogon). For our purpose, it is sufficient to run only the *GridFTP* service on *pulpo-dtn*. we won't run *MyProxy* service, nor *OAuth* service, on *pulpo-dtn*.

1) Modify `/etc/globus-connect-server.conf`:
```ini
[Globus]
User = ucsc
Password = %(GLOBUS_PASSWORD)s

[Endpoint]
Name = ucsc#pulpo-dtn
Public = True
DefaultDirectory = /~/

[Security]
FetchCredentialFromRelay = True
IdentityMethod = CILogon
Gridmap = /etc/grid-security/grid-mapfile
CILogonIdentityProvider = University of California, Santa Cruz

[GridFTP]
Server = %(HOSTNAME)s
RestrictPaths = RW~,N~/.*

[MyProxy]

[OAuth]
```

2) Run:
```shell
[root@pulpo-dtn ~]# globus-connect-server-setup
Password:
Using MyProxy server on None
Configured GridFTP server to run on pulpo-dtn.ucsc.edu
Server DN: /C=US/O=Globus Consortium/OU=Globus Connect Service/CN=168b435e-9e58-11e7-acf8-22000a92523b
Using Authentication Method CILogon
Configured Endpoint pulpo-dtn
```

The above command has started the GridFTP service:
```shell
[root@pulpo-dtn ~]# systemctl status -l globus-gridftp-server.service
● globus-gridftp-server.service - LSB: Globus GridFTP Server
   Loaded: loaded (/etc/rc.d/init.d/globus-gridftp-server; bad; vendor preset: disabled)
   Active: active (running) since Tue 2017-09-19 15:58:51 PDT; 9min ago
     Docs: man:systemd-sysv-generator(8)
   CGroup: /system.slice/globus-gridftp-server.service
           └─14869 /usr/sbin/globus-gridftp-server -c /etc/gridftp.conf -C /etc/gridftp.d -pidfile /var/run/globus-gridftp-server.pid -no-detach -config-base-path /
```

Interestingly, as of this writing, Globus still uses the old SysV init scripts (see `/etc/rc.d/init.d/`), rather than Systemd unit files!
```shell
[root@pulpo-dtn ~]# chkconfig --list

Note: This output shows SysV services only and does not include native
      systemd services. SysV configuration data might be overridden by native
      systemd configuration.

      If you want to list systemd services use 'systemctl list-unit-files'.
      To see services enabled on particular target use
      'systemctl list-dependencies [target]'.

globus-gridftp-server   0:off   1:off   2:on    3:on    4:on    5:on    6:off
globus-gridftp-sshftp   0:off   1:off   2:off   3:off   4:off   5:off   6:off
myproxy-server  0:off   1:off   2:off   3:off   4:off   5:off   6:off
netconsole      0:off   1:off   2:off   3:off   4:off   5:off   6:off
network         0:off   1:off   2:on    3:on    4:on    5:on    6:off
```
We note in passing that *globus-gridftp-sshftp* (sshftp access to globus-gridftp-server) is disabled. We can easily enable it when we want to use *sshftp*.

Also note that we use *relay.globusonline.org* to generate key (`hostkey.pem`) and certificate (`hostcert.pem`), in the directory `/var/lib/globus-connect-server/grid-security/`, for *pulpo-dtn*:
```shell
[root@pulpo-dtn ~]# openssl x509 -in /var/lib/globus-connect-server/grid-security/hostcert.pem -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            16:8b:43:5f:9e:58:11:e7:ac:f8:22:00:0a:92:52:3b
    Signature Algorithm: sha512WithRSAEncryption
        Issuer: C=US, O=Globus Consortium, CN=Globus Connect CA 3
        Validity
            Not Before: Sep 19 23:04:43 2017 GMT
            Not After : Sep 15 23:04:43 2037 GMT
        Subject: C=US, O=Globus Consortium, OU=Globus Connect Service, CN=168b435e-9e58-11e7-acf8-22000a92523b
```
Globus uses a self-signed CA certificate:
```shell
[root@pulpo-dtn ~]# openssl x509 -in /var/lib/globus-connect-server/grid-security/certificates/a059cd44.0 -text -noout
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 9931528342451824502 (0x89d3e0687090a776)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=US, O=Globus Consortium, CN=Globus Connect CA 3
        Validity
            Not Before: Feb 23 16:14:02 2016 GMT
            Not After : Feb 18 16:14:02 2036 GMT
        Subject: C=US, O=Globus Consortium, CN=Globus Connect CA 3
```

3) We can visit  [https://www.globus.org/app/endpoints/1b557e2e-9d8e-11e7-ace2-22000a92523b/overview](https://www.globus.org/app/endpoints/1b557e2e-9d8e-11e7-ace2-22000a92523b/overview) to verify that the endpoint `ucsc#pulpo-dtn` has been successfully created!


## Mapping Users
We'll map the X.509 distinguished names of CILogon certificates to local POSIX usernames in `/etc/grid-security/grid-mapfile`. My UCSC username is *shaw*. As an example, I'll describe how to set up the mapping for user *shaw*.

1) Find out my `UID` from UCSC Blue directory service:
```shell
[root@pulpo-dtn ~]# yum -y install openldap-clients

[root@pulpo-dtn ~]# ldapsearch -x -H ldap://ldap-blue.ucsc.edu -LLL 'uid=shaw'
dn: uid=shaw,ou=people,dc=ucsc,dc=edu
homeDirectory: /afs/cats.ucsc.edu/users/t/shaw
gidNumber: 100000
uidNumber: 16348
```

2) Create a user *shaw* on *pulpo-dtn* with no shell access (*/sbin/nologin*):
```shell
[root@pulpo-dtn ~]# groupadd -g 16348 shaw
[root@pulpo-dtn ~]# useradd -u 16348 -g shaw -c "Shawfeng Dong" -m -d /mnt/pulpos/shaw -s /sbin/nologin shaw
```
Note we use the same `UID` from the Blue directory service, which may not be strictly necessary; but we must use the same username. The home directory is on the Ceph Filesystem, which is mounted at `/mnt/pulpos/`.

3) Find out my CILogon certificate subject at [https://cilogon.org/](https://cilogon.org/)
```conf
/DC=org/DC=cilogon/C=US/O=University of California, Santa Cruz/CN=Shawfeng Dong A13576
```

4) Map my CILogon certificate subject to user *shaw*, by appending the following line to `/etc/grid-security/grid-mapfile`:
```conf
"/DC=org/DC=cilogon/C=US/O=University of California, Santa Cruz/CN=Shawfeng Dong A13576" shaw
```

5) Lastly, we perform some tests by transferring files from [ESnet test DTN at LBL](https://fasterdata.es.net/performance-testing/DTNs/) (Globus endpoint `esnet#lbl-diskpt1`) to *pulpo-dtn* (Globus endpoint `ucsc#pulpo-dtn`), using [Globus Online](https://www.globus.org/app/transfer). It works!
