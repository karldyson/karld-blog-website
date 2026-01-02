+++
title = "tcp/53 isn't just for AXFR"
description = "You're not just opening UDP/53 for DNS, are you...?"
date = 2010-03-18
[extra]
toc = false
toc_inline = true
toc_ordered = true
styles = ["posts/demo.css"]
[taxonomies]
tags = ["rfc","axfr","tcp","dns"]
+++

The internet has a defined set of rules known as RFCs. They work together to make sure that all participants of the internet community are working in the same way, and that the things they do as part of that community will work with, and interact correctly with the things that others do.

[RFC1035 section 4.2.1](https://datatracker.ietf.org/doc/html/rfc1035#section-4.2.1) (UDP Transport) states:

> Messages carried by UDP are restricted to 512 bytes (not counting the IP or UDP headers). Longer messages are truncated and the TC bit is set in the header.
 
You then fall back to TCP and repeat your query to get the full response.
 
Yes, AXFR queries are TCP, but TCP isn't exclusively AXFR!
 
Assuming that, because you won't be doing any transfers and therefore don't need to allow <mark>tcp/53</mark>, is wrong, and will invariably involve you having issues with some service or other, due to not getting the correct information from DNS.

I'll give you an example:

You use a service that, for whatever reason, decides that as a basic form of load balancing, to use multiple A records for the 'name' you've queried. So, you ask for `www.example.com` and, rather than give you back a single A record, they give you one for each of their servers. This could easily become longer than 512 bytes, the answer will be truncated, and you *should* repeat your request using TCP. Your resolver knows this, and will automatically do it.

If you've blocked <mark>tcp/53</mark> on your firewall, it'll fail. You'll sit, staring at your computer, thinking that `www.example.com` has fallen over, failed in some way, when they have not. It's not their fault that you've not followed the rules.

It's not just website related records either, email related records (MX, or the TXT for DKIM, DomainKeys or SPF) are other great examples of this.... block <mark>tcp/53</mark> from your mail server, and you could quite easily find yourself not receiving email from some senders.

[edited later to add the following]

Further reading (thanks to Duncan for the pointer) of [RFC1123 section 6.1.3.2](https://datatracker.ietf.org/doc/html/rfc1123#section-6.1.3.2) (Transport Protocols) states:

> DNS resolvers and recursive servers MUST support UDP, and
> SHOULD support TCP, for sending (non-zone-transfer) queries.
> Specifically, a DNS resolver or server that is sending a
> non-zone-transfer query MUST send a UDP query first. If the
> Answer section of the response is truncated and if the
> requester supports TCP, it SHOULD try the query again using
> TCP.

I guess I'm a little disappointed to see "SHOULD" instead of "MUST", but given the document was written in 1989, I think today, it should be read as "SHOULD, if you want it to work". It does go on to say:

> Whether it is possible to use a truncated answer depends on the application. A mailer must not use a truncated MX response, since this could lead to mail loops.
