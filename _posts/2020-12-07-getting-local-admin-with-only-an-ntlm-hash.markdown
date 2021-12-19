---
layout: post
title: Getting Local Admin with only an NTLM Hash
date: '2020-12-07 13:32:00'
---

Imagine if an unprivileged user (i.e. not a member of local administrators) found an NTLM hash of a user within the local administrators group. Could the unprivileged user obtain admin privileges? TLDR; Yes!

At first I thought maybe you could Pass-The-Hash to local services like WMI, SMB, etc using something like [https://github.com/Kevin-Robertson/Invoke-TheHash](https://github.com/Kevin-Robertson/Invoke-TheHash). I quickly discovered that those services do not allow user credentials to be specified for local connections. If anyone has a workaround to this, without requiring a second machine on the network, please let me know @aceb0nd.

The PTH technique _could_ work from a remote system to the compromised device, depending on LocalAccountTokenFilterPolicy and FilterAdministratorToken, but I considered that cheating.

I then realised, the Windows change password functionality only requires knowing the users NTLM hash. I used Mimikatz to update the password of the administrative account.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2020/12/change_password2.png" class="kg-image" alt loading="lazy" width="1348" height="1035" srcset="/images/size/w600/2020/12/change_password2.png 600w,/images/size/w1000/2020/12/change_password2.png 1000w,/images/2020/12/change_password2.png 1348w" sizes="(min-width: 1200px) 1200px"><figcaption>lsadump::changentlm to update the administrative account password</figcaption></figure>

I then used runas to execute PowerShell as the admin user.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2020/12/admin2.png" class="kg-image" alt loading="lazy" width="1238" height="861" srcset="/images/size/w600/2020/12/admin2.png 600w,/images/size/w1000/2020/12/admin2.png 1000w,/images/2020/12/admin2.png 1238w" sizes="(min-width: 1200px) 1200px"><figcaption>runas to execute a program with administrative privileges</figcaption></figure>

A UAC bypass would be required to further escalate privileges past the filtered admin token but I consider that out of scope for the question. In conclusion, if an unprivileged user finds an administrative NTLM hash, they can compromise the account fairly easily.

