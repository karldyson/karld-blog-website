+++
title = "Alexa, why are you broken...?"
description = "My Amazon Echo Dot and Show devices stopped working properly shortly after the advert and malware blocking was introduced"
date = 2021-03-01
[extra]
toc = false
[taxonomies]
tags = ["alexa","dns","blocking"]
+++

For a bit of context, I have an Echo Show as a bedside alarm clock, and I also have an Echo Clock paired to an Echo Dot in the kitchen, primarily to visualise timers when I'm cooking.

After a power cut late on Friday evening, the Echo Show came back on but displaying the wrong time (more or less an hour behind, but not exactly). If you asked "Alexa, what time is it?", the correct time would be spoken, despite the wrong time still being displayed. Weird.

So, I got up, late, of course, and went to make breakfast. The Echo Clock in the kitchen is now also displaying the wrong time, but not the same wrong time that the Echo Show is displaying. Again, asking the paired Echo Dot for the time results in it speaking the correct time. Weirder and weirder.

I tried a bunch of things with the clock, reset and re-pair, checking the location and timezone settings for the devices in the app, but nothing resolved the incorrect time. I also noticed that the clock wasn't displaying timers properly, and the Echo Dot had stopped announcing the end of timers.

I'd checked whether the Echo devices were unable to contact some central cloud service at Amazon with a quick check of twitter and some down detectors, and they didn't seem to indicate widespread problems, and then it struck me: I wondered if my RPZ had picked up one or more entries that was causing this?

For some context at this point, I've been trialling the Energized Protection "Ultimate" in RPZ format. More about that soon in another post.

I got the MAC addresses for the Echo units from the Alexa app, grabbed the assigned IPs from the firewall and then looked in the RPZ logs to see if anything was being blocked for those client IPs -- **BINGO!**

The log entries all start at the time of the power outage, and the TTL on (at least) `fireoscaptiveportal.com` is `60`, and so I wonder if the Echo devices resolve the IP and then continue to use the resolved IP, ignoring the TTL?

There were four domains continually being blocked for the two devices:

* `fireoscaptiveportal.com`
* `mas-sdk.amazon.com`
* `prod.amazoncrl.com`
* `unagi-na.amazon.com`

I added whitelist entries to the RPZ for those, and immediatly could hear the Echo Dot in the kitchen announcing something ... it was a timer from a few days before. As I stopped one, it would tell me about another, until it seemed to get very confused, resulting in a power off/power on.

The RPZ entries I added were:

```
fireoscaptiveportal.com.override. 5 IN CNAME rpz-passthru.
*.fireoscaptiveportal.com.override. 5 IN CNAME rpz-passthru.
```

Both the Echo Show and Echo Clock immediately corrected their displayed time, and timers were both displayed correctly and announced at the end correctly.

I've subsequently added fixed IP leases in the firewall's DHCP config for the Echo devices, and added client-ip whitelisting for them in the RPZ.

```
32.105.0.1.10.rpz-client-ip.override. 5 IN CNAME rpz-passthru.
32.106.0.1.10.rpz-client-ip.override. 5 IN CNAME rpz-passthru.
```
