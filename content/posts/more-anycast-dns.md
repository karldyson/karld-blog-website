+++
title = "More Anycast DNS"
description = "Anycasting DNS to the internet"
date = 2020-11-22
[extra]
toc = false
toc_inline = true
toc_ordered = true
styles = ["posts/demo.css"]
[taxonomies]
tags = ["anycast","dns","bgp","exabgp","network","junos"]
+++

### aka “how to do BGP with Vultr using ExaBGP”

Having [previously written about](@/posts/anycasting-dns.md) locally anycasting services within my home network, I recently decided to run an experiment anycasting a prefix on the internet.

I’ve used ExaBGP before, and so it was a no brainer to use it again.

For anycasted services, it offers a couple of benefits;

* it’s small and lightweight
* it’s in many linux distros
* it can easily spawn a watchdog process that you can use to control your prefix advertisements

I’ve been using Vultr for my authoritative DNS servers for a while, and so it was also a bit of a no brainer to use their services for this. I’m familiar with their UI, I already have an account, and even on the cheapest virtual machines, you can do BGP; you just need to send them a letter of authority (LOA) proving you own the address space you plan on advertising.

I have my own IPv6 PI address space from RIPE, courtesy of a friendly sponsoring LIR, as well as my own ASN, so I was all set.

The other nice thing about Vultr is that the BGP session is the same peer IP at the Vultr end regardless of which of their datacentres you choose to spin things up in which means your automation to configure things is easier.

Vultr insist on a BGP session password, and the first problem I ran into turns out to be related to this, and so part of the reason for writing this is to help out anyone that also runs into this problem.

I went down a bit of a rabbit hole thinking the problem was to do with multihop BGP (Vultr’s sessions are multihop) and wondered if I needed to be setting the TTL on the outbound packets. This turned out not to be the case, but I left the settings in place anyway.

Either in a template or neighbor configuration, depending on the complexity of your needs, you're looking for  `outgoing-ttl 2;` and `incoming-ttl 2;`

I installed BIRD and used one of Vultr’s canned configs for the virtual machine in question, and this worked like a charm, so this steered me in the direction of the problem being in my configuration of ExaBGP.

BIRD would have done the job, but I’d have had to write a new watchdog, and the one I have for ExaBGP is tried and tested, tweaked to my needs, and works well, so I was keen to get ExaBGP working. It’s a complete re-write from the one I talk about in the earlier blog post, so maybe I’ll write a post on that soon…

By default, ExaBGP expects `md5-password` to be a <mark>base64 encoded string</mark>, and so if what you’ve specified is just the plain text string for the session, it won’t work. If you want to specify the plain text password in this parameter, you need to set `md5-base64` to `false`.

I’m using ansible to automate configuration, and so the template for my `exabgp.conf` looks like this:

```
process monitor {
    run /usr/local/bin/exabgp-healthcheck anycast;
    encoder text;
}
 
neighbor 2001:19f0:ffff::1 {
    local-address {{ ansible_default_ipv6.address }};
    router-id {{ router_id }};
    local-as {{ as_number }};
    peer-as 64515;
    hold-time 10;
    group-updates true;
    md5-password {{ md5_password }};
    md5-base64 false;
    outgoing-ttl 2;
    incoming-ttl 2;
 
    capability {
        graceful-restart 10;
    }
 
    family {
        ipv6 unicast;
    }
 
    api service {
        processes [ monitor ];
    }
}
```

I don’t have any IPv4 prefixes to advertise, so you’d need to add the relevant bits to the above if you do.

I had upgraded ExaBGP to version 4 as part of a distro upgrade on my internal resolvers, and rather than update the watchdog script, I opted for reverting ExaBGP’s setting instead. So in `exabgp.env` I altered the `ack` setting to `false` in the `[exabgp.api]` section.
