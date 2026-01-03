+++
title = "Cost Effective Temperature Monitoring"
description = "Temperature monitoring with DS18B20 1-wire bus sensors and a raspberry pi"
date = 2012-08-20
updated = "2017-10-23"
[extra]
toc = false
[taxonomies]
tags = ["linux","temperature","owfs","one-wire","monitoring","raspberry-pi","perl","munin"]
+++

<figure>
{{ image(url="temperature-graph.png") }}
<figcaption>Temperature Graph produced by Munin</figcaption>
</figure>

I'd been thinking about some temperature monitoring at home for a while, particularly in the nursery, so I can see if it's getting too hot or cold in there while I'm asleep.

When my Raspberry Pi arrived, I decided to put it to work.

After quite a bit of googling & reading, I found that the one wire file system stuff was easy to install, and had been done by others (See references below).

Whilst it's possible to go and buy the individual components, and I'm quite capable with a soldering iron, I have to admit to taking the lazy route and purchased some bits from [Sheepwalk Electronics](http://sheepwalkelectronics.co.uk/)

I wanted to monitor temperature in the garage, outside, the loft, the front bedroom, and the small bedroom on the back of the house (currently, the nursery).

I also monitor temperature in the lounge, but that comes from the Current Cost unit that I also monitor electricity usage with.

So, I'd need a host adaptor, and a sensor for each of the places to be monitored.

The Sheepwalk sensor modules are made up, or kit form, and include RJ45 sockets to ease putting together a network with standard cat5 cable. The sensors can either be parasitic and pull power from the bus, or can have power supplied. I've set mine up as parasitic, and it's working OK so far.

One wire is best suited to a linear network of devices strung together, but seems quite robust, and is working in a star arrangement.

I purchased Sheepwalk's SWE2+SWE0 pack which comes with an SWE2 (basically, a sensor plus 6 RJ45 sockets) and four SWE0 (a basic sensor on the end of 2m of cat5). I also purchased a USB host adaptor, and the RJ11 to RJ45 cable, although this would be simple to make up. Lastly, I picked up some RJ45 couplers, as they're a good price and I'd be sure to need some!

I installed [Raspbian](http://www.raspberrypi.org/downloads) (the official/recommended linux distro for the Raspberry Pi) on the SD card and booted the Raspberry. I gave it a static IP on my network so I could probe it from the Munin instance later on without fear of the IP changing.

After inserting the USB adaptor, I installed OWFS

```
kdyson@rpi1 ~ $ apt-get install owfs
```

There's no init script for owfs itself, only for the supporting services, but [Neil Baldwin's page](http://neilbaldwin.net/blog/weather/raspberry-pi-data-logger/) that I'd been reading had one, which saved me the hassle of writing one.

With that started, I plugged in the RJ11 to RJ45 lead, and plugged the SWE2 into the end.

A fresh `ls -al /mnt/1wire/` showed the new sensor, and

```
kdyson@rpi1 ~ $ cat /mnt/1wire/28.873AC4030000/temperature
31.5625
kdyson@rpi1 ~ $
```

I tested the rest of the sensors, before hooking them up where I wanted them. I was a little concerned that the cable length might be a problem to the furthest sensor - the sensor in the loft is about 25m of cable from the SWE2, and is strung off the SWE1 that provides the "Front Bedroom" sensor.

Lastly, I hacked up a munin plugin. All I needed was the `munin-node` package (`apt-get install munin-node`) as my remote `munin` instance would be collecting the data and producing the graphs (it's already doing things like the already mentioned temperature in the lounge, electricity usage, and more mundane things like the SNR & sync rate of my ADSL line from my router).

So, in summary:

* Raspberry Pi + SD Card + Raspbian + OWFS + Munin-Node
  * USB Host Adaptor + RJ11 to RJ45 Lead
    * SWE2 (Garage)
      * SWE0 (Garage Outside)
      * SWE1 (Front Bedroom)
        * SWE0 (Loft)
      * SWE0 (Small Bedroom)

# Code Snippets

First up is the munin plugin. I took a chunk of inspiration from [http://err.no/personal/blog/2010/Nov/02](http://err.no/personal/blog/2010/Nov/02) but the perl OWNet module was playing up for me, and given it was 10pm, I took the lazy route and hacked it about to just look around the file system. You can tell how lazy I was, because you can see I used `glob()` instead of `opendir()` etc. I'm not proud of it, but it's working.

You'd need to drop this either directly in `/etc/munin/plugins` or someplace else and symlink it to there. I did the latter.

```perl
#!/usr/bin/perl

#%# family=auto
#%# capabilities=autoconf

use strict;
use warnings;
use utf8;

my $debug = 0;

chdir "/mnt/1wire/bus.0";

my @buses = grep { /bus./ } glob("*");

print "got buses ".join(", ", @buses)."\n" if $debug;

my %sensors;
my %sensor_names;
for my $bus (@buses) {
  for my $sensor (glob($bus."/*")) {
    print "got sensor $sensor\n" if $debug;
    my $p = $sensor;
    $sensor =~ s|^bus.\d+/||;
    my $sensor_name = $sensor;
    $sensor_name =~ s|\.||;
    next if $sensor =~ /(interface|alarm|simultaneous)/;
    next unless $sensor =~ /^28\./;
    $sensors{$sensor} = $p;
    $sensor_names{$sensor} = $sensor_name;
  }
}

my %labels;
if(open(A, '/etc/owfs-aliases')) {
  while(<a>) {
    chomp;
    print "got label $_\n" if $debug;
    my($s,$l) = m/^([\da-zA-Z\.]+)\s*=\s*(.+)$/;
    $labels{$s} = $l;
  }
  close A;
}

use Data::Dumper;
print Dumper(\%sensors)."\n" if $debug;
print Dumper(\%labels)."\n" if $debug;

if (defined $ARGV[0]) {
  if ($ARGV[0] eq 'autoconf') {
    if (-d "/mnt/1wire/bus.0") {
      print "yes\n";
      exit 0;
    }
    print "no\n";
    exit 1;
  } elsif ($ARGV[0] eq 'config') {
    print "graph_title Temperature\n";
    print "graph_args --base 1000\n";
    print "graph_vlabel Temp in °C\n";
    print "graph_category sensors\n";
    print "graph_info This graph shows the temperature in degrees Celsius of the sensors on the network.\n";
    print "$sensor_names{$_}.label $labels{$_}\n" foreach (keys %sensors);
    exit 0;
  }
}

for my $sensor (keys %sensors) {
  print "sensor is $sensor\n" if $debug;
  if ($0 =~ /ow_([\w\.]+)/) {
    print "1 is $1\n" if $debug;
    next unless ($1 eq $sensor || $1 eq $labels{$sensor});
  }
  my $file = "/mnt/1wire/bus.0/".$sensors{$sensor}."/temperature";
  my $temp = `cat $file`;
  chomp $temp;
  printf "%s.value %.4f\n", $sensor_names{$sensor}, $temp;
}

exit 0;
```

Next up, the aliases file `/etc/owfs-aliases` - again, it was late, owfs aliases didn't appear to be working quite how I was expecting, and so I hacked the plugin to use this file:

```
28.5086C4030000 = Front_Bedroom
28.7D61C4030000 = Garage
28.684EC4030000 = Garage_Outside
28.873AC4030000 = Loft
28.C786C4030000 = Small_Bedroom
```

