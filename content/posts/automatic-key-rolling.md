+++
title = "Automatic Key Rolls"
date = 2020-11-22
[extra]
toc = false
toc_inline = true
toc_ordered = true
styles = ["posts/demo.css"]
[taxonomies]
tags = ["dns","keyroll","ksk","zsk","csk","dnssec"]
+++

I recently moved my test domains onto a separate DNS master so that I could more freely tinker with these domains without risk to my regular stable domains.

I use catalog zones (maybe this is for another post!) to distribute the zones to save myself the bother of having to configure all the slaves, particularly since I've recently started spinning up an experimental anycast network of virtual machines, and so I added a second catalog to my slave servers, and away we went.

I'm a keen user of debian, and so I built the test master on 'sid' so I could run bleeding edge.

The benefit, primarily, was that whereas Debian 10 gets BIND 9.11.5, sid gets 9.16.8 (at the time of writing) as well as a newer version of openssl, meaning I could sign a zone with algorithm 16, ED448. DS digest algorithm 4 (SHA-384) is also supported.

The biggy for me, though, and the main driver for having the newer version of BIND, was `dnssec-policy;` getting BIND to automatically roll your keys.

I'm not aware of an ability in this version to support either a hook to run a script to interact with your registrar to update a DS record when your KSK rolls, nor an ability to automate CDS or CDNSKEY, but they'll be coming at some point in the future.

I decided to test this by auto-rolling my ZSK, so, I added the following to `/etc/bind/named.conf.options` :

```
dnssec-policy normal {
	dnskey-ttl PT1H;
	keys {
		ksk lifetime unlimited algorithm ecdsa384;
		zsk lifetime 90D algorithm ecdsa384;
	};
	max-zone-ttl P1D;
	parent-ds-ttl P1D;
	parent-propagation-delay PT1H;
	parent-registration-delay P1D;
	publish-safety PT1H;
	retire-safety PT1H;
	signatures-refresh P5D;
	signatures-validity P2W;
	signatures-validity-dnskey P2W;
	zone-propagation-delay PT5M;
};
```

You don't need to create initial keys or anything; BIND will do all the key juggling automatically, and will store them wherever you set `key-directory` to in your `options` section.

It doesn't matter how you add your domains; I use `rndc addzone` as I have scripts that automate this as well as adding the zone to the catalog (again, that'll be in another post at some point), the key thing being you just specify the policy in the zone. I'm also using `inline-signing;` whether you do will depend on your setup. Here's a sample:

```
zone "some.zone" {
    type master;
    file "/path/to/some.zone.file";
    dnssec-policy "normal";
    inline-signing yes;
};
```

...and as if by magic, `rndc reconfig`, and the keyfiles are created, and the zone signed.
