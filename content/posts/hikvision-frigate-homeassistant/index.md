+++
title = "Hikvision, Frigate and Home Assistant"
description = "Getting the third stream working on the Hikvision cameras and then plumbing that into Frigate"
date = 2025-12-07
[extra]
toc = true
toc_inline = true
toc_ordered = true
styles = ["posts/demo.css"]
[taxonomies]
tags = ["hikvision","stream3","sub-stream","frigate","home-assistant"]
+++

# Introduction

A while back now, I started using Home Assistant and that quickly grew to me adding my first camera and using Frigate.

My first camera was a <mark>DS-2CD2387G2-LU</mark>.

At the time, I had thought my camera had a third stream, but I had only managed to persuade Frigate to work with the first, with patchy results with the second. The third didn't seem to work at all.

More recently, I had wanted to add another camera, and bought the current equivalent of my original (<mark>DS-2CD2387G3-LI2UY</mark>), to remain as consistent as possible.

This too apparently had a third stream, but again, I could not persuade it to work.

Whilst reading something else trying to get audio working (in Safari) for the live feed, I stumbled across a post telling me how to get the third stream working on the cameras.

So, to cut to the chase, you need to switch VCA mode to "Monitoring".

When you save, the camera will reboot and then your third stream will then magically appear.

I had wanted this so that I could have a lower but still reasonable resolution for detection whilst continuing to record the main 4K stream.

I hope this is useful to someone else!

# Details

Some of this may be useful too, so I'll chuck it in here for now...

## Configuration

If, like me, you've been having a challenge with any of this, then this is the config I'm currently running:

Each camera is essentially the same, so I'll just include the config for one.

This is all in the `/addon_configs/ccab4aaf_frigate-fa/config.yaml` file.

This first bit has been there since the original camera, and I'm not entirely sure it's still needed as audio has become a more mainstream feature. But I've not looked at that yet, so it's still there.

```yaml
ffmpeg:
  hwaccel_args: preset-rpi-64-h264
  output_args:
    record: -f segment -segment_time 10 -segment_format mp4 -reset_timestamps 1 
      -strftime 1 -c:v copy -c:a aac
```

Next, go2rtc config:

```yaml
go2rtc:
  streams:
    cam_1:
      - rtsp://user:password@192.0.2.101:554/Streaming/Channels/101
      - "ffmpeg:cam_1#audio=aac"
    cam_1_sub:
      - rtsp://user:password@192.0.2.101:554/Streaming/Channels/103
```

...and then finally, the camera:

```yaml
cameras:
  cam_1:
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/cam_1
          input_args: preset-rtsp-restream
          roles:
            - record
        - path: rtsp://127.0.0.1:8554/cam_1_sub
          input_args: preset-rtsp-restream
          roles:
            - detect
    detect:
      enabled: true
      width: 1920
      height: 1080
      fps: 5
    record:
      enabled: true
    live:
      streams:
        Main Stream: cam_1
        Sub Stream: cam_1_sub
```

## Camera Configuration Screens

### Older Camera

First, the older of the two, running firmware V5.7.3 build 220112

<figure>
{{ image(url="original_camera_1.png") }}
<figcaption>Older camera config screen showing where to flip RCA Resource to Monitoring</figcaption>
</figure>

<figure>
{{ image(url="original_camera_2.png") }}
<figcaption>Older camera config screen showing the third stream showing up post reboot</figcaption>
</figure>

### Newer Camera

Then the newer one, running firmware v5.8.10 build 250605 currently

<figure>
{{ image(url="newer_camera_1.png") }}
<figcaption>Newer camera config screen showing where to flip RCA Resource to Monitoring</figcaption>
</figure>

<figure>
{{ image(url="newer_camera_2.png") }}
<figcaption>Newer camera config screen showing the third stream showing up post reboot</figcaption>
</figure>

