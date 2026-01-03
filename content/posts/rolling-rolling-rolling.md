+++
title = "Rolling, Rolling, Rolling..."
description = "The root zone is going to roll it's KSK following the RFC5011 defined process and method"
date = 2017-03-17
[extra]
toc = true
disclaimer = "This keyroll is long in the past now, but understanding the process is still useful"
[taxonomies]
tags = ["dns","rfc5011","keyroll","ksk","dnssec"]
+++

# Introduction

In October 2017, ICANN are going to roll the key signing key in the root of the DNS.

If you're not technical and don't know what I just said, this post isn't for you.

If, however, you run a validating recursive resolver, read on...

In October (the 11th to be exact), the key will roll and you'll need to have done one of two things...

* Update your root trust anchor manually
* Check your resolver is RFC5011 compliant.

But first, a little...

# Background...

So you know how DNSSEC works...

...you sign a zone. More specifically, you generate two keys, a key to sign the zone (ZSK), and a key to sign the keys (KSK). The zone gets bigger because for each record set, a signature is generated and added (RRSIG records). The public part of the keyset is also added to the zone (DNSKEY records). Some form of proof of non-existance is added (NSEC or NSEC3).

Next, once the keys and signatures have made it to all of the nameservers for the zone, you generate a delegated signer record (DS) from the KSK, and you publish that in the parent. The parent then signs the DS record, and hey presto, your chain of trust is made.

So, where's the DS record for the root...?

To make this chain of trust work, resolvers that want to validate the DNSSEC chain of trust need a starting point in the root...

Your resolver has a trust anchor for the root. Depending on what you're using for a resolver, this will either be the DS of the root KSK, or the public part of the KSK.

Your resolver will have this built in, but then, if configured correctly, will use an automatic mechanism to keep that key up to date and roll it when required.

# RFC5011

RFC5011 defines how a resolver can automatically update a trust anchor for a zone.

So that you can check whether your resolver will follow this process, ICANN have an automated testbed for the KSK roll, which I encourage you to look at.

# ICANN's Automated Test

Each week, they create a new zone, and they sign it with a set of newly generated keys. Purposefully broken DS records are published in the parent zone, so that a normal validating resolver will SERVFAIL (because validation fails).

By adding a trust anchor to your resolver, the zone will validate.

If correctly configured, your resolver will now look for new key signing keys, and will observe them, and use them as per RFC5011.

So, lets take a look at this. Before I add a trust anchor, I can check that the zone doesn't validate:

```hl_lines=7
$ dig @::1 2017-03-12.automated-ksk-test.research.icann.org soa

; <<>> DiG 9.9.5-9+deb8u6-Debian <<>> @::1 2017-03-12.automated-ksk-test.research.icann.org soa
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: SERVFAIL, id: 39100
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;2017-03-12.automated-ksk-test.research.icann.org. IN SOA

;; Query time: 1908 msec
;; SERVER: ::1#53(::1)
;; WHEN: Mon Mar 20 13:22:57 GMT 2017
;; MSG SIZE  rcvd: 77
```

We can see in line 7, that we have a <mark>SERVFAIL</mark> response.

This server is running BIND. So, first we check that the server is configured manage keys using RFC5011:

```hl_lines=3
options {
    ...
    dnssec-validation auto;
    ...
};
```

If you're just adding this, don't forget to `rndc reconfig`.

# Trust Anchor

Now, we need to add a trust anchor:

# BIND

```
managed-keys {
  2017-03-12.automated-ksk-test.research.icann.org initial-key 257 3 8
  "AwEAAa9qsSLDI+H0keqE3Yzdr6XuhqhBQVWw5xdgNoWLhE4VxSEIBz9I
  uCA4w4ssSrClZ59seNc76ltDFcKJv3X9jDjzRtBLjenIgV4n/3GpKrAA
  nRlYbUtpBEdlk4mxoL3BlX8pfLg7RQfTlWaxOUga1+CChcVieFF/si/e
  ePc9HpZbWxHZRLCAE8dlDa0aa0tfVAZWOnaifpmbTvhDK3tdvMU0tfG2
  YfsOYcFB9z2KWmCDYwCONNKtls3p6wMwolun1h8IYo0PF98vqjAp3NVR
  ZvKKdgyF/bZ/iJtAZFytXvXU6Gwa5tOm1wgP6wuKupscP8KHBluZyOSK
  w4RMTk6YBdE=";
};
```

This is added in your `named.conf` file.

Once again, don't forget to `rndc reconfig`.

# Unbound

If you're running Unbound, then you can add the DNSKEY or DS records to a file in a location that Unbound can read and write to (so, somewhere like `/var/lib/unbound/` and then add a `auto-trust-anchor-file` line in the `server:` section of your `unbound.conf` file.

```
cat /var/lib/unbound/2017-03-12.automated-ksk-test.research.icann.org.ds
2017-03-12.automated-ksk-test.research.icann.org. IN DS 3934 8 1 47AA8AAF4D75B3D9C58448F241F793EBC4977821
2017-03-12.automated-ksk-test.research.icann.org. IN DS 3934 8 2 0D27F2E6EA9CA548F1896A71FB07CED86074D3462F2A720D6177F3C5CEC15F0D
```

Note; the file doesn't look like this once you've told Unbound about it, as it uses the file to store metadata related to the RFC5011 process.

```
server:
    ...
    auto-trust-anchor-file: "/var/lib/unbound/2017-03-12.automated-ksk-test.research.icann.org.ds"
    ...
```

After adding those, you'll want to `unbound-control reload` to pick up the changes.

# Testing

```hl_lines=7 8
$ dig @::1 2017-03-12.automated-ksk-test.research.icann.org soa

; <<>> DiG 9.9.5-9+deb8u6-Debian <<>> @::1 2017-03-12.automated-ksk-test.research.icann.org soa
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 30413
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 3

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;2017-03-12.automated-ksk-test.research.icann.org. IN SOA

;; ANSWER SECTION:
2017-03-12.automated-ksk-test.research.icann.org. 60 IN	SOA ns1.research.icann.org. automated-ksk-test.research.icann.org. 1489968062 3600 600 86400 60

;; AUTHORITY SECTION:
2017-03-12.automated-ksk-test.research.icann.org. 60 IN	NS ns2.research.icann.org.
2017-03-12.automated-ksk-test.research.icann.org. 60 IN	NS ns1.research.icann.org.

;; ADDITIONAL SECTION:
ns1.research.icann.org.	3600	IN	A	192.0.34.56
ns2.research.icann.org.	3600	IN	A	192.0.45.56

;; Query time: 428 msec
;; SERVER: ::1#53(::1)
;; WHEN: Mon Mar 20 13:44:24 GMT 2017
;; MSG SIZE  rcvd: 181
```

This time, we can see that on line 7, we have a <mark>NOERROR</mark> response, and on line 8, we can see that we have <mark>ad</mark> in the `flags`.

# What's next...

Now, we wait. The next step is that ICANN's automated test lab will generate and publish a new KSK into the zone on the 19th.
