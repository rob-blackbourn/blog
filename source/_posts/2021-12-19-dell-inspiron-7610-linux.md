---
layout: post
title:  "Setting up a Dell Inspiron 16 Plus (7610) for Linux"
date:   2021-12-19 07:50:00 +0000
categories:
- Admin
tags:
- dell
- inspiron
- 7610
- linux
comments: true
---

## Introduction

I recently bought a Dell Inspiron 16 Plus (7610) for use with Linux.
There have been a number of issues that have been solved by smart people on the internet.
This is a short list of things that I have done to make things work.

I used the Ubuntu 21.10 distribution.

## Status

The trackpad is sometimes still weird, but it doesn't get in the way too much.

## Hardware

* CPU: i7-11800H
* Memory: 16G
* Graphics: Intel UHD Graphics
* Disk: 512GB

Note: I specifically chose the Intel UHD graphics to avoid issues with proprietary NVIDIA drivers.

## This issues

I've installed Ubuntu 21.10.

* Secure boot
* Flickering screen
* Battery drain while lid closed
* Short battery life when unplugged
* Trackpad weirdness

### Secure boot

This is the first laptop I've had with secure boot.
When installing Ubuntu 21.10 from a USB stick you get a prompt about "secure boot".
I went off and googled this, and by the time I got back it had timed out and made a decision for me.
I installed a few times, but ended up just switching secure boot off in the BIOS settings.
This hasn't caused me any problems.

### Flickering Screen

This is due to a kernel driver default setting.
You can find the answer and links to explanations [here](https://askubuntu.com/questions/1361640/screen-flickering-on-inspiron-16-plus-7610-after-installing-20-04-lts).

I added the args `i915.enable_psr=0` to the grub command line in `/etc/default/grub`.

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash i915.enable_psr=0"
```

Then

```bash
$ sudo update-grub
$ reboot
```

### Battery drain while lid closed

I noticed the battery loosing power when the lid was closed and the power unplugged.
This seems to have something to do with the sleep mode the laptop enters.
I found an answer [here](https://www.reddit.com/r/Dell/comments/8b6eci/xp_13_9370_battery_drain_while_suspended/).

I added the args `mem_sleep_default=deep` to the grub command line in `/etc/default/grub`.

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash i915.enable_psr=0 mem_sleep_default=deep"
```

Then

```bash
$ sudo update-grub
$ reboot
```

# Short battery life when unplugged

This is a problem with all Linux laptops, and it is not specific to the Dell. It seems the way manufacturers get a long battery life when unplugged
is by turning things down (like cpu frequency) or turning things off (like turbo boost). These get turned back up when
the system notices the load requirements have increased.

To solve this you need some power management software.
There are a number of packages out there. I went for
[auto-cpufreq](https://github.com/AdnanHodzic/auto-cpufreq).

```bash
$ sudo snap install auto-cpufreq
```

I also decided to add the boot flag `intel_pstate=disable` to `/etc/default/grub`
as recommended.

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash i915.enable_psr=0 mem_sleep_default=deep intel_pstate=disable"
```

Then

```bash
$ sudo update-grub
$ reboot
```

The [thermald](https://wiki.debian.org/thermald) package is also
recommended, but that was already installed.

### Trackpad weirdness

Sometimes the trackpad is weird.
The cursor movement becomes unresponsive, and clicks don't work.
It seems to be more of a problem when the laptop is plugged in, but this may be anecdotal.

There are two views on this.

#### Power Management

The first is that it is an issue with power management on the trackpad.
You can find a discussion of that [here](https://askubuntu.com/questions/1231126/touchpad-power-management-issue-sticky-cursor-input-detection-delay-autosus?newreg=19ccd4cd1c1e46f38af9124cc16e4c54).

To address this I created a file `/lib/systemd/system/disable-trackpad-pm.service` with the contents:

```
[Unit]
Description=Disables trackpad power management to work around input delays

[Service]
Type=oneshot
ExecStart=/bin/sh -c "echo on > /sys/bus/i2c/devices/i2c-0/device/power/control"

[Install]
WantedBy=multi-user.target
```

Then:

```bash
$ sudo systemctl daemon-reload
$ sudo systemctl enable disable-trackpad-pm
$ reboot
```

This has helped, but I still get the weirdness occasionally, especially when the laptop is plugged in.

#### Mechanical

The second view is that there is a mechanical problem.
The thought is that the battery (which is positioned under the trackpad)
interferes with the trackpad.
One solution proposed is to add some material (often plastic) between the battery and the trackpad.
You can find the instructions [here](https://www.dell.com/community/Inspiron/Running-List-of-Inspiron-7610-Touchpad-Issue-Posts/m-p/8095040/highlight/true#M132906).

I haven't tried this yet, but I'm thinking about it, unless another driver issue gets found in the next month or so.

## Thoughts

I really like the laptop (apart from the numeric keypad; why?),
and it's been frustrating battling through these issues.

I'm hoping someone gets to the bottom of the trackpad issue, as
this is an excellent machine and great value for money.
