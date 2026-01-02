+++
title = "IPv6 Privacy Extensions"
description = "What are IPv6 Privacy Extensions?"
date = 2011-04-07
updated = "2017-03-21"
[extra]
toc = false
toc_inline = true
toc_ordered = true
styles = ["posts/demo.css"]
[taxonomies]
tags = ["ipv6","privacy-extensions"]
+++

# Introduction

If you've switched on IPv6, whether via a tunnel, or natively, your machine is likely to have stepped out from behind NAT and now has a globally routable address.

Whilst NAT is not security, and inbound connections to your machine are likely behind a firewall, that doesn't change one thing: in many cases your IPv6 address is made up automatically by auto-configuration.

It's made up of two parts, the last 64 bits are put together partly from the MAC address of your network interface, and the rest comes from the network prefix.

This makes your machine globally identifiable, and therefore, trackable by third parties, such as web sites.

To this end, [RFC3041](https://datatracker.ietf.org/doc/html/rfc3041), superseded by [RFC4941](https://datatracker.ietf.org/doc/html/rfc4941) defines privacy extensions.

When enabled, your machine still has the auto-configuration address, but now also has a randomised additional address that changes periodically and is used for outbound connections.

On many linux distributions this is disabled by default. Windows XP is the same. Windows Vista enables it by default, and I believe newer versions of Windows also enable it by default.

I don't have access to any Windows Server installations with IPv6, so I'm unsure if the server editions do this too.

I mention servers, as you probably don't want this on a server. Imagine, for example, a mail server making an outbound connection from a random and short lived IP address. It's unlikely to have a valid PTR, for example, and many, not all I grant you, but many MTAs will not like that.

# Enabling it on Linux

Enabling it on linux distributions is quite straight forward:

as root:

`echo 2 > /proc/sys/net/ipv6/conf/all/use_tempaddr`

You can automate this at boot in the normal way; edit `/etc/sysctl.conf` and add the line:

`net.ipv6.conf.all.use_tempaddr=2`

You can swap all for a specific IPv6 enabled interface, such as `eth0` if you require.

# Enabling it on OS X

OS X has it disabled by default. I believe you can add to, or create `/etc/sysctl.conf` with the following:

`net.ipv6.conf.all.use_tempaddr=1`

Don't quote me on that last one; I've not tested it!

[edit 21/Mar/2017]: MacOS Sierra seems to have it enabled by default...
