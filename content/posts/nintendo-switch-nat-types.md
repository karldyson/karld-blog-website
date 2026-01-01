+++
title = "Nintendo Switch NAT Types"
description = "Tinkering with NAT on my firewall to see if I can improve the Nintendo Switch detected NAT type"
date = 2025-04-25
[taxonomies]
tags = ["Nintendo","Switch","Network","NAT","Juniper","SRX"]
categories = ["Technical"]
+++

Like lots of people, my daughter has a Nintendo Switch.

A few weeks ago she came to me because a game she was trying to play online was complaining about the NAT type.
So we had a look in the network connection test and found it was reporting <b>NAT Type D</b>.

I have a Juniper SRX here (of course) and the Switch was just using the general outbound source NAT that looks a little like this (actual IPs redacted, naturally).

```Junos
> show configuration security nat source rule-set general rule general
match {
    source-address 192.0.2.0/24;
}
then {
    source-nat {
        pool {
            general;
        }
    }
}
```

The pool is just a very basic pool with a simple address specified.

```Junos
> show configuration security nat source pool general
address {
    198.51.100.100/32;
}
```

Some googling later and I find that this seems to be because port translation is in use.

Seems the Nintendo does not like that.

The Switch already has a static DHCP lease so it always gets the same IP.

I’m fortunate to have a /29 from my ISP and I had an IP spare, so I created a new pool for the Nintendo with port translation disabled.

```Junos
> show configuration security nat source pool nintendo-switch
address {
    198.51.100.101/32;
}
port {
    no-translation;
}
```

Good news. This gets the NAT Type up to **NAT Type B** and the game started working.

But this got me to thinking. What was needed for **NAT Type A**…?

Given nothing else was using the public IP, I altered the NAT configuration to a static NAT.

```Junos
> show configuration security nat static rule-set general rule nintendo-switch
match {
    destination-address 198.51.100.101/32;
}
then {
    static-nat {
        prefix-name {
            nintendo-switch;
        }
    }
}
```

This, however, *still* results in **NAT Type** ***B***.

There seemed to be two obvious options remaining.

1. Type A is actually “No NAT at all”

1. Type A is “there’s effectively no firewalling“

Testing option 2 was quicker and simpler than 1.

As much as I detest `any-any` type policies, the outbound policy all along for the Switch was a basic “allow the Switch outbound to the internet” policy.

So I added an `any-any` inbound policy permitting anything inbound to the Switch.

Bingo! **NAT Type A**.

So, that’s horrible.

Given **NAT Type B** is good enough for 99 point something percent of things, that `any-any` inbound policy was disabled as soon as the test completed.

I will have to have a little faff with the network so that I can drop the Switch into a VLAN where I can give it a public IP directly so there’s actually no NAT at all, and see what it says then.

I just thought this might be useful to someone.

Thanks for reading.
