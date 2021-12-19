---
layout: post
title: Interesting Web Vulnerabilities Part 2
date: '2021-03-28 13:14:32'
---

This blog post series serves as a reference to the "cool" and often less commonly discussed web hacking techniques or methods that exist. The list was originally created for my own reference, but I wanted to share it publicly for anyone interested. Part 1 can be found [here]( __GHOST_URL__ /interesting-web-attacks/).

### 1. Windows Defender as a Side-channel to Leak Data

Windows Defender (and potentially other antivirus engines) can be used as a side-channel to leak data in specific circumstances. Windows Defender will, as part of the scanning process, execute supported file types within a sandbox environment to check for malicious behavior/signatures. This can be leveraged as an oracle to read information within the file itself, using the fact that a detection can be triggered conditionally. This technique has been used in the CTF challenges phpnote at TokyoWesterns 2019 and Gyotaku The Flag at WCTF2019.

The premise is the ability to partially control the contents of a file stored on disk that also contains a secret. By injecting JScript code into the file, which will be executed by Windows Defender, you can perform a boolean based operation, such as checking the first character of the secret, and triggering a detection based on the outcome. The idea being:

eval("EICAR") =\> detected  
eval("EICA" + secret[0]) -\>  
 detected -\> the first character of secret is "R"  
 not detected -\> the first character of secret is not "R"

There are a few additional preconditions within the actual proof of concept, such as requirement to control data both before and after the secret. This makes real world exploitation extremely unlikely, nevertheless it's an interesting and unique attack to know exists for CTF challenges.

- [https://speakerdeck.com/icchy/lets-make-windows-defender-angry-antivirus-can-be-an-oracle](https://speakerdeck.com/icchy/lets-make-windows-defender-angry-antivirus-can-be-an-oracle)
- [https://westerns.tokyo/wctf2019-gtf/wctf2019-gtf-slides.pdf](https://westerns.tokyo/wctf2019-gtf/wctf2019-gtf-slides.pdf)

### 2. Leading-trailing Whitespace

Leading-trailing whitespace can be used to exploit inconsistencies within application logic to trick an application into performing unauthorized operations. The CTFd platform briefly contained a vulnerability (CVE-2020-7245) which allowed an account takeover using trailing whitespace, due to a mismatch during the password reset function. Essentially:

1. Register an account that already exists by appending whitespace to make the username unique;
2. Perform a password reset which will send a reset link to the email address used during registration;
3. Submit the password reset, which will reset the password of the victim user due to the fact that filter\_by in the SqlAlchemy ORM ignores trailing whitespace.

This type of bug can exist in any application that inconsistently handles data, but the most common issues are generally edge cases surrounding leading-trailing whitespace characters.

### 3. [GitHack](https://github.com/BugScanTeam/GitHack), [gitjacker](https://github.com/liamg/gitjacker)and &nbsp;[DS-Crawler](https://github.com/0x080/DS-Crawler)

GitHack and Gitjacker can download git repositories and extracts their contents from sites where the .git directory has been mistakenly uploaded. Similarly, DS-Crawler can crawl web servers for unintended files/folders by parsing Apple's .DS\_Store file format. These tools can be used to find interesting, potentially sensitive, unlisted files, folders and content.

### 4. EICAR DoS

This is a fairly simple test case that can result in very unexpected behaviour. Basically, make your username, or password, or something in the application, the EICAR test file (or something that looks malicious like the string "Invoke-Mimikatz -DumpCreds") and see what happens. Technically speaking, anti-virus should only detect EICAR at the beginning of a file, but we all know how good anti-virus are at following standards. There is also the possibility that an internal WAF detects and blocks HTTP requests containing the EICAR string.

The potential outcomes are limitless, but include:  
- getting the database deleted;  
- bricking some internal synchronization functionality;  
- self DoS;  
- nothing.

### 4. Static or Publicly Disclosed Application Secrets

I recently achieved remote code execution on an application that shared the PHP Symfony Framework secret between customers. I had to request a trial of the application, extract the secret and execute code as described in the [Secret Fragments: Remote Code Execution on Symfony Based Websites](https://www.ambionics.io/blog/symfony-secret-fragment) blog.

The attack itself relates to any useful application secret that has been leaked on online (such as GitHub), or is shared between different instances of the same software. The exact same scenario occurred with Microsoft Exchange when the ASP.NET Machine Keys were static between installations. This allowed the Machine Keys to be disclosed and used for Remote Code Execution. The vulnerability was assigned CVE-2020-0688, and [multiple](https://github.com/ravinacademy/CVE-2020-0688) [public](https://github.com/zcgonvh/CVE-2020-0688) [exploits](https://github.com/Ridter/cve-2020-0688) are available on GitHub. The tool [Blacklist3r](https://github.com/NotSoSecure/Blacklist3r) contains a database of Machine Keys that have appeared online, and potentially been used by developers in production systems.

The key takeaway is that when penetration testing an application, try and obtained a copy of the software, whether it be a trial, evaluation, maybe pirated, for the purpose of disclosing applications secrets.

