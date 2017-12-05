---
layout: post
title: FUSE for macOS
tags: [macOS, Mac OS X, Ceph]
---

In this post, I document how I installed FUSE for macOS on my iMac.<!-- more -->

1) There was an old installation of MacFUSE on the iMac. [Remove MacFUSE & NTFS-3G](https://apple.stackexchange.com/questions/177885/how-to-completely-remove-fuse-for-mac-os-x-macfuse-ntfs-3g)

{% highlight shell_session %}
$ sudo /System/Library/Filesystems/ntfs-3g.fs/Support/uninstall-package.sh

$ pkgutil --pkgs | grep fuse
com.google.macfuse
com.google.macfuse.core

$ kextstat | grep fuse
(no output)

$ pkgutil --files com.google.macfuse.core
$ pkgutil --files com.google.macfuse

$ sudo /Library/Filesystems/fusefs.fs/Support/uninstall-macfuse-core.sh
MacFUSE Uninstaller: Can not find the Archive.bom for MacFUSE Core package.

$ cd /Library/Filesystems
$ ls
NetFSPlugins/ fusefs.fs/
$ sudo rm -rf fusefs.fs/
$ cd /Library/Frameworks/
$ sudo rm -rf MacFUSE.framework/

$ sudo pkgutil --forget com.google.macfuse.core
Forgot package 'com.google.macfuse.core' on '/'.

$ sudo pkgutil --forget com.google.macfuse
Forgot package 'com.google.macfuse' on '/'.
{% endhighlight %}

2) Reboot.

3) Install [FUSE for macOS](https://osxfuse.github.io/), successor to MacFUSE.
{% highlight shell_session %}
$ pkgutil --pkgs | grep -i fuse
com.github.osxfuse.pkg.Core
com.github.osxfuse.pkg.PrefPane

$ pkgutil --files com.github.osxfuse.pkg.PrefPane
...
usr/local/include/osxfuse/fuse/fuse.h
usr/local/include/osxfuse/fuse/fuse_common.h
usr/local/include/osxfuse/fuse/fuse_common_compat.h
usr/local/include/osxfuse/fuse/fuse_compat.h
usr/local/include/osxfuse/fuse/fuse_darwin.h
usr/local/include/osxfuse/fuse/fuse_lowlevel.h
usr/local/include/osxfuse/fuse/fuse_lowlevel_compat.h
usr/local/include/osxfuse/fuse/fuse_opt.h
usr/local/include/osxfuse/fuse.h
usr/local/lib/libosxfuse.2.dylib
usr/local/lib/libosxfuse.dylib
usr/local/lib/libosxfuse.la
usr/local/lib/libosxfuse_i64.2.dylib
usr/local/lib/libosxfuse_i64.dylib
usr/local/lib/libosxfuse_i64.la
...
{% endhighlight %}

4) Install and test [SSHFS](https://github.com/osxfuse/sshfs/releases).
{% highlight shell_session %}
$ pkgutil --pkgs | grep -i ssh
com.github.osxfuse.pkg.SSHFS

$ pkgutil --files com.github.osxfuse.pkg.SSHFS
usr
usr/local
usr/local/bin
usr/local/bin/sshfs
usr/local/share
usr/local/share/man
usr/local/share/man/man1
usr/local/share/man/man1/sshfs.1

$ otool -L /usr/local/bin/sshfs
/usr/local/bin/sshfs:
        /usr/local/lib/libosxfuse_i64.2.dylib (compatibility version 10.0.0, current version 10.3.0)
        /System/Library/Frameworks/Carbon.framework/Versions/A/Carbon (compatibility version 2.0.0, current version 136.0.0)
        /usr/lib/libgcc_s.1.dylib (compatibility version 1.0.0, current version 1.0.0)
        /usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 111.1.7)
        /System/Library/Frameworks/CoreServices.framework/Versions/A/CoreServices (compatibility version 1.0.0, current version 32.0.0)
        /System/Library/Frameworks/CoreFoundation.framework/Versions/A/CoreFoundation (compatibility version 150.0.0, current version 476.19.0)

$ mkdir /tmp/ssh
$ sshfs p:/var/www /tmp/ssh -ocache=no -onolocalcaches -ovolname=ssh
$ df -h
Filesystem      Size   Used  Avail Capacity   iused    ifree %iused  Mounted on
p:/var/www     492Gi   20Gi  447Gi     5%    101971 32666029    0%   /private/tmp/ssh

$ umount /tmp/ssh
{% endhighlight %}

## ToDo
If I have some time, a fun exercise will be to port **ceph-fuse** to *FUSE for macOS*.

We note in passing that there is [CephFS Client on Windows based on Dokan 0.6.0](https://drupal.star.bnl.gov/STAR/blog/mpoat/cephfs-client-windows-based-dokan-060).
