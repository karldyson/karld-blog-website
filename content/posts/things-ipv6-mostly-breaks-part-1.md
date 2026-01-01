+++
title = "Things IPv6 Mostly Breaks, Part 1"
date = 2025-05-08
[extra]
toc = false
toc_inline = true
toc_ordered = true
styles = ["posts/demo.css"]
[taxonomies]
tags = ["dhcp","dns64","ipv6","ipv6-mostly","juniper","srx","junos","nat64","rdnss","slaac","option108","option-108","option 108","zwift"]
+++

Following my recent article on [ipv6-mostly](@/posts/ipv6-mostly.md), last weekend I enabled this on my main network segment as it had all gone well on the guest network.

I had half expected a bunch of things to stop working in some way...

Would the Hue app on my phone still talk to the Hue bridge?

Would the Alexa app on my phone still talk to the devices in my home?

I assumed a bunch of these things probably all make connections to a cloud service and communicate between themselves via that.

Nothing appeared to break.

Until I tried to cycle on Zwift.

Zwift seems to work fine, however, the companion app doesn't work as an in-game companion. It talks to Zwift all ok, shows events, activities, etc, just no companion.

I suspected that the game communicates its local/internal IP address to Zwift's servers, and then the companion app gets the IP from there and makes a local connection direct to the game.

However, the local IP that the game will find on the interface is an unroutable IPv4 address between `192.0.0.2` and `192.0.0.7` from the DS-Lite reserved IP addresses (see [RFC6333](https://datatracker.ietf.org/doc/html/rfc6333))

So, if the game communicates this via Zwift servers, the companion app is going to fail to make a connection to that.

I manually added a valid IPv4 address to both the Apple TV where the game is running, and my iPhone, and (with a game exit & restart) the companion started working just fine.

I've raise a support ticket with Zwift. I'll let you know...
