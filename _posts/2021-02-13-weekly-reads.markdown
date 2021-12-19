---
layout: post
title: Weekly Reads
date: '2021-02-13 01:08:57'
tags:
- news
---

I read a lot of blog posts related to the InfoSec industry because I love learning. I have decided to keep a record of the best research and tools I discover each week.

## 19 - 26 April

- [https://s3cur3th1ssh1t.github.io/Named-Pipe-PTH/](https://s3cur3th1ssh1t.github.io/Named-Pipe-PTH/)

## 12 - 18 April

- [https://shenaniganslabs.io/2021/04/13/Airstrike.html](https://shenaniganslabs.io/2021/04/13/Airstrike.html) - Good research, good impact, highly recommend giving it a read.
- [https://micahvandeusen.com/the-power-of-seimpersonation/](https://micahvandeusen.com/the-power-of-seimpersonation/) - Good write-up on the potato class of privilege escalation attacks. 

### 15 - 21 March 2021

- [https://posts.specterops.io/abstracting-scheduled-tasks-3b6451f6a1c5](https://posts.specterops.io/abstracting-scheduled-tasks-3b6451f6a1c5)

### 8 - 14 March 2021

- We have the unauthenticated Exchange RCE tracked as CVE-2021-27065 with multiple public PoCs [here](https://github.com/hausec/ProxyLogon), [here](https://github.com/ZephrFish/Exch-CVE-2021-26855/blob/main/ExchangeSheller.py) and [here](https://github.com/rapid7/metasploit-framework/blob/e5c76bfe13acddc4220d7735fdc3434d9c64736e/modules/exploits/windows/http/exchange_proxylogon_rce.rb). 
- [https://www.graplsecurity.com/post/anatomy-of-an-exploit-rce-with-cve-2020-1350-sigred](https://www.graplsecurity.com/post/anatomy-of-an-exploit-rce-with-cve-2020-1350-sigred) - SigRed getting the love it deserves with an awesome write-up.
- [https://windows-internals.com/exploiting-a-simple-vulnerability-part-2-what-if-we-made-exploitation-harder/](https://windows-internals.com/exploiting-a-simple-vulnerability-part-2-what-if-we-made-exploitation-harder/) - Excellent Windows exploit development research.
- [https://research.checkpoint.com/2021/playing-in-the-windows-sandbox/](https://research.checkpoint.com/2021/playing-in-the-windows-sandbox/) - Deep dive into Windows Sandbox internals. 

### 1 - 7 March 2021

- [https://sensepost.com/blog/2020/routopsy-hacking-routing-with-routers/](https://sensepost.com/blog/2020/routopsy-hacking-routing-with-routers/) - Attacking Layer 3 network protocols in internal networks. 
- [https://github.com/JamesCooteUK/SharpSphere](https://github.com/JamesCooteUK/SharpSphere) - Dump and download VM's memory, then manually extract credentials from LSASS. This tool is extra useful with the below CVE.
- Unauthenticated VMware vCenter RCE tracked as CVE-2021-21972 was disclosed with multiple public PoCs.
- [https://s3cur3th1ssh1t.github.io/Bypass\_AMSI\_by\_manual\_modification/](https://s3cur3th1ssh1t.github.io/Bypass_AMSI_by_manual_modification/) - Some good research of bypassing AMSI.
- [https://github.com/cribdragg3r/Alaris](https://github.com/cribdragg3r/Alaris) - Seems like a promising shellcode loader for bypassing antivirus. 
- [https://blog.orange.tw/2021/02/a-journey-combining-web-and-binary-exploitation.html](https://blog.orange.tw/2021/02/a-journey-combining-web-and-binary-exploitation.html) - I found this to be an interesting exploitation chain leveraging a user-after-free vuln in PHP.
- [https://thezerohack.com/how-i-might-have-hacked-any-microsoft-account](https://thezerohack.com/how-i-might-have-hacked-any-microsoft-account) - Microsoft account takeover with a tricky brute force attack. 

### 22 - 28 February 2021

- [https://itm4n.github.io/windows-registry-rpceptmapper-exploit/](https://itm4n.github.io/windows-registry-rpceptmapper-exploit/) - A detailed write-up of the previously mentioned Perfusion tool, which includes a neat trick for getting an interactive SYSTEM shell.
- [https://blog.joeminicucci.com/2021/who-let-the-arps-out-from-arp-spoof-to-domain-compromise](https://blog.joeminicucci.com/2021/who-let-the-arps-out-from-arp-spoof-to-domain-compromise) - I always find ARP spoofing a viable method on internal networks, and this blog demonstrates how it lead to Domain Admin.
- [https://www.mdsec.co.uk/2021/02/farming-for-red-teams-harvesting-netntlm/](https://www.mdsec.co.uk/2021/02/farming-for-red-teams-harvesting-netntlm/) - This is an amazing toolkit for poisoning file shares with hash leaking canaries. I look forward to testing it on future Red Teams / Internals.

### 15 - 21 February 2021

- [https://www.ambionics.io/blog/symfony-secret-fragment](https://www.ambionics.io/blog/symfony-secret-fragment) - This got me RCE on a webapp test. Absolutely amazing work, and high-caliber write-up.
- [https://alephsecurity.com/2021/02/16/apport-lpe/](https://alephsecurity.com/2021/02/16/apport-lpe/) - Cool method to get root on all versions of Ubuntu dating back to 12.04. PoC linked at the end. 

### 8 - 14 February 2021

- [Dependency Confusion: How I Hacked Into Apple, Microsoft and Dozens of Other Companies](https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610)
- [Bypassing LastPass’s “Advanced” YubiKey MFA: A MITM Phishing Attack](https://pberba.github.io/security/2020/05/28/lastpass-phishing/)
- [System Threads and their elusiveness. ‘Practical Reverse Engineering’ solutions - Part 2](https://www.matteomalvica.com/blog/2021/02/11/practical-re-win-solutions-ch3-system-threads/)
- [Relay Attacks via Cobalt Strike Beacons](https://pkb1s.github.io/Relay-attacks-via-Cobalt-Strike-beacons/)
- [Relaying 101](https://luemmelsec.github.io/Relaying-101/)
- [Perfusion](https://github.com/itm4n/Perfusion) - Windows 7, Windows Server 2008R2, Windows 8, and Windows Server 2012 privilege escalation. PoC included. 
