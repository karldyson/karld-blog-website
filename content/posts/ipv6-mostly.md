+++
title = "IPv6, Mostly..."
date = 2025-04-26
[extra]
toc = true
toc_inline = true
toc_ordered = true
styles = ["posts/demo.css"]
[taxonomies]
tags = ["dhcp","dns64","ipv6","ipv6-mostly","juniper","srx","junos","nat64","rdnss","slaac","option108","option-108","option 108"]
+++

{% alert(warning=true) %}
<mark>TL;DR</mark>: This article turned out longer than expected, so if you just want the tl;dr, [skip to that bit](#configuration-time).
{% end %}

# Introduction

IPv4 has run out. This is not news.

IPv6 uptake, however, has been slow given it was first created in 1995.

It can be intimidating, if you're not a network expert, and there are still things that are perceived to just work best if you're doing IPv4.

Examples include getting information about the network to a client, other than the client's IP address.

With IPv4, when you connect to a network, by default in most cases, your device asks the network for an IP address using a protocol called Dynamic Host Configuration Protocol (DHCP).

But the protocol doesn't just give your device an IP address; it can also let your device know about a bunch of other things on the network. Most commonly the DNS servers that will resolve names for you, but also things like NTP servers, proxy discovery, and more can be conveyed to the device using this mechanism.

So IPv6 just needs a version of DHCP then? Well sure, and it does, cunningly named DHCPv6. But for lots of reasons, on a client network, you probably don't want to be handing out IPv6 addresses with DHCPv6. IPv6 is more modern and can sort out its addresses statelessly and automatically using something called SLAAC. You probably want IPv6 to be stateless for a variety of reasons. Privacy is high up the list.

# Privacy?

Let's take a step back.

With IPv4, the address space is tiny compared to the number of devices on the modern internet. Think about your phones, iPads, laptops, home assistant, smart speakers; the list goes on. Even a modest modern home has dozens of devices needing an IP address.

So, there are blocks of IPv4 addresses reserved for use within private networks that will never be routed on the internet. Those private addresses can be routinely used in many networks concurrently because whilst IP addresses have to be unique, they only have to be unique within a given network.

Your device gets one of those private addresses.

But, how do you communicate on the internet if your device has an IP address that doesn't work on the internet?

# Network Address Translation (NAT)

When your device makes a connection to something on the internet, your router does something called Network Address Translation (NAT).

That is to say, on the inside of your network there will be plenty of those private addresses for all of the devices, but your router will map them all onto a much smaller pool of internet routable addresses and then juggle making sure the right internet traffic is sent to the right device.

This means that it's harder for websites to track you using your IP address, because a whole bunch of devices and users will be "hidden" behind a much smaller NAT pool, possibly just a single IP.

With IPv6, the size of the address space is vast. There's no need for NAT and so the IP address your machine has, is the IP that things you connect to on the internet see. For example when you browse a website.

Instead of all devices on a network being hidden behind one IP, every individual device has a real internet IP address.

This makes it a lot easier for websites to use your IP address to track you, if your IP address is now per device.

# So just how big is IPv6 to not need NAT...?

Won't we run out? We did with IPv4 after all...

IPv4 has a total of 232 IP addresses. That's 4,294,967,296. Four billion and change. Sounds like a lot, right?

IPv6 has a total of 2128 IP addresses.

That's 340,282,366,920,938,463,463,374,607,431,768,211,456

340 trillion trillion trillion.

For context, that's more than 100 times the number of atoms on the surface of the Earth[^1].

It was calculated that we could assign a unique /48 to every human being on Earth for the next 480 years before we would run out.

A /48 is the expected common assignment block size. It's what my ISP has assigned me for my home network. It's the smallest block of IPv6 address space that you can route on the wider internet.

Your typical individual network subnet will be assigned a /64.

A /48 contains 65,536 /64 subnets. 

A /64 network contains 18,446,744,073,709,551,616 usable IPv6 addresses.

# Back to Privacy, then...

So, IPv6 has the concept of Privacy Extensions ([RFC8981](https://datatracker.ietf.org/doc/html/rfc8981)) which means that it has a mechanism for your device to assign itself new IP addresses temporarily, use them for a while, and then discard them. But it requires that your network is assigning addresses statelessly.

# Router Advertisements

If you're doing stateless IPv6 addresses, and you want to move away from using IPv4, how do you tell your devices about things like DNS resolvers?

In order to statelessly configure its IPv6 addresses, your device sends something called Router Solicitation (RS) messages to the network. Available routers on the network respond with Router Advertisement (RA) messages.

These RAs include the network prefix(es) in use so SLAAC and privacy extensions can work. They can also be used to tell the client device whether to actually use stateless addresses, whether to try DHCPv6, or whether to do a bit of both, and use stateless addresses but with other information (like DNS servers) from DHCPv6.

More recently, to avoid the need for DHCPv6 at all, these RAs can now contain the DNS server information (RDNSS).

OK, so we'll switch off IPv4 inside our networks? We'll give all the clients an IPv6 address.

We can use the RA to give devices DNS servers.

But what about things on the internet that are still only configured for IPv4...?

How do you connect to them?

# NAT64

In a similar way to how NAT works above, translating networks of private addresses into a smaller pool of internet routable addresses, it's possible to translate addresses between address families: ie: between IPv4 and IPv6.

Your router can tell your device that the network has the ability to do NAT64.

With this, devices connect to a mapped IPv6 address for any internet device that only has IPv4.

There's what's called a well-known allocation for NAT64: `64:ff9b::/96`.

This allows us to map the entire IPv4 address space. For example, let's say your device wants to connect to a website that only has IPv4 and its address is `198.51.100.189`. This maps to `64:ff9b::c633:64bd`.

If you look carefully at the end of that, you can see we have `c633:64bd`. 

`c6` in hexadecimal is `198` in decimal.

Converting the rest, then: `33` becomes `51`, `64` becomes `100`, and `bd` becomes `189`.

Your device makes an IPv6 connection to `64:ff9b::c633:64bd` and when this reaches the router, it knows to translate this back the other way and make an IPv4 connection to `198.51.100.189`.

But how does the device know to attempt to connect to that mapped IPv6 address?

# DNS64

One option is that you give them a DNS server that does DNS64.

Usually, dual stack devices will prefer IPv6. As such, when they do DNS lookups to get the IP address for whatever the device needs to connect to, they will look up an IPv6 (AAAA) DNS record first, and fall back to IPv4 (A record).

So, a DNS64 capable DNS server looks up the AAAA record. If it exists, it gives the response back to the device and everything works over IPv6.

However, if the AAAA record doesn't exist, the service only supports IPv4, the DNS64 server looks up the IPv4 A record, but instead of telling the requesting device, the DNS64 server generates the relevant mapped NAT64 IP address and returns that in the AAAA record.

The device makes what it thinks is an IPv6 connection, and the NAT64 configuration on the router does the translation to IPv4.

Google offer public DNS64 servers on `2001:4860:4860::64` and `2001:4860:4860::6464`

But, some components that run on your machine can decide to use DNS servers of their own choice. This can lead to some things not using a DNS64 server, not getting the mapped IP, and continuing to use IPv4 which may be undesirable.

# Customer Translator (CLAT)

A second method is a customer translator or CLAT.

Your device learns that the network can do NAT64 from the network itself. It learns what the prefix is and can do the mapping from the desired IPv4 address to IPv6 under the covers within the operating system.

Further, your applications don't need to know that they're not making an IPv4 connection.

Your device puts an unroutable placeholder IPv4 address on its network interface to keep up the pretence that IPv4 is working as usual.

When your application attempts to make an IPv4 connection, the CLAT within your device works out the mapping, and makes the IPv6 connection to the mapped IPv6 address. The router does NAT64 and it all works seamlessly.

This even works for IP literals.

# DHCP Option 108

So, how do we persuade devices that the network supports IPv6, and can do NAT64 for those pesky "IPv4 only" things?

Many devices still send out DHCP for IPv4 first.

So there's a mechanism for saying "if you support CLAT, can use the network's NAT64, and don't want or need an IPv4 DHCP lease".

It's called "option 108".

Devices capable of IPv6 mostly, will set option 108 in their DHCP request. If the DHCP server replies with option 108, the client doesn't take an IPv4 lease and applies the placeholder/unroutable IPv4 address to the interface.

# So, lets make it work...

I wanted to tinker with this and see it in action.

My router/firewall was a Juniper SRX240 which sadly will not run a new enough version of Junos to support RDNSS.

It didn't support telling the clients about the NAT64 prefix either, and it seemed very unhappy with me trying to persuade it to do DHCPv6 for just the DNS information.

So, this felt like a great excuse to finally replace it, and so now I have a SRX340 in its place.

The next thing to consider is client device support. My laptop is a MacBook, my family use iPhones and iPads. There's the usual plethora of IoT things around the network too.

Your mileage may vary if you have other things around your network.

So I started with the guest VLAN as there's less in there to disrupt.

I want stateless addresses. I need to configure router advertisements anyway, so I'd like to skip needing DHCPv6 configured.

# Configuration Time 

## NAT

First we'll configure NAT64 so that when we tell the clients it's available, it actually is. We need to do two things here:

1. We need to take traffic destined for the NAT64 prefix and static NAT it to IPv4. You can see this in the static NAT section at the top of the following configuration snippet.

1. We need to then source NAT the traffic so that it comes from an internet routable IPv4 address. I'm fortunate here in that I have a /29 from my ISP, and so to aid troubleshooting, I have the NAT64 traffic SNAT from a different IP to the IPv4 native traffic.

<details>
<summary>Junos Configuration</summary>

```Junos
security {
    nat {
        static {
            rule-set nat64 {
                from zone guest;
                rule nat64 {
                    match {
                        source-address 2001:db8:97::/64;
                        destination-address 64:ff9b::/96;
                    }
                    then {
                        static-nat {
                            inet;
                        }
                    }
                }
            }
        }
        source {
            pool guest {
                address {
                    192.0.2.230/32;
                }
            }
            pool guest-nat64 {
                address {
                    192.0.2.229/32;
                }
            }
            rule-set guest {
                from zone guest;
                to zone untrust;
                rule guest {
                    match {
                        source-address 192.168.97.0/24;
                    }
                    then {
                        source-nat {
                            pool {
                                guest;
                            }
                        }
                    }
                }
                rule guest-nat64 {
                    match {
                        source-address 2001:db8:97::/64;
                        destination-address 0.0.0.0/0;
                    }
                    then {
                        source-nat {
                            pool {
                                guest-nat64;
                            }
                        }
                    }
                }
            }
        }
    }
}
```

</details>

## SLAAC, RDNSS and PREF64

Next we need to tell the SRX to do SLAAC, RDNSS and PREF64...

First, our interface needs an IPv6 address.

```Junos
interfaces {
    irb {
        unit 197 {
            description Guest;
            family inet {
                address 192.168.97.1/24;
            }
            family inet6 {
                address 2001:db8:97::1/64;
            }
        }
    }
}
```

Next, SLAAC, RDNSS and PREF64.

Within the protocols router-advertisement section, we will use the following options:

* `dns-server-address` option to set the RDNSS server(s)
* `nat-prefix` to set the PREF64 prefix

```Junos
protocols {
    router-advertisement {
        interface irb.197 {
            preference high;
            max-advertisement-interval 20;
            min-advertisement-interval 3;
            other-stateful-configuration;
            solicit-router-advertisement-unicast;
            default-lifetime 9000;
            dns-server-address 2001:4860:4860::8888 {
                lifetime 9000;
            }
            prefix 2001:db8:97::/64;
            nat-prefix 64:ff9b::/96 {
                lifetime 18000;
            }
        }
    }
}
```

## Option 108

Lastly, we'll support clients that support DHCP option 108 in our DHCP configuration.

```Junos
access {
    address-assignment {
        family inet {
            network 192.168.97.0/24;
            range r1 {
                low 192.168.97.128;
                high 192.168.97.191;
            }
            dhcp-attributes {
                maximum-lease-time 9000;
                router {
                    192.168.97.1;
                }
                propagate-settings irb.197;
                option 108 unsigned-integer 9000;
            }
        }
    }
}
```

# Did it work?

This is from a MacBook running MacOS 15.

We can see that the RA contains the configured details.

```
$ ipconfig getra en4
RA Received 04/25/2025 23:06:51 from fe80::cee1:9400:c55a:d430, length 96, hop limit 64, lifetime 9000s, reachable 0ms, retransmit 0ms, flags 0x48=[ other ], pref=high
	source link-address option (1), length 8 (1): cc:e1:94:5a:d4:30
	rdnss option (25), length 24 (3):  lifetime 9000s, addr: 2001:4860:4860::8888
	prefix info option (3), length 32 (4):  2001:db8:97::/64, flags [ onlink auto ], valid time 2592000s, pref. time 604800s
	pref64 option (38), length 16 (2): 64:ff9b::/96 lifetime 18000s
```

We can see that the interface has stateless IPv6 configured, including privacy extensions. It has also enabled CLAT46, and has added the 'placeholder' IPv4 address.

```
$ ifconfig en4
en4: flags=88e3<UP,BROADCAST,SMART,RUNNING,NOARP,SIMPLEX,MULTICAST> mtu 1500
	options=6464<VLAN_MTU,TSO4,TSO6,CHANNEL_IO,PARTIAL_CSUM,ZEROINVERT_CSUM>
	ether a0:ce:c8:dd:5b:74
	inet6 fe80::454:7b3a:963b:7fab%en4 prefixlen 64 secured scopeid 0xd
	inet6 2001:db8:97:8ea:22c2:54cd:a385 prefixlen 64 autoconf secured
	inet6 2001:db8:97:a4ba:e260:2aa1:59c0 prefixlen 64 deprecated autoconf temporary
	inet6 2001:db8:97:479:67b7:1474:4197 prefixlen 64 deprecated autoconf temporary
	inet 192.0.0.2 netmask 0xffffffff broadcast 192.0.0.2
	inet6 2001:db8:97:e9bf:52f3:939c:6a03 prefixlen 64 autoconf temporary
	inet6 2001:db8:97:2:c078:2867:d6d prefixlen 64 clat46
	nat64 prefix 64:ff9b:: prefixlen 96
	nd6 options=201<PERFORMNUD,DAD>
	media: autoselect (1000baseT <full-duplex>)
	status: active
```

The device thinks it has an IPv4 connection to something in Microsoft...

```
$ netstat -f inet -an|grep ESTAB|grep 192.0
tcp4       0      0  192.0.0.2.51079        52.123.128.14.443      ESTABLISHED
<snip>
```

What does the firewall have to say about that....?

```
kdyson@er> show security flow session destination-prefix 52.123.128.14
Total sessions: 0
```

...but `52.123.128.14` maps to `64:ff9b::347b:800e` so lets check again...

```
kdyson@er> show security flow session destination-prefix 64:ff9b::347b:800e
Session ID: 115964145637, Policy name: web-browsing/10, Timeout: 1778, Session State: Valid
  In: 2001:db8:97:8ab:f46e:f285:917f/51062 --> 64:ff9b::347b:800e/443;tcp, Conn Tag: 0x0, If: irb.197, Pkts: 29, Bytes: 11041,
  Out: 52.123.128.14/443 --> 192.0.2.229/18910;tcp, Conn Tag: 0x0, If: pp0.2, Pkts: 24, Bytes: 8486,
Total sessions: 1
```

The IP details have been redacted/sanitised, of course, but note that the SNAT IP is the guest-nat64 pool IP and not the regular guest pool IP.

# The End!

Well, that turned out a bit longer than expected and has a couple of tangents along the way. If you stuck around for all of it, well done and thanks.

I hope it's been useful or interesting!

# Footnotes

[^1]: [Internet Society IPv6 FAQ](https://www.internetsociety.org/deploy360/ipv6/faq)

# Further Reading

* [https://www.ietf.org/archive/id/draft-link-v6ops-6mops-01.html](https://www.ietf.org/archive/id/draft-link-v6ops-6mops-01.html)
* [https://2023.apricot.net/assets/files/APPS314/apnic55-deployingipv_1677492529.pdf](https://2023.apricot.net/assets/files/APPS314/apnic55-deployingipv_1677492529.pdf)
* [https://blog.apnic.net/2019/06/07/how-to-slaac-dhcpv6-on-juniper-vsrx/](https://blog.apnic.net/2019/06/07/how-to-slaac-dhcpv6-on-juniper-vsrx/)
