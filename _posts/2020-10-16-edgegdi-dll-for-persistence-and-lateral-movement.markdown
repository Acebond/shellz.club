---
layout: post
title: EdgeGdi.dll for Persistence and Lateral Movement
date: '2020-10-16 07:01:06'
---

I recently read [https://chadduffey.com/2020/10/10/edgegdi.html](https://www.chadduffey.com/2020/10/edgegdi.html) which describes how EdgeGdi.dll can be used for persistence, although with the caveat that “Windows wont tolerate the implementation as described here [linked blog post] over a reboot, it’ll blue screen”.

I decided to take on the challenge of solving the blue screen on reboot issue. I created the DLL as described, placed it into System32 and when I rebooted received the stop code 0xC000007B (see Figure 1). Googling the stop code provided no helpful information.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="https://lh3.googleusercontent.com/sbDaiv36rsziushSklPv0W78ufWV6V-IGgNEMLv7zRkkFE2WRiWf_9QYkRifD6B_LtJdWBZE4byyWt-XRvXj70MXasZ-RCbUrpZcNW8Ccili6Ao10m0Wlvii-YTtn4Miv-KGtVDk" class="kg-image" alt loading="lazy"><figcaption>Figure 1 - Stop code 0xC000007B</figcaption></figure>

I rebooted again, but this time with a Kernel debugger attached and received an error message detailing the issue (see Figure 2). The csrss.exe process was trying to load our persistence DLL which failed the device integrity policy.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="https://lh3.googleusercontent.com/c8dP-UUHFhI3ZcT-H00RECaDaESe6l9QLDHvlCHPvjowUywaa8Sgy5HtGh7jLQ9l97j2qQt91NekRXWKvu-7GvNNeqnAaoDuoK9qjpW-scLGFsUD5zZh_cxTBcUa8vOTI0B6ZbDi" class="kg-image" alt loading="lazy"><figcaption>Figure 2 - Code Integrity violation</figcaption></figure>

The csrss (Windows subsystem) process is marked signature restricted (Microsoft only) (see Figure 3) and is trying to load the unsigned EdgeGdi.dll which causes a code integrity violation. The process is also marked critical, which means that if it exits for any reasons, the system crashes.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="https://lh6.googleusercontent.com/X5HvBcfnwHfXrUbXPs5rs32MMQMpV7NknIvX7PGxWozOsTLs2V7RQVL-GXQdVm-7-OCg-GZqeWs-iX2BVkL0T4tCdvCYhI27ZLbWAXjit2AkhQqqUZtqZgvtQDP3RdhgA1JGgCf9" class="kg-image" alt loading="lazy"><figcaption>Figure 3 - CSRSS process mitigation policies</figcaption></figure>

The only solution is to prevent csrss from loading the unsigned DLL. I called my friend [@\_smashery\_](https://twitter.com/_smashery_) to help and our solution was to configure an Access Control Entry (ACE) to deny SYSTEM access to the EdgeGdi.dll (see Figure 4), which prevents csrss from loading the unsigned DLL, which prevents the system crash.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="https://lh4.googleusercontent.com/Kiwz2rSSxBsmT2q-ss6dK7viFl_VvNOBMEq54JffmdA-43757Ouc1a5Lq_sEykC0uvaQ2ATltgkcU55yQ3kwhwNKYD4jzj9bKSSpuKAZ2rJ6uVr8HHGlrSvx5i0f2VzyKMS_iptS" class="kg-image" alt loading="lazy"><figcaption>Figure 4 - Discretionary Access Control List to for EdgeGdi.dll</figcaption></figure>

The caveat now being that EdgeGdi.dll cannot be loaded by a SYSTEM process, but it WILL be loaded by processes running with NETWORK SERVICE privileges, which can be trivially escalated to SYSTEM privileges using one of the potato methods, see [Game Over Privileges]( __GHOST_URL__ /game-over-privileges/) for more details.

Now EdgeGdi.dll can be used for stealthy, stable persistence, or stealthy lateral movement by dropping it into System32 on another machine as described [here](https://www.mdsec.co.uk/2020/10/i-live-to-move-it-windows-lateral-movement-part-3-dll-hijacking/).

