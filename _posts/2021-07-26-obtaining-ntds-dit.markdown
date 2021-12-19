---
layout: post
title: Dumping Active Directory Credentials
date: '2021-07-26 05:52:05'
---

All Active Directory user account password hashes are stored inside the ntds.dit database file on the Domain Controllers. However, if you have ever tried copying the file, you'll probably have received the following error message.

<figure class="kg-card kg-image-card"><img src="/images/2021/07/meme_for_blog.jpg" class="kg-image" alt loading="lazy" width="500" height="764"></figure>

Well as it turns out, the LSASS process has already opened the file, and when it called [CreateFileW](https://docs.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew) to open ntds.dit, it set the dwShareMode parameter to the value 0, which "Prevents other processes from opening a file or device if they request delete, read, or write access". What can we do about this? Well there are 4 different techniques that can be used to bypass the exclusive file handle restrictions. These are:

1. Directory Replication Service (DRS) Remote Protocol
2. LSASS Manipulation
3. Volume Shadow Copy Service
4. Low-level Disk Reading

## 1. Directory Replication Service

The Directory Replication Service (DRS) Remote Protocol can be leveraged to remotely download the information (such as password hashes and Kerberos keys) stored within the ntds.dit database. The most common tools that implement this technique are:

- Mimikatz (lsadump::dcsync)
- Impacket (secretsdump.py)
- SharpKatz (--Command dcsync)

Synchronizing data using the DRS Remote Protocol requires replication permissions, which Domain Admins have by default. The technique must be performed within the context of a user that has the highlighted privileges below.

<figure class="kg-card kg-image-card kg-card-hascaption"><img src="/images/2021/07/image.png" class="kg-image" alt loading="lazy" width="598" height="681"><figcaption>Default Domain Admin permissions</figcaption></figure>

The Mimikatz `lsadump::dcsync` command can be used to dump all NTLM hashes, Kerberos keys, or specific information based on GUID.

<figure class="kg-card kg-image-card kg-card-hascaption"><img src="/images/2021/07/image-1.png" class="kg-image" alt loading="lazy" width="921" height="292" srcset="/images/size/w600/2021/07/image-1.png 600w,/images/2021/07/image-1.png 921w" sizes="(min-width: 720px) 720px"><figcaption>Dump all NTLM hashes</figcaption></figure><figure class="kg-card kg-image-card kg-card-hascaption"><img src="/images/2021/07/image-2.png" class="kg-image" alt loading="lazy" width="1104" height="894" srcset="/images/size/w600/2021/07/image-2.png 600w,/images/size/w1000/2021/07/image-2.png 1000w,/images/2021/07/image-2.png 1104w" sizes="(min-width: 720px) 720px"><figcaption>Dump KRBTGT credential information and Kerberos keys</figcaption></figure><figure class="kg-card kg-image-card kg-card-hascaption"><img src="/images/2021/07/image--1-.png" class="kg-image" alt loading="lazy" width="888" height="499" srcset="/images/size/w600/2021/07/image--1-.png 600w,/images/2021/07/image--1-.png 888w" sizes="(min-width: 720px) 720px"><figcaption>Dump Trust Keys</figcaption></figure>

The same information can also be retrieved using Impacket secretsdump.py or SharpKatz. I personally consider this technique the stealthiest, as no code needs to be executed on the Domain Controller, and I've never had it detected during internal engagements.

## LSASS Manipulation

Since the LSASS has an open handle to the file, an attacker can manipulate the LSASS process in a number of ways to obtain the contents of the ntds.dit file or information stored within the file. I'm going to ignore the destructive or distributive methods such as closing the handle, terminating LSASS or stopping the NTDS service (`Stop-Service -Name "NTDS" -Force`), but these do work. After stopping the service, you can simply copy the file without any issues.

Mimikatz can inject or patch the LSASS process and leverage it's functionality to dump the credential material stored within ntds.dit. The [mimikatz-deep-dive-on-lsadumplsa-patch-and-inject](https://blog.3or.de/mimikatz-deep-dive-on-lsadumplsa-patch-and-inject.html) blog post explains this really well. TLDR: The `/patch` method should be considered more OPSEC safe. The `/inject` method will create a new thread inside LSASS.

<figure class="kg-card kg-image-card kg-card-hascaption"><img src="/images/2021/07/image-3.png" class="kg-image" alt loading="lazy" width="700" height="660" srcset="/images/size/w600/2021/07/image-3.png 600w,/images/2021/07/image-3.png 700w"><figcaption>lsadump::lsa /patch (optionally /name:KRBTGT to print for a specific user)</figcaption></figure><figure class="kg-card kg-image-card kg-card-hascaption"><img src="/images/2021/07/image-4.png" class="kg-image" alt loading="lazy" width="752" height="747" srcset="/images/size/w600/2021/07/image-4.png 600w,/images/2021/07/image-4.png 752w" sizes="(min-width: 720px) 720px"><figcaption>lsadump::lsa /inject /name:KRBTGT (omitting the name will print the information for all users)</figcaption></figure>

Other tools such as Meterpreter hashdump use similar techniques to dump the credential material by injecting into LSASS. This approach is the most detectable as any anomalous manipulation of the LSASS process, which is heavily monitored by EDR, should be considered a critical alert.

## Volume Shadow Copy

The Volume Shadow Copy Service (VSS) is a built-in Windows mechanism that enables the creation of consistent, point-in-time copies of data, known as shadow copies or snapshots. The VSS allows the copying of in-use files, such as the ntds.dit database and SYSTEM hive. A number of built-in Windows tools exist that can be used to copy files using the VSS.

### ntdsutil

The ntdsutil tool (available from Windows 2008 and later) can be used to backup the ntds.dit database and SYSTEM hive (which contains the key required to extract the password hashes). This is actually one of the intended purposes of the tool, to create Active Directory Install from Media (IFM) files. The below command must be executed with administrative privileges on the Domain Controller.

    C:\>ntdsutil
    ntdsutil: activate instance ntds
    ntdsutil: ifm
    ifm: create full c:\windows\temp\snapshot
    ifm: quit
    ntdsutil: quit

This can be shortened into a single line command line so:

    ntdsutil "ac i ntds" "ifm" "create full c:\windows\temp\snapshot" q q

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/06/image.png" class="kg-image" alt loading="lazy" width="1242" height="802" srcset="/images/size/w600/2021/06/image.png 600w,/images/size/w1000/2021/06/image.png 1000w,/images/2021/06/image.png 1242w" sizes="(min-width: 1200px) 1200px"><figcaption>ntdsutil one-liner executed in Cobalt Strike</figcaption></figure>
### vssadmin, DiskShadow and esentutl

These built-in tools can also be used to copy files using the VSS. The location of the ntds.dit database file, which defaults to `C:\Windows\NTDS\ntds.dit`, can in the rare case of a non-default setting, be found by checking the `DSA Database file` value in the registry key `HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Parameters`. Examples of copying the files using vssadmin and esentutl are shown below:

<figure class="kg-card kg-code-card"><pre><code>esentutl.exe /y /vss c:\windows\ntds\ntds.dit /d c:\Windows\temp\ntds.dit
reg save hklm\system C:\windows\temp\system</code></pre>
<figcaption>Copying ntds.dit using esentutl </figcaption></figure><figure class="kg-card kg-code-card"><pre><code>vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\NTDS.dit C:\ShadowCopy
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\ShadowCopy
vssadmin delete shadows /shadow={GUID}</code></pre>
<figcaption>Copying ntds.dit using vssadmin</figcaption></figure>

I would suggest compressing (and maybe encrypting) the files before exfiltrating the data to save bandwidth. I use [MiddleOut](https://github.com/FortyNorthSecurity/MiddleOut). The credential material can then be dumped using something like [gosecretsdump](https://github.com/C-Sto/gosecretsdump) (must faster then Impacket secretsdump.py and bonus points because I'm a go fanboy).

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/07/image-6.png" class="kg-image" alt loading="lazy" width="1768" height="558" srcset="/images/size/w600/2021/07/image-6.png 600w,/images/size/w1000/2021/07/image-6.png 1000w,/images/size/w1600/2021/07/image-6.png 1600w,/images/2021/07/image-6.png 1768w" sizes="(min-width: 1200px) 1200px"><figcaption>Dumping credential material using gosecretsdump</figcaption></figure><figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/07/image-5.png" class="kg-image" alt loading="lazy" width="1439" height="661" srcset="/images/size/w600/2021/07/image-5.png 600w,/images/size/w1000/2021/07/image-5.png 1000w,/images/2021/07/image-5.png 1439w" sizes="(min-width: 1200px) 1200px"><figcaption>Dumping credential material using secretsdump.py</figcaption></figure>

The VSS method is fairly stealthy, can be done remotely using WMI or WinRM, and it's unlikely that the events are being monitored or alerted on.

## Low-level Disk Reading

This is probably the stealthiest local method in terms of detection. The tools [Invoke-NinjaCopy](https://raw.githubusercontent.com/BC-SECURITY/Empire/master/data/module_source/collection/Invoke-NinjaCopy.ps1) (based on [An-NTFS-Parser-Lib](https://www.codeproject.com/Articles/81456/An-NTFS-Parser-Lib)), [RawCopy](https://github.com/jschicht/RawCopy), and my own tool [NTFSCopy](https://github.com/RedCursorSecurityConsulting/NTFSCopy) (all the credit goes to [NtfsLib](https://github.com/LordMike/NtfsLib)) implement NTFS structure parsing to copy the contents of the file.

My personal preference is NTFSCopy as it is compatible with execute-assembly (in-memory execution) within C2 tools such as Cobalt Strike.

<figure class="kg-card kg-image-card kg-width-wide"><img src="/images/2021/07/image-7.png" class="kg-image" alt loading="lazy" width="1533" height="754" srcset="/images/size/w600/2021/07/image-7.png 600w,/images/size/w1000/2021/07/image-7.png 1000w,/images/2021/07/image-7.png 1533w" sizes="(min-width: 1200px) 1200px"></figure>

Hopefully this helps you dumping those juicy Active Directory credentials.

