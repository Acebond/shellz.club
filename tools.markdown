---
layout: page
title: Tools
permalink: '/tools/'
---

### [PPLKiller](https://github.com/RedCursorSecurityConsulting/PPLKiller)
PPLKiller leverages a trusted MSI driver to disable LSA Protection; allowing credentials to be dumped from memory. The tool supports removing the Protected Process Light (PPL) attributes from any process and manipulating process tokens.

### [NTFSCopy](https://github.com/RedCursorSecurityConsulting/NTFSCopy)
An execute-assembly compatible tool that can copy in-use files such as ntds.dit using NTFS structure parsing. The tool simply a wrapper for [NtfsLib](https://github.com/LordMike/NtfsLib).

### [LSASecretsTool](https://github.com/Acebond/LSASecretsTool)
An execute-assembly compatible tool for manipulating LSA secrets using the undocumented but official LSASS API calls. This includes reading, writing, creating and deleting LSA secrets.

### [CVE-2020-0668](https://github.com/RedCursorSecurityConsulting/CVE-2020-0668)
Implementation of CVE-2020-0668 which leverages symbolic links to perform a privileged file move operation that can lead to privilege escalation on all versions of Windows from Vista to 10, including server editions.

### [SharpHashSpray](https://github.com/RedCursorSecurityConsulting/SharpHashSpray)
An execute-assembly compatible tool for spraying local admin hashes (NTLM). By default the tool will automatically fetch a list of all domain joined hosts to check. Alternatively a target range can be provided.

### [GetAdDecodedPassword](https://github.com/RedCursorSecurityConsulting/GetAdDecodedPassword)
This tool queries Active Directory for users with the UnixUserPassword, UserPassword, unicodePwd, or msSFU30Password properties populated. It then decodes those password fields and displays them to the user.

<!--kg-card-begin: html--><!--
GoMSBuild (Private)
Generate tailored MSBuild compatible project files for executing shellcode on endpoints running application whitelisting solutions. The tool employs a number of antivirus and EDR evasion techniques, including encryption of the shellcode, sandbox detection, environment keying and multiple shellcode injection methods.
--><!--kg-card-end: html-->