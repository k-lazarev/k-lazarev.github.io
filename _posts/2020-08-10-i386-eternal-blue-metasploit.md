---
layout: post
title: i386, EternalBlue, and Metasploit
---

One of now-depreciated *Advanced+* 32-bit machines on <a href="https://www.virtualhackinglabs.com/">Virtual Hacking Labs</a> was vulnerable to EternalBlue / DoublePulsar, and I spent countless number of hours trying to figure out why my meterpreter shell dies after 2-3 seconds, even when I target the correct architecture.<br><br>Lo and behold: specify *PROCESSINJECT* equal to *spoolsv.exe*. From my experience, this is the most reliable 32-bit process to inject into. You may also try *explorer.exe*, *wlms.exe*, and *lsass.exe*, but all of them crashed on me and I couldn't establish the initial foothold.<br><br>P. S. There's quite low chance that you will encounter 32-bit Windows machine in the wild, but I've definitely seen one in 2019.
