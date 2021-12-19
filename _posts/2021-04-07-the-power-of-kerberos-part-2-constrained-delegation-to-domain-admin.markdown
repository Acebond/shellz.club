---
layout: post
title: 'The Power of Kerberos Part 2: Unconstrained Delegation'
date: '2021-04-07 07:10:08'
---

We are continuing from Part 1 and leveraging Unconstrained Delegation on HEADHUNTER to gain Domain Admin privileges via the printer bug.

<figure class="kg-card kg-image-card"><img src="/images/2021/04/image-1.png" class="kg-image" alt loading="lazy" width="694" height="464" srcset="/images/size/w600/2021/04/image-1.png 600w,/images/2021/04/image-1.png 694w"></figure>
## Preconditions for Unconstrained Delegation

1. Control of a host configured for Unconstrained Delegation, this could be admin access, the machine account password or Kerberos keys.
2. Method to coerce a high value target to authenticate to the Unconstrained Delegation host. The printer bug is a universally good choice.

## Background Knowledge

I wanted to provide a high-level overview of Kerberos authentication for those who are unfamiliar. Skip this section if you already understand the basic of Kerberos authentication.

1. When a user authenticates on a domain-joined device, the password they input is hashed and used as a key to encrypt a timestamp which is sent to the Key-Distribution Center (KDC), which is located on the domain controller. This encrypted timestamp is often called **AS-REQ** (Authentication Server Request).
2. The domain controller will verify the AS-REQ using its stored version of the password hash, check the timestamp is within acceptable limits, check logon restrictions, group memberships, the account status, etc. The KDC then responds with an **AS-REP** (Authentication Server Reply) which, assuming all checks pass, will contain a TGT (Ticket-Granting-Ticket). This ticket is encrypted with the Kerberos service account, known as KRBTGT.
3. Now the user can request a TGS (Ticket-Granting-Service), a ticket to access a service principal. This is done by providing the TGT and requested SPN in a **TGS-REQ** (TGS Request) to the KDC. 
4. The KDC sends back the requested TGS, which is encrypted with the hash of the service account associated with the SPN. This step is often called **TGS-REP** (TGS Response). The TGS contains a Privileged Attribute Certificate (PAC) that contains information about the user and their group memberships.
5. The user can now present the TGS to the service they wish to access, which will decrypt the TGS using its machine account password, and determine if access should be granted using the PAC.

When a host is configured for Unconstrained Delegation, the KDC will include the TGT inside the TGS in the **TGS-REP**. This will only occur, if the **TGS-REQ** is for a SPN on a host which has the attribute **TrustedForDelegation** set to **True**. The Unconstrained Delegation host will open the TGS and place the TGT into LSASS memory for the purpose of impersonating the user.

## Performing the Attack - The Normal Way

The entire attack depends on precondition #1 having already been achieved. There are a number of methods that can be used for precondition #2, but this blog post will use the printer bug, which worked in my lab environment of a fully updated Windows Server 2016 and 2019. The printer bug is a “feature” within the Windows Print System Remote Protocol that allows a host to query another host, asking for an update on a print job.

First we use Rubeus on HEADHUNTER to monitor for, and extract all Ticket-Grant-Tickets (TGTs) for our target user, which will be the SKYFORTH domain controller machine account. Omitting the **/targetuser** will display all captured TGTs which is recommended in the real-world.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/04/image-2.png" class="kg-image" alt loading="lazy" width="1278" height="384" srcset="/images/size/w600/2021/04/image-2.png 600w,/images/size/w1000/2021/04/image-2.png 1000w,/images/2021/04/image-2.png 1278w" sizes="(min-width: 1200px) 1200px"><figcaption>Rubeus monitor action</figcaption></figure>

We can now execute the [printer bug](https://github.com/leechristensen/SpoolSample) to coerce SKYFORTH (a domain controller) into authenticating to HEADHUNTER. This will cause the SKYFORTH machine account to authenticate to HEADHUNTER, and since HEADHUNTER is configured for Unconstrained Delegation, it will receive the TGT for SKYFORTH. We can extract the TGT from HEADHUNTER and impersonate the domain controller machine account.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/04/image-3.png" class="kg-image" alt loading="lazy" width="1569" height="1268" srcset="/images/size/w600/2021/04/image-3.png 600w,/images/size/w1000/2021/04/image-3.png 1000w,/images/2021/04/image-3.png 1569w" sizes="(min-width: 1200px) 1200px"><figcaption>Printer bug execution and TGT extraction</figcaption></figure>

We can use the replication rights of the SKYFORTH account to perform a synchronization operation to download any secret material stored within the domain. The replication rights can also be used to push changes to the Active Directory data store, although I prefer to avoid modifications when possible.

Mimikatz can be used to perform the synchronization operation, and I have chosen to read the key material for the KRBTGT account. This can be used to create a golden ticket, providing unrestricted access to every domain-joined device within the forest. I decided to perform the post exploitation within the context of the unprivileged Chaos\Bravo account to ensure no accidental usage of Chaos\Alpha privileges.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/04/image-4.png" class="kg-image" alt loading="lazy" width="1548" height="1123" srcset="/images/size/w600/2021/04/image-4.png 600w,/images/size/w1000/2021/04/image-4.png 1000w,/images/2021/04/image-4.png 1548w" sizes="(min-width: 1200px) 1200px"><figcaption>Usage of the SKYFORTH TGT to create a golden ticket for Chaos\Alpha</figcaption></figure>

Domain Admin access has been achieved as evident by listing the directory contents of the CHAOS-DC C$ share.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/04/image-5.png" class="kg-image" alt loading="lazy" width="1506" height="850" srcset="/images/size/w600/2021/04/image-5.png 600w,/images/size/w1000/2021/04/image-5.png 1000w,/images/2021/04/image-5.png 1506w" sizes="(min-width: 1200px) 1200px"><figcaption>Proof the golden ticket works by accessing the C$ share on the CHAOS-DC</figcaption></figure>
## Performing the Attack - The Unconventional Way

The above attack can be performed without executing any code on HEADHUNTER. If you recall from precondition #1, I mentioned that the machine account password is sufficient to complete the attack. We can dump the machine account password using the access achieved in Part 1 and a modified version of [SharpSecDump](https://github.com/G0ldenGunSec/SharpSecDump). &nbsp;I added the following code to the `PrintSecret` function within the if statement condition for displaying the `$MACHINE.ACC` secret.

<figure class="kg-card kg-code-card"><pre><code class="language-csharp">var computerAcct = System.Text.Encoding.Unicode.GetString(secretBlob.secret);
secretOutput += computerAcct + "\r\n";
secretOutput += "base64: " + System.Convert.ToBase64String(System.Text.Encoding.UTF8.GetBytes(computerAcct));</code></pre>
<figcaption>Print raw and base64 machine account password</figcaption></figure><figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/04/image-10.png" class="kg-image" alt loading="lazy" width="1439" height="749" srcset="/images/size/w600/2021/04/image-10.png 600w,/images/size/w1000/2021/04/image-10.png 1000w,/images/2021/04/image-10.png 1439w" sizes="(min-width: 1200px) 1200px"><figcaption>HEADHUNTER machine account password retrieved remotely</figcaption></figure>

Use the machine account password to generate the AES Kerberos keys for HEADHUNTER. I used Powermad, although there are several tools that can perform the calculations. For computer accounts, the salt is the uppercase realm name + the literal word "host" + the lowercase FQDN of the host.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/04/image-11.png" class="kg-image" alt loading="lazy" width="1440" height="166" srcset="/images/size/w600/2021/04/image-11.png 600w,/images/size/w1000/2021/04/image-11.png 1000w,/images/2021/04/image-11.png 1440w" sizes="(min-width: 1200px) 1200px"><figcaption>Calculate Kerberos key from machine account password</figcaption></figure>

We can actually coerce SKYFORTH into send it's TGT to an arbitrary IP address by configuring an SPN on HEADHUNTER for a hostname that does not exist, then configuring a DNS entry for that hostname, and lastly, using the printer bug to coerce authentication to the newly created hostname.

We can easily create the SPN within the context of CHAOS\Bravo using the GenericWrite privileges from Part 1.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/04/image-12.png" class="kg-image" alt loading="lazy" width="1053" height="185" srcset="/images/size/w600/2021/04/image-12.png 600w,/images/size/w1000/2021/04/image-12.png 1000w,/images/2021/04/image-12.png 1053w"><figcaption>SPN created for host/pwned.chaos.local using CHAOS\Bravo</figcaption></figure>

Alternatively, without these privileges, we can use the HEADHUNTER machine account to create the SPN using the msDS-AdditionalDnsHostName method described by [Dirk-jan](https://dirkjanm.io/krbrelayx-unconstrained-delegation-abuse-toolkit/#control-over-serviceprincipalname-attribute-of-the-unconstrained-delegation-account). Read his amazing write-up to understand the technical details.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/04/image-17.png" class="kg-image" alt loading="lazy" width="2000" height="749" srcset="/images/size/w600/2021/04/image-17.png 600w,/images/size/w1000/2021/04/image-17.png 1000w,/images/size/w1600/2021/04/image-17.png 1600w,/images/size/w2400/2021/04/image-17.png 2400w" sizes="(min-width: 1200px) 1200px"><figcaption>SPN creation using via msDS-AdditionalDnsHostName method using HEADHUNTER$ account</figcaption></figure>

By default, authenticated users have the 'Create all child objects' permission on the Active Directory-Integrated DNS zone. Most records that do not currently exist in an AD zone can be added/deleted. Powermad can be used to create a DNS entry for the pwned.chaos.local hostname. I choose an IP address within EC2, which will be running our [krbrelayx](https://github.com/dirkjanm/krbrelayx) capture server.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/04/image-14.png" class="kg-image" alt loading="lazy" width="1206" height="179" srcset="/images/size/w600/2021/04/image-14.png 600w,/images/size/w1000/2021/04/image-14.png 1000w,/images/2021/04/image-14.png 1206w" sizes="(min-width: 1200px) 1200px"><figcaption>DNS entry created for pwned.chaos.local</figcaption></figure>

For the capture server to be able to decrypt the TGS, it must be provided the previously calculated Kerberos AES256 key for HEADHUNTER. The krbrelayx toolkit automates the entire capture process, including decryption of the received TGS and extraction of the TGT.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/04/image-15.png" class="kg-image" alt loading="lazy" width="1902" height="354" srcset="/images/size/w600/2021/04/image-15.png 600w,/images/size/w1000/2021/04/image-15.png 1000w,/images/size/w1600/2021/04/image-15.png 1600w,/images/2021/04/image-15.png 1902w" sizes="(min-width: 1200px) 1200px"><figcaption>Received TGT for SKYFORTH$</figcaption></figure>

Run the printer bug, asking SKYFORTH to send the status of print jobs to pwned.chaos.local, our capture server. During the authentication process, SKYFORTH will checks for any CIFS SPNs associated with pwned.chaos.local, and find the SPN we created. Note that HOST is a catch-all SPN, which includes CIFS. Since the SPN exists on a server with the attribute **TrustedForDelegation** set to **True** , SKYFORTH (being a KDC itself) will include its TGT inside the TGS sent.

<figure class="kg-card kg-image-card kg-width-wide kg-card-hascaption"><img src="/images/2021/04/image-16.png" class="kg-image" alt loading="lazy" width="1071" height="72" srcset="/images/size/w600/2021/04/image-16.png 600w,/images/size/w1000/2021/04/image-16.png 1000w,/images/2021/04/image-16.png 1071w"><figcaption>Printer bug execution - ignore the error that is often display. This is referring to the RPC server on our capture server being unavailable after authentication has occurred.</figcaption></figure>

This ticket can be used as shown in The Normal Way to gain Domain Admin privileges. Part 3 will continue from CHAOS-DC and pivot forests [turns out you cannot have Constrained Delegation configured across forest trusts] demonstrate attacks using the final type of Kerberos Delegation, Constrained Delegation.

## References

- [https://dirkjanm.io/krbrelayx-unconstrained-delegation-abuse-toolkit/](https://dirkjanm.io/krbrelayx-unconstrained-delegation-abuse-toolkit/)
- [https://github.com/dirkjanm/krbrelayx](https://github.com/dirkjanm/krbrelayx)
