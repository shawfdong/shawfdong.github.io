---
layout: post
title: PF on Mac OS X
tags: [Security, Mac OS X]
---

Starting from version 10.7 (Lion), Mac OS X includes 2 firewalls: PF & Application Firewall. Both are disabled by default.<!-- more -->

* Table of Contents
{:toc}

PF
--
Mac OS X 10.6 (and earlier) came with [IPFW](https://www.freebsd.org/doc/handbook/firewalls-ipfw.html), a port of FreeBSD's stateful firewall. IPFW was deprecated in OS X 10.7, and was completely removed in OS X 10.10; it was replaced with PF. [PF](http://www.openbsd.org/faq/pf/) (Packet Filter) is OpenBSD's system for filtering TCP/IP traffic and doing Network Address Translation. PF in OS X, however, appears to be based on the [FreeBSD port of PF](https://www.freebsd.org/doc/handbook/firewalls-pf.html). Like FreeBSD 9.X and later, OS X appears to use the same version of PF as OpenBSD 4.5.

<p class="note">The latest OpenBSD version is 5.6 (as of January 2015); and the configuration syntax for PF changed around 4.6/4.7.</p>

Apple has enhanced PF so that various system components might choose to enable and disable PF, as indicated by the following snippet in `/etc/pf.conf`:
{% highlight plaintext %}
# This file contains the main ruleset, which gets automatically loaded
# at startup.  PF will not be automatically enabled, however.  Instead,
# each component which utilizes PF is responsible for enabling and disabling
# PF via -E and -X as documented in pfctl(8).  That will ensure that PF
# is disabled only when the last enable reference is released.
{% endhighlight %}

These two flags, `-E` and `-X`, are absent from **pfctl** on other BSDs. Here's how they are documented in **pfctl(8)**:
{% highlight conf %}
     -E      Enable the packet filter and increment the pf enable reference count.
     -X token
             Release the pf enable reference represented by the token passed.
     -s References  Show pf-enable reference statistics (pid/name of enabler, token, timestamp).
{% endhighlight %}

The main PF configuration file is `/etc/pf.conf`, which defines the following main ruleset by default in OS X 10.9 & 10.10:
{% highlight conf %}
scrub-anchor "com.apple/*"
nat-anchor "com.apple/*"
rdr-anchor "com.apple/*"
dummynet-anchor "com.apple/*"
anchor "com.apple/*"
load anchor "com.apple" from "/etc/pf.anchors/com.apple"
{% endhighlight %}

The main ruleset loads sub rulesets defined in `/etc/pf.anchors/com.apple`, using [anchor](http://www.openbsd.org/faq/pf/anchors.html):
{% highlight conf %}
anchor "200.AirDrop/*"
anchor "250.ApplicationFirewall/*"
{% endhighlight %}

The launchd configuration file for PF is `/System/Library/LaunchDaemons/com.apple.pfctl.plist`. PF is disabled by default:
{% highlight shell %}
$ sudo pfctl -s info
No ALTQ support in kernel
ALTQ related functions disabled
Status: Disabled                              Debug: Urgent
{% endhighlight %}

Application Firewall
--------------------
OS X v10.5.1 and later include Application Firewall that allow the users to control connections [on a per-application basis](http://support.apple.com/en-us/HT201642) (rather than a per-port basis). Application Firewall is disabled by default.

After enabling the Application Firewall (**System Preferences** -> **Security & Privacy** -> `Firewall` -> `Turn On Firewall`), you'll find PF is enabled too:
{% highlight shell %}
$ sudo pfctl -s info
Status: Enabled for 0 days 00:02:03           Debug: Urgent

$ sudo pfctl -s References
TOKENS:
PID      Process Name                 TOKEN                    TIMESTAMP
618      socketfilterfw               9813589183660731843      0 days 00:03:31

$ sudo pfctl -a com.apple -s rules
anchor "200.AirDrop/*" all
anchor "250.ApplicationFirewall/*" all
{% endhighlight %}

Apparently Application Firewall enables PF using `pfctl -E`. In addition to its own rules, Application Firewall generates a set of dynamic rules (sub ruleset) for PF through anchor point `com.apple/250.ApplicationFirewall`. At this stage, the sub ruleset is empty, which [got someone really confused](http://www.zomo.co.uk/2011/09/pf-on-os-x-10-7/).

But if either "Enable stealth mode" or "Block all incoming connections" is checked in `Firewall Options...`, dynamic rules for PF will indeed be created:
{% highlight shell %}
$ sudo pfctl -a com.apple/250.ApplicationFirewall -s rules
scrub in all fragment reassemble
block drop in inet proto icmp all icmp-type echoreq
block drop in inet proto icmp all icmp-type echoreq
block drop in inet6 proto ipv6-icmp all icmp6-type echoreq
{% endhighlight %}

**Note** there is a bug in Apple's implementation of PF! According to pfctl(8):
> If the anchor name is terminated with a '*' character, the -s flag will recursively print all anchors in a brace delimited block.

But it produces an error instead:
{% highlight shell %}
$ sudo pfctl -a 'com.apple/*' -sr
anchor "*" all {
pfctl: DIOCGETRULES: Invalid argument
}
anchor "*" all {
pfctl: DIOCGETRULES: Invalid argument
}
{% endhighlight %}

We have to use the full anchor path:
{% highlight shell %}
$ sudo pfctl -v -s Anchors
  com.apple
  com.apple/200.AirDrop
  com.apple/200.AirDrop/Bonjour
  com.apple/250.ApplicationFirewall

$ sudo pfctl -a "com.apple/200.AirDrop/Bonjour" -sr
pass in on p2p0 inet6 proto udp from any to any port = 5353 keep state
pass out on p2p0 proto tcp all flags any keep state
{% endhighlight %}

As you can see, a set of dynamic PF rules is created for [AirDrop](https://en.wikipedia.org/wiki/AirDrop) too. I surmise they are still created by Application Firewall, because according to the output of `pfctl -s References`, PF has only been enabled once, by Application Firewall.

Besides using the **Security & Privacy** Preference pane, you can also configure the Application Firewall from the command line. The utilities for Application Firewall are located at `/usr/libexec/ApplicationFirewall`. The default configuration file is `/usr/libexec/ApplicationFirewall/com.apple.alf.plist`; and the running configuration file is `/Library/Preferences/com.apple.alf.plist`.

Stopping and starting Application Firewall is easy enough, using [launchd](http://launchd.info/). To stop:
{% highlight shell %}
$ sudo launchctl unload /System/Library/LaunchAgents/com.apple.alf.useragent.plist
$ sudo launchctl unload /System/Library/LaunchDaemons/com.apple.alf.agent.plist
{% endhighlight %}

To start:
{% highlight shell %}
$ sudo launchctl load /System/Library/LaunchDaemons/com.apple.alf.agent.plist
$ sudo launchctl load /System/Library/LaunchAgents/com.apple.alf.useragent.plist
{% endhighlight %}

We can configure the settings of Application Firewall using `socketfilterfw`:
{% highlight shell %}
usage: /usr/libexec/ApplicationFirewall/socketfilterfw [-c] [-w] [-d] [-l] [-T] [-U] [-B] [-L]
    [-a listen or accept] [-p pid to write] [--getglobalstate] [--setglobalstate on | off]
    [--getblockall] [--setblockall on | off] [--listapps]
    [--getappblocked <path>] [--blockapp <path>] [--unblockapp <path>]
    [--add <path>] [--remove <path>] [--getallowsigned] [--setallowsigned]
    [--getstealthmode] [--setstealthmode on | off]
    [--getloggingmode] [--setloggingmode on | off]
    [--getloggingopt] [--setloggingopt throttled | brief | detail]
{% endhighlight %}

pflog
-----
Logging support for PF is provided by `pflog`. The pflog interface is a pseudo-device which makes visible all packets logged by PF. Logged packets can easily be monitored in real time by invoking **tcpdump** on the pflog interface.

Create a `pflog` interface:
{% highlight shell %}
$ man 4 pflog
$ sudo ifconfig pflog0 create
{% endhighlight %}

Monitor all packets logged by PF:
{% highlight shell %}
$ sudo tcpdump -n -e -ttt -i pflog0
{% endhighlight %}

Destroy the `pflog` interface when you are done with it:
{% highlight shell %}
$ sudo ifconfig pflog0 destroy
{% endhighlight %}

Precedence
----------
If two firewalls, Application Firewall & PF, are both running, you may wonder whose rules take precedence. Let's find out.

The logs of Application Firewall are saved in `/var/log/appfirewall.log`. You'll see a lot entries like the following, repeating roughly 2 times per minute on my iMac:
{% highlight plaintext %}
Jan 20 00:03:35 manjusri.local socketfilterfw[228] <Info>: Dropbox109: Deny UDP CONNECT (in:22 out:0)
Jan 20 00:03:35 manjusri.local socketfilterfw[228] <Info>: ntpd: Deny UDP CONNECT (in:2 out:0)
Jan 20 00:03:35 manjusri.local socketfilterfw[228] <Info>: netbiosd: Deny UDP CONNECT (in:13 out:0)
Jan 20 00:03:44 manjusri.local socketfilterfw[228] <Info>: Stealth Mode connection attempt to UDP 3 time
Jan 20 00:03:44 manjusri.local socketfilterfw[228] <Info>: Stealth Mode connection attempt to TCP 2 time
{% endhighlight %}

Add the following as the first rule of `/etc/pf.conf`:
{% highlight conf %}
set skip on lo0
{% endhighlight %}

Add the following 3 lines to `/etc/pf.conf` (to block incoming traffic but allow outgoing traffic):
{% highlight conf %}
pass in quick proto udp to any port 5353
block in
pass out quick
{% endhighlight %}
The first rule is to allow incoming **Bonjour** traffic. In a hostile environment, e.g., a public WiFi, we'll put the above 3 lines at the end of the file to block all incoming traffic, in which case, the sub rulesets in anchor "com.apple" will have no effect!

<p class="note">For each packet or connection evaluated by PF, <em>the last matching rule</em> in the ruleset is the one which is applied.</p>

In work environment, you can put the 3 lines right above the line:
{% highlight conf %}
anchor "com.apple/*"
{% endhighlight %}

Reload `/etc/pf.conf`:
{% highlight shell %}
$ sudo pfctl -f /etc/pf.conf
{% endhighlight %}

Show the currently loaded filter rules:
{% highlight shell %}
$ sudo pfctl -s rules
scrub-anchor "com.apple/*" all fragment reassemble
block drop in all
pass out all flags S/SA keep state
anchor "com.apple/*" all
{% endhighlight %}

Check `/var/log/appfirewall.log` again. You'll find no new log entry for Application Firewall appears in the file.

So one can conclude that PF rules are applied first, then the rules for Application Firewall.

SSH
---
To enable OpenSSH server on OS X, in the Sharing Preference pane of System Preferences, check "Remote Login". Or from the command line:
{% highlight shell %}
$ sudo launchctl load -w /System/Library/LaunchDaemons/ssh.plist
{% endhighlight %}

**launchctl(1)** says such about the `-w` flag:
<blockquote>
-w Overrides the Disabled key and sets it to false. In previous versions, this option would modify the configuration file. Now the state of the Disabled key is stored elsewhere on-disk.
</blockquote>
but where exactly is the 'elsewhere'? After some digging, I find it is `/private/var/db/launchd.db/com.apple.launchd/overrides.plist`.

However, I don't like the default configuration for sshd. I prefer to have password authentication disabled. Add the following options to `/etc/ssh/sshd_config`:
{% highlight conf %}
PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
{% endhighlight %}

Restart sshd:
{% highlight shell %}
$ sudo launchctl unload /System/Library/LaunchDaemons/ssh.plist
$ sudo launchctl load /System/Library/LaunchDaemons/ssh.plist
{% endhighlight %}

**Note** to allow incoming traffics to the OpenSSH server through Application Firewall, you must allow incoming connections for `/usr/libexec/sshd-keygen-wrapper`, either in **System Preferences** -> **Security & Privacy** -> `Firewall` -> `Firewall Options...`, or from the command line:
{% highlight shell %}
$ sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /usr/libexec/sshd-keygen-wrapper
{% endhighlight %}

Configuring PF
--------------
The Application Firewall's rule of allowing all incoming incoming traffics to the OpenSSH server offers no defense against brute force attack. Leaving the ssh port open on the internet, the server will get thousands of brute force login attempts each day. PF provides an elegant solution to this problem.

Append the following lines to `/etc/pf.conf` (see [Section 30.3.3.5 - Using Overload Tables to Protect SSH of FreeBSD Handbook](https://www.freebsd.org/doc/handbook/firewalls-pf.html) for an explanation):
{% highlight conf %}
table <bruteforce> persist
block quick from <bruteforce>
pass in inet proto tcp to any port ssh \
    flags S/SA keep state \
    (max-src-conn 5, max-src-conn-rate 5/5, \
     overload <bruteforce> flush global)
{% endhighlight %}

Reload `/etc/pf.conf`:
{% highlight shell %}
$ sudo pfctl -f /etc/pf.conf
{% endhighlight %}

Over time, the table **bruteforce** will be filled by overload rules and its size will grow incrementally, taking up more memory. We can expire table entries using `pfctl`. For example, this command will remove **bruteforce** table entries which have not been referenced for a day (86400 seconds):
{% highlight shell %}
$ sudo pfctl -t bruteforce -T expire 86400
{% endhighlight %}

To automate the process, let's create a timed job using launchd that runs the above command once per day (see [Timed Jobs Using launchd](https://developer.apple.com/library/mac/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/ScheduledJobs.html)).

Create a launchd configuration file `/Library/LaunchDaemons/edu.ucsc.manjusri.pfctl-expire.plist`, with the following content:
{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>edu.ucsc.manjusri.pfctl-expire</string>
	<key>WorkingDirectory</key>
	<string>/var/run</string>
	<key>UserName</key>
	<string>root</string>
	<key>GroupName</key>
	<string>wheel</string>
	<key>Program</key>
	<string>/sbin/pfctl</string>
	<key>ProgramArguments</key>
	<array>
		<string>pfctl</string>
		<string>-t</string>
		<string>bruteforce</string>
		<string>-T</string>
		<string>expire</string>
		<string>86400</string>
	</array>
	<key>StartCalendarInterval</key>
	<dict>
		<key>Hour</key>
		<integer>10</integer>
		<key>Minute</key>
		<integer>10</integer>
	</dict>
</dict>
</plist>
{% endhighlight %}

Start the timed job:
{% highlight shell %}
$ sudo launchctl load /Library/LaunchDaemons/edu.ucsc.manjusri.pfctl-expire.plist
{% endhighlight %}

**P.S.** There are a few articles on the Internet on using PF on Mac OS X, but they often bypass the configuration file `/etc/pf.conf` (e.g. , [Using pf on OS X Mountain Lion](http://blog.scottlowe.org/2013/05/15/using-pf-on-os-x-mountain-lion/)). If one takes that route, one must disable the Application Firewall. Otherwise Application Firewall will enable PF using the ruleset in `/etc/pf.conf`. Only one ruleset will get loaded at last and become effective; but which one wins will probably be indeterministic or at least could be a surprise. I choose the approach described in this article, because:
1. I always like to try something different
2. I prefer layered defense. In this case, I have 2 firewalls running on the Mac.
