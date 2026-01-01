+++
title = "Key Validation"
date = 2017-04-24
[extra]
toc = true
toc_inline = true
toc_ordered = true
styles = ["posts/demo.css"]
disclaimer = "This keyroll is long in the past now, but understanding the process is still useful"
[taxonomies]
tags = ["dns","rfc5011","keyroll","ksk","dnssec","validation"]
+++

# Introduction

Following on from my post about [the new key being added to the zone](@/posts/a-new-key.md), the required 30 days have passed and if your resolver is RFC5011 compliant, it should now trust the key.

# Checking...

You can check this as follows:

## BIND

```hl_lines=46
$ cat /var/named/managed-keys.bind
$ORIGIN .
$TTL 0  ; 0 seconds
@                       IN SOA  . . (
                                1904       ; serial
                                0          ; refresh (0 seconds)
                                0          ; retry (0 seconds)
                                0          ; expire (0 seconds)
                                0          ; minimum (0 seconds)
                                )
                        KEYDATA 20170425142612 20170210095625 19700101000000 257 3 8 (
                                AwEAAagAIKlVZrpC6Ia7gEzahOR+9W29euxhJhVVLOyQ
                                bSEW0O8gcCjFFVQUTf6v58fLjwBd0YI0EzrAcQqBGCzh
                                /RStIoO8g0NfnfL2MTJRkxoXbfDaUeVPQuYEhg37NZWA
                                JQ9VnMVDxP/VHL496M/QZxkjf5/Efucp2gaDX6RS6CXp
                                oY68LsvPVjR0ZSwzz1apAzvN9dlzEheX7ICJBBtuA6G3
                                LQpzW5hOA2hzCTMjJPJ8LbqF6dsV6DoBQzgul0sGIcGO
                                Yl7OyQdXfZ57relSQageu+ipAdTTJ25AsRTAoub8ONGc
                                LmqrAmRLKBP1dfwhYB4N7knNnulqQxA+Uk1ihz0=
                                ) ; KSK; alg = RSASHA256; key id = 19036
                                ; next refresh: Tue, 25 Apr 2017 14:26:12 GMT
                                ; trusted since: Fri, 10 Feb 2017 09:56:25 GMT
2017-03-12.automated-ksk-test.research.icann.org KEYDATA 20170424152612 20170317172529 19700101000000 257 3 8 (
                                AwEAAa9qsSLDI+H0keqE3Yzdr6XuhqhBQVWw5xdgNoWL
                                hE4VxSEIBz9IuCA4w4ssSrClZ59seNc76ltDFcKJv3X9
                                jDjzRtBLjenIgV4n/3GpKrAAnRlYbUtpBEdlk4mxoL3B
                                lX8pfLg7RQfTlWaxOUga1+CChcVieFF/si/eePc9HpZb
                                WxHZRLCAE8dlDa0aa0tfVAZWOnaifpmbTvhDK3tdvMU0
                                tfG2YfsOYcFB9z2KWmCDYwCONNKtls3p6wMwolun1h8I
                                Yo0PF98vqjAp3NVRZvKKdgyF/bZ/iJtAZFytXvXU6Gwa
                                5tOm1wgP6wuKupscP8KHBluZyOSKw4RMTk6YBdE=
                                ) ; KSK; alg = RSASHA256; key id = 3934
                                ; next refresh: Mon, 24 Apr 2017 15:26:12 GMT
                                ; trusted since: Fri, 17 Mar 2017 17:25:29 GMT
                        KEYDATA 20170424152612 20170418002534 19700101000000 257 3 8 (
                                AwEAAfUtjasCuLysD4MbjG3v4Kyu0vvVJ/0cIreP6flt
                                MeZmwQ5SRta/mB+eFVjau+6YKra2UeTKxojBovHH2lZr
                                w7NNejL44/Xps4gR3LSVMnCdwras+yvj4en64ghRGWYO
                                uB+Icb0AqrCUhLFWR8yx41UkfaA2vzFnM2xTx0N0+o6R
                                6UciWuwJResomQupOjNUy2ZAi81Y3pb0x3Lw4POjpcSJ
                                zrK4aZ/5UPymplqhLEU2DsoQmyFlM5RNTt0YXR8XM4Yw
                                su/scxg0u00IF1GC8xcyZUTMc1Rz98AY1VUo5QqUp9Vb
                                Aed5Aw1nNYfjLTj+zOykedgmjms1iNgh9EY111c=
                                ) ; KSK; alg = RSASHA256; key id = 19741
                                ; next refresh: Mon, 24 Apr 2017 15:26:12 GMT
                                ; trusted since: Tue, 18 Apr 2017 00:25:34 GMT
```

We can see in the output above that the new key, keytag 19741, is now trusted.

## Unbound

```hl_lines=11
$ cat /var/lib/unbound/2017-03-12.automated-ksk-test.research.icann.org.ds
; autotrust trust anchor file
;;id: 2017-03-12.automated-ksk-test.research.icann.org. 1
;;last_queried: 1493044058 ;;Mon Apr 24 14:27:38 2017
;;last_success: 1493044058 ;;Mon Apr 24 14:27:38 2017
;;next_probe_time: 1493047519 ;;Mon Apr 24 15:25:19 2017
;;query_failed: 0
;;query_interval: 3600
;;retry_time: 3600
2017-03-12.automated-ksk-test.research.icann.org.       60      IN      DNSKEY  257 3 8 AwEAAa9qsSLDI+H0keqE3Yzdr6XuhqhBQVWw5xdgNoWLhE4VxSEIBz9IuCA4w4ssSrClZ59seNc76ltDFcKJv3X9jDjzRtBLjenIgV4n/3GpKrAAnRlYbUtpBEdlk4mxoL3BlX8pfLg7RQfTlWaxOUga1+CChcVieFF/si/eePc9HpZbWxHZRLCAE8dlDa0aa0tfVAZWOnaifpmbTvhDK3tdvMU0tfG2YfsOYcFB9z2KWmCDYwCONNKtls3p6wMwolun1h8IYo0PF98vqjAp3NVRZvKKdgyF/bZ/iJtAZFytXvXU6Gwa5tOm1wgP6wuKupscP8KHBluZyOSKw4RMTk6YBdE= ;{id = 3934 (ksk), size = 2048b} ;;state=2 [  VALID  ] ;;count=0 ;;lastchange=1489997718 ;;Mon Mar 20 08:15:18 2017
2017-03-12.automated-ksk-test.research.icann.org.       60      IN      DNSKEY  257 3 8 AwEAAfUtjasCuLysD4MbjG3v4Kyu0vvVJ/0cIreP6fltMeZmwQ5SRta/mB+eFVjau+6YKra2UeTKxojBovHH2lZrw7NNejL44/Xps4gR3LSVMnCdwras+yvj4en64ghRGWYOuB+Icb0AqrCUhLFWR8yx41UkfaA2vzFnM2xTx0N0+o6R6UciWuwJResomQupOjNUy2ZAi81Y3pb0x3Lw4POjpcSJzrK4aZ/5UPymplqhLEU2DsoQmyFlM5RNTt0YXR8XM4Ywsu/scxg0u00IF1GC8xcyZUTMc1Rz98AY1VUo5QqUp9VbAed5Aw1nNYfjLTj+zOykedgmjms1iNgh9EY111c= ;{id = 19741 (ksk), size = 2048b} ;;state=2 [  VALID  ] ;;count=0 ;;lastchange=1492590342 ;;Wed Apr 19 08:25:42 2017
```

Similarly, for Unbound, we can see above that the status is now <mark>VALID</mark>.
