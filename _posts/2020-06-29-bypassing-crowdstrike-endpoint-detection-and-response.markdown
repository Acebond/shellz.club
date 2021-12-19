---
layout: post
title: Bypassing CrowdStrike Endpoint Detection and Response
date: '2020-06-29 10:36:45'
---

In a recent engagement I had to compromise a hardened desktop running CrowdStrike and Symantec Endpoint Protection. The initial code execution method was my reliable favorite MSBuild (C:\Windows\Microsoft.NET\Framework64\v4.0.30319\MSBuild.exe) which could be leveraged to execute C# code as an inline task.

Initially I wrote a very basic loader that used a bruteforce decryption algorithm to run a Cobalt Strike beacon using VirtualAlloc and CreateThread.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2020/06/2020-06-26_15-03-57.png" class="kg-image" alt loading="lazy"><figcaption>VirtualAlloc/CreateThread loader</figcaption></figure>

The shellcode was stored encrypted within the C# code and decrypted using a multibyte XOR key with the last 3 bytes removed. The while loop continuously increments the decryption key, &nbsp;performs XOR decryption and hashes the decrypted payload until the hash value of the decrypted shellcode matches its original hash value. This methods prevents antivirus or snooping eyes from easily reading the shellcode, and in fact no antivirus product would spend the amount of time required to decrypt the shellcode even if they knew how to run C# code in MSBuild project files.

CrowdStrike detected and prevented the VirtualAlloc/CreateThread method and classified it as “Process Hollowing”.

I'd never seen that technique detected before, but had been waiting for the day. I knew I’d have to take CrowdStrike seriously and wrote a new loader that used the same decryption routine but now executed code using the undocumented NtMapViewOfSection and NtQueueApcThread functions.

The **NtMapViewOfSection** routine maps a view of a section object into the virtual address space of a subject process.

The **NtQueueApcThread** routine adds a user-mode asynchronous procedure call (APC) object to the APC queue of the specified thread.

These methods are far less common, and considered more stealthy. The majority of the loader code was taken from the Sharp-Suite UrbanBishop POC ([https://github.com/FuzzySecurity/Sharp-Suite](https://github.com/FuzzySecurity/Sharp-Suite)) which implemented exactly what I wanted. I updated the POC slightly to support inline shellcode and automatic process selection for the injection.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2020/06/2020-06-26_16-06-09.png" class="kg-image" alt loading="lazy"><figcaption>NtMapViewOfSection/NtQueueApcThread loader with debug statements and error checking removed to fit into a screenshot</figcaption></figure>

The loader now successfully bypassed the CrowdStrike prevention rules. The use of MSBuild did trigger a detection alert in this particular configuration that was unfortunately unavoidable unless a different initial code execution method was used. The detection alert was not a prevention which meant the shell was allowed to live.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2020/06/nasty-loader-1.png" class="kg-image" alt loading="lazy"><figcaption>Successful execution with debug statements</figcaption></figure>

The loader does create a suspended thread in the remote process to queue the APC. This can be avoided by selecting an already existing thread, however the thread needs to enter/be in an alertable state for the APC to execute and picking the wrong thread can impact the stability of the remote process. The trade-off for that extra bit of stealthiness did not seem worthwhile but I figured it worth mentioning for anyone writing detections.

_Note_: This is a cross-post of a blog entry I wrote for Red Cursor. The original can be found here: [https://www.redcursor.com.au/blog/bypassing-crowdstrike-endpoint-detection-and-response](https://www.redcursor.com.au/blog/bypassing-crowdstrike-endpoint-detection-and-response)

