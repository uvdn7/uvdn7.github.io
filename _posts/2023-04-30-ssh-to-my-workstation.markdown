---
layout: post
title: "`ssh` to my workstation over Internet"
date: '2023-04-30 03:25:24'
tags:
- network
---

I have eero at home with eero plus subscription as I like the parental controls. To my delight, the subscription also comes with [DDNS](https://support.eero.com/hc/en-us/articles/360049818252-What-is-DDNS-). I wanted to set it up so I can ssh to my workstation from a laptop from our town library or a coffee shop.

1. Enabled IP reservation and port forwarding. Eero has an [article](https://support.eero.com/hc/en-us/articles/207908443-How-do-I-configure-port-forwarding-) about how to do it. I picked port `8080` and kept the internal port to be `22`, so that `ssh lupan@localhost` and `ssh lupan@xxx.eero.online -p 8080` would both work (instead of typing -p 8080 always). Eero makes it very easy to configure in the app.
2. `systemctl start sshd`
3. `ssh lupan@localhost` and `ssh lupan@xxx.eero.online -p 8080` both worked as expected 
