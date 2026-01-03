+++
title = "IPv6 Privacy Extensions on Linux"
description = "How to make IPv6 privacy extensions work on Linux"
date = 2025-04-28
[extra]
toc = true
[taxonomies]
tags = ["ipv6","linux","slaac","privacy","privacy-extensions"]
+++

# Introduction

This is definitely one of the blog posts written to remind me how to do something.

I'm sure I knew this, but returning to it on a new box caused me frustration that Googling did not help with. Maybe my google-fu is broken?

It was only after quite a bit of googling and reading things that I spotted something in a comment that reminded me of the thing I was missing.

So, I'm writing this so that it will remind me in future, and may help others.

# Scenario

I had built a new Raspberry Pi 5 as a pi-hole server.

Of course, in order to ssh to it and browse to the admin UI, it has fixed IP addresses.

Actual DNS service is done by advertising service IP(s) into the router's routing table so I can do some ECMP and resilience.

I wanted it to recurse to the internet from privacy extensions IPs so that the recursive source IPs change over time.

I knew I needed to stick a `2` in `/proc/sys/net/ipv6/conf/eth0/use_tempaddr` and that things worked best if you also stick `1` in `/proc/sys/net/ipv6/conf/eth0/accept_ra`

But, nothing. No dynamic IPv6 addresses.

Bother.

I knew there was something else I was pretty sure there's a third file I need to do something with, but could I remember which one it was?

Nope.

# Solution

So, I'll cut to the chase.

For automatic IP configuration, there also needs to be a `1` in `/proc/sys/net/ipv6/conf/eth0/autoconf`

# Configuration

This is what I have done to fix this such that it's reboot and upgrade safe. By all means leave me a comment below if I've missed something obvious here!

`sysctl` can be persuaded to make these changes as follows:

```
$ cat /etc/sysctl.d/99-tempaddr.conf
# accept router-advertisements
net.ipv6.conf.all.accept_ra=1
net.ipv6.conf.default.accept_ra=1

# use temporary addresses (privacy extensions)
net.ipv6.conf.all.use_tempaddr=2
net.ipv6.conf.default.use_tempaddr=2

# auto configure addresses
net.ipv6.conf.all.autoconf=1
net.ipv6.conf.default.autoconf=1
```

I don't use NetworkManager for my `eth0` IP addresses, as I'm more familiar with `/etc/network/interfaces` so this is my static config and a poke to make sure that `eth0` picks up the settings too.

```
$ cat /etc/network/interfaces.d/ethernet
auto eth0
allow-hotplug eth0
iface eth0 inet static
	address 10.1.1.53
	netmask 255.255.255.0
	gateway 10.1.1.1

iface eth0 inet6 static
	address 2001:db8::53
	netmask 64
	gateway 2001:db8::1
        post-up /usr/sbin/sysctl net.ipv6.conf.eth0.accept_ra=1
        post-up /usr/sbin/sysctl net.ipv6.conf.eth0.autoconf=1
        post-up /usr/sbin/sysctl net.ipv6.conf.eth0.use_tempaddr=2
```

# Fixed!

...and this is what the output looks like now that that is working...

```
$ ip add sh eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 2c:cf:67:8a:03:50 brd ff:ff:ff:ff:ff:ff
    inet 10.1.1.53/24 brd 10.1.1.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 2001:db8::a863:8e:293f:1ec7/64 scope global temporary dynamic
       valid_lft 525174sec preferred_lft 6706sec
    inet6 2001:db8::2ecf:67ff:fe8a:350/64 scope global dynamic mngtmpaddr
       valid_lft 2591996sec preferred_lft 604796sec
    inet6 2001:db8::53/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::2ecf:67ff:fe8a:350/64 scope link
       valid_lft forever preferred_lft forever
```

..and remote things on the internet see the correct source for my outbound connections...

```
$ dig i.p.c.je txt @i.p.c.je +short
"query received from source IP: 2001:db8::a863:8e:293f:1ec7"
"query received from source port: 47948"
"query received over: udp"
"edns size: 1232"
```

```
$ curl -6 c.je/ip
Your IP is 2001:db8::a863:8e:293f:1ec7
You're using curl/7.88.1
```

# Technicalities

## use_tempaddr

Values in this file have the following meanings:

* `<= 0` : Privacy Extensions is disabled
* `== 1` : Privacy Extensions is enabled, but public addresses are preferred over temporary addresses
* `> 1` : Privacy Extensions is enabled, but temporary addresses are preferred over public addresses

Note that in some very brief and not very scientific testing, if autoconf is enabled and use_tempaddr is set to 1, the addresses that seem to be preferred are the autoconf addresses even if you have a static configured. I don't think this surprises me, but worth noting.

## accept_ra

Simply put, `0` to disable, `1` to enable the acceptance of router advertisements.

Note that if you disable this, you will need to specify the gateway statically if you have not already.

If this is disabled but `use_tempaddr` is non zero, temporary addresses can still be created but just not using prefix information from the router advertisements.

## autoconf

As above; `0` to disable, `1` to enable the automatic configuration of IP addresses.

If `accept_ra` is set to `1` and this is set to `0`, router advertisements will be used for the gateway but not for IP address configuration locally.

If this is enabled and `use_tempaddr` is not, stateless (SLAAC) addresses will be added to the interface but no privacy extensions addresses.
