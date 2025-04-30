---
title: Reverse Engineering A Windows Driver For Linux
description: Making the unsupported supported
slug: reversing-turbo
date: 2025-04-30
categories: dev
draft: true
---
I was recently watching PewDiePie's [video](https://youtu.be/pVI_smLgTY0?si=Yj5UFrUWDxJg7mFV) on his experience switching to Linux and something he said reminded me of one of the reasons why I enjoy using this operating system so much—the absolute freedom you have. If something doesn't work the way you want it to or a certain feature is unsupported you can dig into the code and fix it yourself!

As promised on my [blog-post]({{<ref "lfx-mentorship/#enabling-turbo-support-on-my-laptop">}}) about the  Linux kernel mentorship program, this article will be about my endeavours in trying to bring [turbo]({{<ref "lfx-mentorship/#enabling-turbo-support-on-my-laptop">}}) support for my laptop to Linux.

## The beginning
It all started when I was sitting zoned out in front of my laptop. My eyes glanced upon the keyboard and I was staring at a particular key when a question came to mind. They key in particular was a button which opens the NitroSense app used to change performance profiles of the laptop on Windows.

Now obviously, this key did not do anything on Linux since the app (as is the case with most proprietary software) only works on Windows. I still wondered if it was possible to change thermal profiles on Linux. To my surprise, someone had already made a [kernel module](https://github.com/JafarAkhondali/acer-predator-turbo-and-rgb-keyboard-linux-module) for this purpose! Unfortunately (or fortunately depending on how you look at it), it only supported the Acer Predator Series of laptops, while my laptop was from the Nitro Series.

Regardless, it proved as a good starting point and while going through the source code, I realized that this module was nothing more than a modified fork of a file from the platform profile subsystem from the kernel tree.
