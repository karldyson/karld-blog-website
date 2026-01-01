+++
title = "Anycasting DNS"
date = 2015-01-18
[taxonomies]
tags = ["dns","bgp","anycast","network","junos","juniper","bind9","exabgp"]
+++

# Introduction...

I wanted to have a tinker with anycasting, and DNS seemed a sensible place to start, and easy to test and muck about with. So, I spun up a couple of DNS resolvers, and decided what my anycasted IP addresses would be. They need to be outside of the subnets I'm using on the rest of my network, as I want to route traffic to them. I've put the underlying machine's unicast addresses in this subnet too, but you wouldn't have to, depending on your set up.

# Servers...

The nameservers are, essentially, identical to servers that'd deal with unicast traffic, except for the following changes. I'm using BIND, but it really doesn't matter what you use.

We need to bind up the anycast addresses so that the O/S will deal with their traffic...

In my case, my anycasted addresses will be `10.1.53.1` and `10.1.53.2`, and I'm using Debian, so my additions to `/etc/network/interfaces` are:

```
auto lo:1
iface lo:1 inet static
address 10.1.53.1
netmask 255.255.255.255

auto lo:2
iface lo:2 inet static
address 10.1.53.2
netmask 255.255.255.255
```

We need to stop the machine responding to ARP for these. Actually, we tell it to stop responding to ARP requests unless the interface the ARP arrives on matches the ARP'd for IP, so because we've bound them up to the loopback, we don't want the machine to respond via `eth0`, for example, so I added the following to `/etc/sysctl.conf`:

```
net.ipv4.conf.eth0.arp_ignore = 1
net.ipv4.conf.eth0.arp_announce = 2
```

# BGP & Load Balancing...

Now we need to advertise the anycast addresses to our router. In this case, we'll use BGP to do this. To do that, we'll use ExaBGP. Grab that and install it on the server, and then the config looks something like this. My router is `10.1.53.254`, and my two nameservers live in `10.1.53.0/24`

```
neighbor 10.1.53.254 {
  router-id 10.1.53.11;
  local-address 10.1.53.11;
  local-as 64601;
  peer-as 64601;
  hold-time 10;

  process watch-nameserver {
    run /usr/local/bin/nameserver_watchdog;
  }

  static {
    route 10.1.53.1/32 next-hop 10.1.53.11 watchdog anycastdns withdraw;
    route 10.1.53.2/32 next-hop 10.1.53.11 watchdog anycastdns withdraw;
    route xxxx.xxxx.xxxx:53::1/128 next-hop xxxx.xxxx.xxxx:53::11 watchdog anycastdns withdraw;
    route xxxx.xxxx.xxxx:53::2/128 next-hop xxxx.xxxx.xxxx:53::11 watchdog anycastdns withdraw;
  }
}
```

I withdraw the routes from the outset, so that the watchdog will announce them upon successful testing.

The router's BGP config looks like this (it's JunOS):

```Junos
# show protocols bgp group dns-anycast
local-address 10.1.53.254;
hold-time 10;
family inet {
    unicast;
}
family inet6 {
    unicast;
}
peer-as 64601;
local-as 64601;
multipath;
neighbor 10.1.53.11;
neighbor 10.1.53.12;
```

I'm going to equally load balance between the two servers, but you could set a localpref on each server, for example, and have server1 handle .1 primarily with server2 taking over in the event of failure, and vice versa.

Don't fall for JunOS' misleading 'per packet' configuration item; this will, despite appearances, load balance per flow based on a hashing algorithm.

```Junos
# show routing-options forwarding-table
export dns-anycast-loadbalance;

# show policy-options policy-statement dns-anycast-loadbalance
then {
    load-balance per-packet;
}
```

# Monitoring and Health...

We've included a watchdog in the ExaBGP config. Without this, clearly if the nameserver fails entirely, then the BGP session will be torn down, and the traffic directed to the other host. However, if the nameserver daemon fails, then the BGP session will remain, and traffic will be disrupted. Therefore, there's a watchdog that'll check that the nameserver daemon is listening, and will perform a lookup against it, announcing the anycast address(es) while it's up, and withdrawing them in the event of failure. The watchdog looks like this:

```perl
#!/usr/bin/perl

use strict;

my $debug = 0;

unless($debug) {
	$SIG{'INT'} = sub {};
}
select STDOUT;
$| = 1;

use IO::Socket;
use Net::DNS;

my $state = 'init';

my $ip;
my $domain;
if(open(C,"/etc/nameserver_watchdog.conf")) {
	chomp(($ip, $domain) = split /:/, <C>);
	close C;
} else {
	$ip = '127.0.0.1';
	$domain = 'localdomain';
}
print "checking $ip for $domain\n" if $debug;

while(1) {
	eval {
		local $SIG{ALRM} = sub { die 'Timed Out'; };
		alarm 2;
		print "attempting connect... state is [$state]\n" if $debug;
		my $socket = IO::Socket::INET->new(Proto=>'tcp', PeerAddr=>$ip, PeerPort=>53, Timeout=>2);
		if($socket && $socket->connected() && do_lookup($ip, $domain)) {
			print "announce watchdog anycastdns\n" if $state ne 'up';
			$socket->close();
			alarm 0;
			$state = 'up';
			print "state set to up\n" if $debug;
		} else {
			print "withdraw watchdog anycastdns\n" if $state ne 'down';
			$state = 'down';
			print "state set to down\n" if $debug;
		}
	};
	if($@) {
		print "state is [$state]\n" if $debug;
		print "withdraw watchdog anycastdns\n" if $state ne 'down';
		$state = 'down';
		print "state set to down in barf\n" if $debug;
	}
	alarm 0;
	sleep 10;
}

sub do_lookup {
	my $ip = shift;
	my $domain = shift;
	my $r = Net::DNS::Resolver->new;
	$r->nameservers($ip);
	$r->tcp_timeout(5);
	$r->udp_timeout(5);
	my $q = $r->query($domain,'SOA');
	my $found = 0;
	print "Answer: ".($q->answer)[0]->serial."\n" if $debug;
	$found++ if ($q->answer)[0]->serial =~ m/^\d+$/;
	if($debug > 1) {
		require Data::Dumper;
		print Data::Dumper::Dumper($q)."\n\n";
	}
	return 1 if $q && $found;
	print "Error:\n" if $debug;
	print $r->errorstring if $debug;
	print "\n===\n" if $debug;
	return 0;
}
```

`/etc/nameserver_watchdog.conf` contains lines of the format `ip.ad.dr.ess:domain.com`.

It'll announce the address in the event that a tcp connection succeeds as well as a DNS lookup that you'd expect the server should answer or be permitted to recurse for you. If the DNS daemon stops responding the watchdog will withdraw the routes; if the server fails, the BGP session will fail, and the route will be withdrawn anyway.
