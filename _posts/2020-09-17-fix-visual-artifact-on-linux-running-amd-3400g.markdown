---
layout: post
title: Fix visual artifact on Linux running AMD 3400G
date: '2020-09-17 20:56:10'
tags:
- linux
- hardware
---

I did a mini-itx build with AMD 3400G. It works great except occasionaly visual artifacts which affected Firefox the most when watching videos. The problem is solved by disabling IOMMU. I verfied that setting kernel parameter [`iommu=pt` also works](https://bugs.launchpad.net/ubuntu/+source/xserver-xorg-video-amdgpu/+bug/1848741).

<!--kg-card-end: markdown--><figure class="kg-card kg-image-card kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2020/09/Screenshot_2020-09-17_13-47-18-1.png" class="kg-image" alt loading="lazy"><figcaption>https://en.wikipedia.org/wiki/Input%E2%80%93output_memory_management_unit</figcaption></figure><!--kg-card-begin: markdown-->

IOMMU is just like MMU which provides memory virtualization but for PCI devices instead of CPU. AMD 3400G has an integrated graphics card, which has to use system main memory. IOMMU provides extra protection but it also slows things down due to this extra step of virtual-physical memory translation. I do not use VMs on this machine, so I really don't need IOMMU. DMA (direct memory access) from the graphics card would be much faster.

There's a kernel parameter `amd_iommu` [https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=amd\_iommu](https://www.kernel.org/doc/html/latest/admin-guide/kernel-parameters.html?highlight=amd_iommu), which I just set it to `off` in `/etc/default/grub`. Visual artifacts are gone after `update-grub` and reboot.

<!--kg-card-end: markdown-->