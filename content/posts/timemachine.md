+++
title = "TimeMachine"
description = "...after the Synology died, one of the things that needed replacing was TimeMachine..."
date = 2025-12-18
[extra]
toc = true
[taxonomies]
tags = ["macos","timemachine","samba","linux"]
categories = ["Technical"]
+++

# Introduction

I have a MacBook and whilst some of the things I care about get dropped in iCloud and magic'd away onto someone else's computer for safe keeping and sync'ing to my other Apple devices, I like to back it up too. The Synology does (did?) TimeMachine and so the first thing to replace was that.

I googled and read a bunch of articles, and installed samba and avahi-daemon.

I added the following to the bottom of `/etc/samba/smb.conf`

```
[TimeMachine]
   comment = Time Machine Backup
   path = /srv/timemachine
   browseable = yes
   writable = yes
   valid users = my_username
   create mask = 0600
   directory mask = 0700
   spotlight = yes
   vfs objects = catia fruit streams_xattr
   fruit:aapl = yes
   fruit:time machine = yes
```

...and I set my samba password with `sudo smbpasswd -a my_username`

Now, this RPi is in a different VLAN to my main client LAN, and so whilst I'm sure the Mac would spot this in Finder automagically if they were in the same VLAN, or, if I was bouncing mDNS to and from the network services VLAN, <kbd>⌘ Command</kbd> + <kbd>K</kbd>, typing `smb://the.fqdn.of.the.rpi` into the address bar and hitting Connect worked just fine (after adding the appropriate firewall configuration to permit tcp/445 from the Mac to the RPi.

I authenticated and then opened the TimeMachine control panel. The rest worked just as you'd expect, and a backup kicked off soon after.

# ...the SRX config...

Just in case you're wondering; this is the slightly redacted config for the SRX...

```Junos
# edit security policies from-zone <zone where Mac lives> to-zone <zone where RPi lives>;
# show policy timemachine
match {
    source-address timemachine-clients;
    destination-address timemachine-servers;
    application timemachine;
}
then {
    permit;
}

# top
# show applications application-set timemachine
application tcp-445;

# show applications application tcp-445
protocol tcp;
destination-port 445;
```
