+++
title = "Death of the Synology...?"
description = "...I think there's a strong chance that my Synology is dead..."
date = 2026-01-11
[extra]
toc = false
[taxonomies]
tags = ["synology"]
categories = ["Technical"]
+++

I had a text a couple of weeks before Christmas from a friend who lives at the other end of the village, asking if the power was out.

It wasn't for me, but a huge area the other end of the village was out and was due to be out for hours.

So, they came to visit for a few hours.

Anyway.

In the recent [TimeMachine](@/posts/timemachine.md) post I alluded to TimeMachine being one of the first things to replace after the Synology had died, or, at least, failed to come back.

Something happened that evening; some sort of power glitch. The lights didn't go out or flicker, but ~~some~~/~~most~~/all ? of the stuff connected to the UPS in the garage seems to have rebooted at the same time.

...and thankfully everything seems to have survivedi... ...apart from the Synology.

It's a DS1812+ and my inbox suggests I originally ordered it on January 10<sup>th</sup> 2013, so I guess it's had a fairly good innings.

Mostly, it's a secondary storage for data and not the primary location for anything so it's more an inconvenience than anything.

Google, and a friend, suggest that it sitting there failing to spin up the drives whilst stubornly flashing it's little blue light means that it's likely unhappy rather than dead, so now that Christmas and New Year are behind us, I'll open it up and see if any of the obvious gotchas are to blame.

Searching suggests the first places to check are:

* BIOS battery
* Disk on Module (DOM, internal storage for its operating system)
* PSU
* Failing RAM

Since electricity prices soared a few years ago now, and I switched off the lovely, but very thirsty servers running my virtual machines, anything that I want to run home here typically runs on some flavour of Raspberry Pi. They're small, do the job, and when it comes to electricity, they are quite the opposite, even in what is now becoming a small group, compared to a Dell R720XD full of 15k SAS drives.

The oldest of the Raspberry Pi's are getting long in the tooth and so I'm working on replacing a couple of them.

As mentioned elsewhere, the oldest, I think, is the one doing the old temperature monitoring, currently being replaced with a collection of Pico 2W units. There are [other posts](/tags/temperature) here that talk about that.

The next oldest two are a general workhorse RPi4 that kinda does some monitoring, is the auth DNS server for my internal DNS zone, and the RPiB that does NTP. I wrote recently [about the new stratum zero time server](@/posts/stratum-zero-ntp-server/index.md)

With the possible death of the Synology, I wanted to take time to think about whether I *need* a NAS, or whether other solutions would work better for my needs, and so in the mean time, I got on with replacing the workhorse and the time server in one go, adding a bunch of extra storage for [TimeMachine](@/posts/timemachine.md).

As you'll know, I'm a bit of a DNS geek, and I also have DNS related project underway that will gradually use some of the storage (circa 3GB per day), so I'll know what the longer term plan is for the Synology before I need to work out whether I need to put more storage in the new RPi.

Sorry if a search engine sent you this way hoping for an answer; I'm trying to be better (or at least less bad) at writing these things up, and so I'll come back and write up what I find soon...

