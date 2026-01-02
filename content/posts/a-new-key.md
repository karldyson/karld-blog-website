+++
title = "A New Key..."
description = "The next step of the RFC5011 keyroll of the root zone KSK"
date = 2017-03-21
[extra]
toc = true
toc_inline = true
toc_ordered = true
styles = ["posts/demo.css"]
disclaimer = "This keyroll is long in the past now, but understanding the process is still useful"
[taxonomies]
tags = ["dns","rfc5011","keyroll","ksk","dnssec"]
+++

# Introduction

Further to my post on [ICANN's automated KSK testlab](@/posts/rolling-rolling-rolling.md), ICANN generated a new key on the 19th, and added it to the test zone that we're using, and we can see it below:

# The New Key

```hl_lines=36-45
$ dig +multiline @::1 2017-03-12.automated-ksk-test.research.icann.org dnskey

; <<>> DiG 9.9.5-9+deb8u6-Debian <<>> +multiline @::1 2017-03-12.automated-ksk-test.research.icann.org dnskey
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 36605
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;2017-03-12.automated-ksk-test.research.icann.org. IN DNSKEY

;; ANSWER SECTION:
2017-03-12.automated-ksk-test.research.icann.org. 60 IN	DNSKEY 257 3 8 (
				AwEAAa9qsSLDI+H0keqE3Yzdr6XuhqhBQVWw5xdgNoWL
				hE4VxSEIBz9IuCA4w4ssSrClZ59seNc76ltDFcKJv3X9
				jDjzRtBLjenIgV4n/3GpKrAAnRlYbUtpBEdlk4mxoL3B
				lX8pfLg7RQfTlWaxOUga1+CChcVieFF/si/eePc9HpZb
				WxHZRLCAE8dlDa0aa0tfVAZWOnaifpmbTvhDK3tdvMU0
				tfG2YfsOYcFB9z2KWmCDYwCONNKtls3p6wMwolun1h8I
				Yo0PF98vqjAp3NVRZvKKdgyF/bZ/iJtAZFytXvXU6Gwa
				5tOm1wgP6wuKupscP8KHBluZyOSKw4RMTk6YBdE=
				) ; KSK; alg = RSASHA256; key id = 3934
2017-03-12.automated-ksk-test.research.icann.org. 60 IN	DNSKEY 256 3 8 (
				AwEAAbqImB5UsfE5J/sx3L3uQxjSY5HIPjrlTFKA+cxE
				R8SmU1wWGo21nrNBm3pOIYoC3zhiaCq1Jo6XrTcg+In+
				62g7PeXBO+2QBoHzCBxqbFMPoGpHph7D/OebWOvw5Akz
				MFqus2/JxZtvJOgkBws1EbzOw/lKbJUZVStUiCOZ8wFP
				Xd3X7nQMjVTOu6Cb2uGAVrgBRsARo+2CdcXNEtzNTHU1
				c+VxH9G/t/2VCrueDmr/epUP1adkyNUmXoYaG3eMrdGr
				ml8Dr7OMrt40vlWFp6i3TxltDXG/navXdEmL/w6f+pA6
				Dt9KVw/iEUxB08+4VY6jMkxfWJAD6t5XwCVcKH8=
				) ; ZSK; alg = RSASHA256; key id = 19401
2017-03-12.automated-ksk-test.research.icann.org. 60 IN	DNSKEY 257 3 8 (
				AwEAAfUtjasCuLysD4MbjG3v4Kyu0vvVJ/0cIreP6flt
				MeZmwQ5SRta/mB+eFVjau+6YKra2UeTKxojBovHH2lZr
				w7NNejL44/Xps4gR3LSVMnCdwras+yvj4en64ghRGWYO
				uB+Icb0AqrCUhLFWR8yx41UkfaA2vzFnM2xTx0N0+o6R
				6UciWuwJResomQupOjNUy2ZAi81Y3pb0x3Lw4POjpcSJ
				zrK4aZ/5UPymplqhLEU2DsoQmyFlM5RNTt0YXR8XM4Yw
				su/scxg0u00IF1GC8xcyZUTMc1Rz98AY1VUo5QqUp9Vb
				Aed5Aw1nNYfjLTj+zOykedgmjms1iNgh9EY111c=
				) ; KSK; alg = RSASHA256; key id = 19741

;; Query time: 285 msec
;; SERVER: ::1#53(::1)
;; WHEN: Tue Mar 21 20:17:12 GMT 2017
;; MSG SIZE  rcvd: 905
```

# Troubleshooting

Key <mark>19741<mark> is a new KSK in the zone.

## BIND9

If you look in `managed-keys.bind` (I'm running Debian, and so that's in `/var/cache/bind/`) you'll now see the new key is visible while BIND is observing the new key. RFC5011 defines the period that the resolver must observe the new key for as either at least two times the TTL of the keyset containing the new key, or 30 days; whichever is the longer.

I'm cheating, slightly, and taking a look at `managed-keys.bind` from a different server, because my Debian box is running BIND 9.9.5, whereas I have access to a 9.11 box; you'll see why below:

```hl_lines=21-22 33-34 45-46
$ cat /var/named/managed-keys.bind
$ORIGIN .
$TTL 0	; 0 seconds
@			IN SOA	. . (
				284        ; serial
				0          ; refresh (0 seconds)
				0          ; retry (0 seconds)
				0          ; expire (0 seconds)
				0          ; minimum (0 seconds)
				)
			KEYDATA	20170322222551 20170210095625 19700101000000 257 3 8 (
				AwEAAagAIKlVZrpC6Ia7gEzahOR+9W29euxhJhVVLOyQ
				bSEW0O8gcCjFFVQUTf6v58fLjwBd0YI0EzrAcQqBGCzh
				/RStIoO8g0NfnfL2MTJRkxoXbfDaUeVPQuYEhg37NZWA
				JQ9VnMVDxP/VHL496M/QZxkjf5/Efucp2gaDX6RS6CXp
				oY68LsvPVjR0ZSwzz1apAzvN9dlzEheX7ICJBBtuA6G3
				LQpzW5hOA2hzCTMjJPJ8LbqF6dsV6DoBQzgul0sGIcGO
				Yl7OyQdXfZ57relSQageu+ipAdTTJ25AsRTAoub8ONGc
				LmqrAmRLKBP1dfwhYB4N7knNnulqQxA+Uk1ihz0=
				) ; KSK; alg = RSASHA256; key id = 19036
				; next refresh: Wed, 22 Mar 2017 22:25:51 GMT
				; trusted since: Fri, 10 Feb 2017 09:56:25 GMT
2017-03-12.automated-ksk-test.research.icann.org KEYDATA 20170321232551 20170317172529 19700101000000 257 3 8 (
				AwEAAa9qsSLDI+H0keqE3Yzdr6XuhqhBQVWw5xdgNoWL
				hE4VxSEIBz9IuCA4w4ssSrClZ59seNc76ltDFcKJv3X9
				jDjzRtBLjenIgV4n/3GpKrAAnRlYbUtpBEdlk4mxoL3B
				lX8pfLg7RQfTlWaxOUga1+CChcVieFF/si/eePc9HpZb
				WxHZRLCAE8dlDa0aa0tfVAZWOnaifpmbTvhDK3tdvMU0
				tfG2YfsOYcFB9z2KWmCDYwCONNKtls3p6wMwolun1h8I
				Yo0PF98vqjAp3NVRZvKKdgyF/bZ/iJtAZFytXvXU6Gwa
				5tOm1wgP6wuKupscP8KHBluZyOSKw4RMTk6YBdE=
				) ; KSK; alg = RSASHA256; key id = 3934
				; next refresh: Tue, 21 Mar 2017 23:25:51 GMT
				; trusted since: Fri, 17 Mar 2017 17:25:29 GMT
			KEYDATA	20170321232551 20170418002534 19700101000000 257 3 8 (
				AwEAAfUtjasCuLysD4MbjG3v4Kyu0vvVJ/0cIreP6flt
				MeZmwQ5SRta/mB+eFVjau+6YKra2UeTKxojBovHH2lZr
				w7NNejL44/Xps4gR3LSVMnCdwras+yvj4en64ghRGWYO
				uB+Icb0AqrCUhLFWR8yx41UkfaA2vzFnM2xTx0N0+o6R
				6UciWuwJResomQupOjNUy2ZAi81Y3pb0x3Lw4POjpcSJ
				zrK4aZ/5UPymplqhLEU2DsoQmyFlM5RNTt0YXR8XM4Yw
				su/scxg0u00IF1GC8xcyZUTMc1Rz98AY1VUo5QqUp9Vb
				Aed5Aw1nNYfjLTj+zOykedgmjms1iNgh9EY111c=
				) ; KSK; alg = RSASHA256; key id = 19741
				; next refresh: Tue, 21 Mar 2017 23:25:51 GMT
				; trust pending: Tue, 18 Apr 2017 00:25:34 GMT
```

On my 9.9.5 server, I don't have the helpful comments. We can see, helpfully, that the root key (19036), and our original testlab key (3934) are trusted. We can also see that the server observing key 19741 because the instead of trusted since we can see trust pending.

## Unbound

If you remember from the original post, whereas BIND keeps a track in `managed-keys.bind`, Unbound tracks the metadata in the external file we specified with `auto-trust-anchor-file:`. The file has been updated in a similar way to BIND's:

```hl_lines=15 22
$ cat /var/lib/unbound/2017-03-12.automated-ksk-test.research.icann.org.ds
; autotrust trust anchor file
;;id: 2017-03-12.automated-ksk-test.research.icann.org. 1
;;last_queried: 1490135144 ;;Tue Mar 21 22:25:44 2017
;;last_success: 1490135144 ;;Tue Mar 21 22:25:44 2017
;;next_probe_time: 1490138421 ;;Tue Mar 21 23:20:21 2017
;;query_failed: 0
;;query_interval: 3600
;;retry_time: 3600
2017-03-12.automated-ksk-test.research.icann.org.	60	IN	DNSKEY	257 3 8
AwEAAa9qsSLDI+H0keqE3Yzdr6XuhqhBQVWw5xdgNoWLhE4VxSEIBz9IuCA4w4ssSrClZ59seNc76ltDFcKJv3X
9jDjzRtBLjenIgV4n/3GpKrAAnRlYbUtpBEdlk4mxoL3BlX8pfLg7RQfTlWaxOUga1+CChcVieFF/si/eePc9Hp
ZbWxHZRLCAE8dlDa0aa0tfVAZWOnaifpmbTvhDK3tdvMU0tfG2YfsOYcFB9z2KWmCDYwCONNKtls3p6wMwolun1
h8IYo0PF98vqjAp3NVRZvKKdgyF/bZ/iJtAZFytXvXU6Gwa5tOm1wgP6wuKupscP8KHBluZyOSKw4RMTk6YBdE=
;{id = 3934 (ksk), size = 2048b} ;;state=2 [  VALID  ] ;;count=0 ;;lastchange=1489997718 ;;Mon Mar 20 08:15:18 2017

2017-03-12.automated-ksk-test.research.icann.org.	60	IN	DNSKEY	257 3 8
AwEAAfUtjasCuLysD4MbjG3v4Kyu0vvVJ/0cIreP6fltMeZmwQ5SRta/mB+eFVjau+6YKra2UeTKxojBovHH2lZ
rw7NNejL44/Xps4gR3LSVMnCdwras+yvj4en64ghRGWYOuB+Icb0AqrCUhLFWR8yx41UkfaA2vzFnM2xTx0N0+o
6R6UciWuwJResomQupOjNUy2ZAi81Y3pb0x3Lw4POjpcSJzrK4aZ/5UPymplqhLEU2DsoQmyFlM5RNTt0YXR8XM
4Ywsu/scxg0u00IF1GC8xcyZUTMc1Rz98AY1VUo5QqUp9VbAed5Aw1nNYfjLTj+zOykedgmjms1iNgh9EY111c=
;{id = 19741 (ksk), size = 2048b} ;;state=1 [ ADDPEND ] ;;count=34 ;;lastchange=1489997718 ;;Mon Mar 20 08:15:18 2017
```

In line 15, we see the original key (3934) with a status of <mark>VALID</mark>, whereas in line 22 we see the newly spotted key 19741 is <mark>ADDPEND</mark>.

# What's next...?

Now we wait; 30 days, and as long as the key is observed throughout, the key should become trusted at the end of this...
