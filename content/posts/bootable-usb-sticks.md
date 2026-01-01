+++
title = "Bootable USB Sticks"
date = 2018-01-22
[extra]
[taxonomies]
tags = ["usb-stick","flash-drive","bootable"]
+++

I'm finally getting around to building a new ESX host to play with, and of course, nothing has an optical drive in it any more, so I need to make a bootable USB stick.

Maybe I've not done this enough, maybe you're rolling your eyes at me now, saying "really, you should know this".

I made the stick using the VMware instructions, but found that ESX didn't boot. First off problems with it not finding some files (`menu.c32`, `mboot.c32`), and once I'd persuaded it to find them, `loading -c...failed`

Much googling ensued, before finding the gem that solved the problem. I was making the USB stick on a recent Ubuntu install, which was using syslinux v6.something. The useful gem was that anything after some early version of syslinux 4 was not going to work. I got syslinux 3.86 from [kernel.org](https://www.kernel.org/pub/linux/utils/boot/syslinux/3.xx/). That's 32 bit, and so poked Ubuntu with a

```
sudo dpkg --add-architecture i386
```

and then

```
sudo apt-get install libc6:i386 libncurses5:i386 libstdc++6:i386
```

I then re-created the stick from scratch and hey presto, all worked well, so I'm putting this here, partly to remind me, and partly so google will index it and hopefully help you if you're stuck!
