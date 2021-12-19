---
layout: post
title: 'The Power of Kerberos Part 1: Resource-Based Constrained Delegation'
date: '2021-04-06 13:42:00'
---

Imagine the below setup, with Bravo being a low privileged user, and HEADHUNTER being configured with Unconstrained Delegation.

<figure class="kg-card kg-image-card kg-width-wide"><img src=" __GHOST_URL__ /content/images/2021/03/image.png" class="kg-image" alt loading="lazy" width="1374" height="622" srcset=" __GHOST_URL__ /content/images/size/w600/2021/03/image.png 600w, __GHOST_URL__ /content/images/size/w1000/2021/03/image.png 1000w, __GHOST_URL__ /content/images/2021/03/image.png 1374w" sizes="(min-width: 1200px) 1200px"></figure>

This can be exploited to achieve Domain Admin privileges by performing a Resource-Based Constrained Delegation (RBCD) attack to gain control of HEADHUNTER. Then using Unconstrained Delegation on HEADHUNTER to gain control of the Domain Controller via the printer bug.

## Preconditions for RBCD

1. We have code execution within the context of CHAOS\Bravo
2. CHAOS\Bravo has any of GenericAll, GenericWrite, WriteDacl, WriteProperty, etc on HEADHUNTER to configure the **msDS-AllowedToActOnBehalfOfOtherIdentity** attribute
3. The domain has at least one Domain Controller running at least Windows Server 2012
4. We have control of an account with an SPN configured or the ability to create one. Note that by default all members of Domain Users can create new machines accounts which meets this requirement. This account will be referred to as ATTACKER1.

## Performing the Attack

Normally you would observe #2 in BloodHound or PowerView and begin planning the attack. Unless you have control of an account with an SPN, from say obtaining administrative privileges on a computer within the domain or Kerberoasting, you will have to create a new machine account using [Powermad](https://github.com/Kevin-Robertson/Powermad). The machine account will by default have several SPNs registered.

    New-MachineAccount -MachineAccount ATTACKER1 -Password $(ConvertTo-SecureString 'CovidSucks2021' -AsPlainText -Force) -Verbose

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2021/03/image-1.png" class="kg-image" alt loading="lazy" width="1657" height="383" srcset=" __GHOST_URL__ /content/images/size/w600/2021/03/image-1.png 600w, __GHOST_URL__ /content/images/size/w1000/2021/03/image-1.png 1000w, __GHOST_URL__ /content/images/size/w1600/2021/03/image-1.png 1600w, __GHOST_URL__ /content/images/2021/03/image-1.png 1657w" sizes="(min-width: 1200px) 1200px"><figcaption>Add a new machine account directly through an LDAP add request to a domain controller</figcaption></figure>

Next configure RBCD on the target computer, delegating impersonation rights to ATTACKER1. This can be performed using an updated version of [PowerView](https://github.com/ZeroDayLab/PowerSploit/tree/master/Recon). While we can delegate access to any account, the Rubeus s4u action in the next step requires that the account setup for RBCD contain at least one SPN, which is why ATTACKER1 has been chosen.

    Set-DomainRBCD HEADHUNTER -DelegateFrom ATTACKER1 -Verbose

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2021/03/image-2.png" class="kg-image" alt loading="lazy" width="1570" height="347" srcset=" __GHOST_URL__ /content/images/size/w600/2021/03/image-2.png 600w, __GHOST_URL__ /content/images/size/w1000/2021/03/image-2.png 1000w, __GHOST_URL__ /content/images/2021/03/image-2.png 1570w" sizes="(min-width: 1200px) 1200px"><figcaption>Configure RBCD on HEADHUNTER to allow ATTACKER1 delegation rights</figcaption></figure>

To perform the Rubeus s4u action, we must provide a TGT (using **/ticket** ) or hash (using **/rc4** or **/aes256** ) of the account allowed to perform RBCD, which is ATTACKER1 in our instance. The `aes256` key is considered the best for OPSEC. We can calculate the Kerberos encryption keys for ATTACKER1 with the command:

    Rubeus.exe hash /user:ATTACKER1$ /password:CovidSucks2021 /domain:CHAOS.LOCAL

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2021/03/image-7.png" class="kg-image" alt loading="lazy" width="1483" height="509" srcset=" __GHOST_URL__ /content/images/size/w600/2021/03/image-7.png 600w, __GHOST_URL__ /content/images/size/w1000/2021/03/image-7.png 1000w, __GHOST_URL__ /content/images/2021/03/image-7.png 1483w" sizes="(min-width: 1200px) 1200px"><figcaption>Calculate the Kerberos encryption keys using the username/password</figcaption></figure>

In the instance you are using an already compromised computer account, you will need the Kerberos encryption keys, which are derived from the machine account password. These can be obtained using the Mimikatz command `sekurlsa::ekeys`, dumped remotely using [SharpSecDump](https://github.com/G0ldenGunSec/SharpSecDump), or calculated using the above Rubeus command and the machine account password.

Now use [Rubeus](https://github.com/GhostPack/Rubeus) to perform S4U2Self and S4U2Proxy to impersonate a user to the target computer. These Kerberos protocol extensions are outside the scope of this blog post but references have been included that explain the technical details. The Kerberos ticket requests require an SPN that exists on HEADHUNTER be specified (using **/msdsspn** ), and TERMSRV/HEADHUNTER.CHAOS.local, found using BloodHound, has been chosen. Note that the HOST SPN, which often and does exist in my case, is a catch all for several SPNs which are determined by the **SPNmappings** attribute. You can therefore choose say DCOM/HEADHUNTER.CHAOS.local, which does not explicitly exist on HEADHUNTER, but will be handled by HOST.

The user being impersonated should have administrative privileges to the target machine - in the example below, the domain admin CHAOS\Alpha has been chosen. Note that you cannot choose a user which has the attribute **Cannot Be Delegated** set to **True** or members of the **Protected Users** group. These conditions can be checked using BloodHound.

If Rubeus does not choose a Domain Controller running at least **Windows Server 2012** you will have to manually specify the Domain Controller with the **/dc** parameter. Alternative services can be requested with the **/altservice** parameter. I normally request HOST and RPCSS for WMI, and CIFS for SMB access.

    Rubeus.exe s4u /user:ATTACKER1$ /aes256:60BD5F4939AB0F848C5DD3B5650FB772C4718CA58223389CE361940D56508CDD /impersonateuser:ALPHA /msdsspn:TERMSRV/HEADHUNTER.CHAOS.local /domain:CHAOS.LOCAL /altservice:host,cifs,RPCSS /nowrap /ptt

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2021/04/image.png" class="kg-image" alt loading="lazy" width="1729" height="621" srcset=" __GHOST_URL__ /content/images/size/w600/2021/04/image.png 600w, __GHOST_URL__ /content/images/size/w1000/2021/04/image.png 1000w, __GHOST_URL__ /content/images/size/w1600/2021/04/image.png 1600w, __GHOST_URL__ /content/images/2021/04/image.png 1729w" sizes="(min-width: 1200px) 1200px"><figcaption>Rubeus s4u action using the aes256 hash</figcaption></figure>

Lateral movement can now be performed using a number of techniques. We can perform [DLL hijacking]( __GHOST_URL__ /edgegdi-dll-for-persistence-and-lateral-movement/) using SMB access, execute commands with WMI, setup remote scheduled tasks, etc. The below example copies an MSBuild payload onto HEADHUNTER and executes it using WMI. Note the usage of the fully qualified domain name (FQDN) is **VERY IMPORTANT**.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src=" __GHOST_URL__ /content/images/2021/03/image-6.png" class="kg-image" alt loading="lazy" width="1571" height="799" srcset=" __GHOST_URL__ /content/images/size/w600/2021/03/image-6.png 600w, __GHOST_URL__ /content/images/size/w1000/2021/03/image-6.png 1000w, __GHOST_URL__ /content/images/2021/03/image-6.png 1571w" sizes="(min-width: 1200px) 1200px"><figcaption>Lateral movement with WMI</figcaption></figure>

[Part 2]( __GHOST_URL__ /the-power-of-kerberos-part-2-constrained-delegation-to-domain-admin/) will continue from HEADHUNTER and demonstrate obtaining Domain Admin privileges using Unconstrained Delegation and the printer bug.

## References

- [https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html](https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html)
- [https://www.harmj0y.net/blog/redteaming/another-word-on-delegation/](https://www.harmj0y.net/blog/redteaming/another-word-on-delegation/)
- [https://adsecurity.org/?p=2011](https://adsecurity.org/?p=2011)
