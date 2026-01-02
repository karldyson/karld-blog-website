+++
title = "DNSSEC BIND9 Configuration Summary and Cool Stuff"
description = "How to DNSSEC sign your DNS zones with BIND9"
date = 2010-08-20
[extra]
toc = false
toc_inline = true
toc_ordered = true
styles = ["posts/demo.css"]
[taxonomies]
tags = ["rfc","axfr","tcp","dns"]
+++

# Introduction

With the recent signing of the root, I've discovered a sudden interest in DNSSEC, and decided to have a go myself to aid my understanding of it.

This article is written as an aid-memoir to me, and summary of the bits I've read. Of course, I've provided links to the whole blog entries I found the information in, in case you want to read more than I've written.

Whilst the root is signed, only certain TLDs are signed, and so if you want the full chain of trust experience, you want a domain with a signed TLD.

At the moment, `.uk` is signed, but `.co.uk` etc are not, so that rules them out. `.net` is scheduled for around Nov 2010, and `.com` sometime around March 2011.

`.org`, however, is already signed, and so I thought I'd grab one to play with.

# Not All Registrars Are Equal

I used my regular registrar, and registered `karldyson.org` to go with my collection of `.com` and `.co.uk` versions.

This was my first sticking point, because their upstream (Tucows) aren't accredited for DNSSEC yet (and, it would appear, have no plans on doing so).

I'd need my domain to be with a registrar that is accredited.

My registrar helpfully supplied me with a list of registrars that are, so I could choose one and either register a domain there, or move my new one.

I registered another .org to add to another set, this time with GoDaddy. They're on the list.

# Signing The Zone

I had told GoDaddy that I wanted to use my own nameservers during sign up, and so after creating a regular zonefile for bind, I had a look through the blog entry I found at [http://clayshek.wordpress.com/2009/01/13/enabling-dnssec-on-bind/](http://clayshek.wordpress.com/2009/01/13/enabling-dnssec-on-bind/)

Essentially, the steps are (all completed whilst IN the zonefile directory):

* Generate a zone signing key (ZSK) :
> `dnssec-keygen -a RSASHA1 -b 1024 -n ZONE example.org`
* Generate a key signing key:
> `dnssec-keygen -a RSASHA1 -b 2048 -n ZONE -f KSK example.org`
* Concatenate the created public keys into the zone file:
> `cat Kexample.org+*.key >> example.org`
* Sign any child zones first:
> `dnssec-signzone -N INCREMENT child.example.org`
* Concatenate the DS records for the child into the parent zone:
> `cat dsset-child.example.org >> example.org`
* Sign the zone:
> `dnssec-signzone -N INCREMENT example.org`

Generating the ZSK and KSK took ages on my Atom 330 dedicated server, and so I can recommend a good book, or some other talk while you wait for this to finish!

Like the child zone signing, you will get DS records for the parent zone. These need to be supplied to your registrar to maintain the chain of trust. GoDaddy has a nice interface for submitting these, you just need to know what the different bits of the DS records are. They're detailed in [RFC4034](https://datatracker.ietf.org/doc/html/rfc4034) but to save you some time, and sanity....

# Your DS Records

`example.org. 86400 IN DS 60485 5 1 2BB183AF5F22588179A53B0A98631FAD1A292118`

The first four text fields specify the:

1. name `example.org.`
1. TTL `86400`
1. Class `IN`
1. RR type `DS`

Value `60485` is the key tag for the corresponding `example.org.` <kbd>DNSKEY</kbd> RR

Value 5 denotes the algorithm used by this `example.org.` <kbd>DNSKEY</kbd> RR.

Value 1 is the algorithm used to construct the digest.

The rest of the RDATA text is the digest in hexadecimal.

# Your Caching Resolver

Your caching resolver will need DNSSEC enabled for queries. I added the following to my bind server's options section:

```
dnssec-enable yes;
dnssec-validation yes;
```

# Your System Resolver

With your local system pointed at your caching resolver, it would appear you'll need <mark>EDNS0</mark> enabled. This is achieved by adding the following option to your `/etc/resolv.conf`.

```
options edns0
```

This appears to be supported on newer versions of `libresolv` - my Debian 5 system doesn't appear to support it, whereas my Ubuntu 10.04 system does.

# So, on to the cool stuff... SSH

..and so, at last, on to the cool stuff.

Given you can now cryptographically trust DNS, you can do something interesting. Rather than need to verify all the SSH fingerprints, you can store them in DNS and have your SSH client automagically verify that all is well.

I followed a set of instructions I found at [http://blog.exanames.com/2009/06/one-more-thing-to-do-with-dnssec-ssh.html](http://blog.exanames.com/2009/06/one-more-thing-to-do-with-dnssec-ssh.html), and as before, here's a summary.

Run the following two comands on each host you'd like to generate fingerprints for:

```
ssh-keygen -r `hostname`. -f /etc/ssh/ssh_host_rsa_key
ssh-keygen -r `hostname`. -f /etc/ssh/ssh_host_dsa_key
```

This will generate two <kbd>SSHFP</kbd> records that you will need to include in the zonefile, then you can re-sign and re-publish the zone.

In my case, the records generated were for .co.uk variants of the hostname, but I found no problems changing them to .org

You'll then need to persuade SSH to perform verification using DNS. I did this by adding the relevant option to `/etc/ssh/ssh_config`

```
VerifyHostKeyDNS yes
```

There, you're done. You should now be able to ssh to the host(s) concerned without needing to manually verify the fingerprints.
