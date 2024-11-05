### HTB Bastion   10.10.10.134

Welcome to another box on my journey to PNPT.  2 weeks left before we get it going.

So let's start here like we always do, with some nmap.

```
(kali@kali)-$ sudo nmap -T4 -p- -Pn 10.10.10.134      
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-05 03:14 EST
Nmap scan report for 10.10.10.134
Host is up (0.048s latency).
Not shown: 65522 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 32.26 seconds
```

So there is some noise in the 40000+ range but we have some good stuff to look at.  Let's dig just a bit deeper then.

```
(kali@kali)-$ sudo nmap -A -Pn -p 22,135,139,445,5985,47001 10.10.10.134
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-05 03:16 EST
Nmap scan report for 10.10.10.134
Host is up (0.042s latency).

PORT      STATE SERVICE      VERSION
22/tcp    open  ssh          OpenSSH for_Windows_7.9 (protocol 2.0)
| ssh-hostkey: 
|   2048 3a:56:ae:75:3c:78:0e:c8:56:4d:cb:1c:22:bf:45:8a (RSA)
|   256 cc:2e:56:ab:19:97:d5:bb:03:fb:82:cd:63:da:68:01 (ECDSA)
|_  256 93:5f:5d:aa:ca:9f:53:e7:f2:82:e6:64:a8:a3:a0:18 (ED25519)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Microsoft Windows Server 2016 build 10586 - 14393 (96%), Microsoft Windows Server 2016 (95%), Microsoft Windows Server 2008 SP1 (94%), Microsoft Windows 10 1507 (93%), Microsoft Windows 10 1507 - 1607 (93%), Microsoft Windows 10 1511 (93%), Microsoft Windows Server 2012 (93%), Microsoft Windows Server 2012 R2 (93%), Microsoft Windows Server 2012 R2 Update 1 (93%), Microsoft Windows 7, Windows Server 2012, or Windows 8.1 Update 1 (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-11-05T08:16:24
|_  start_date: 2024-11-05T08:13:19
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Bastion
|   NetBIOS computer name: BASTION\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2024-11-05T09:16:23+01:00
|_clock-skew: mean: -19m58s, deviation: 34m37s, median: 0s

TRACEROUTE (using port 139/tcp)
HOP RTT      ADDRESS
1   41.15 ms 10.10.14.1
2   41.24 ms 10.10.10.134

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 25.62 seconds
```

So, looks like SSH, SMB, and a couple odd http servers.  Checked both of those in a browser and wasn't able to get anything really going with them.  So I decided to take a look at the SMB portion.

```
(kali@kali)-$ smbclient -L \\\\10.10.10.134\\                           
Password for [WORKGROUP\kali]:

        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        Backups         Disk      
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 10.10.10.134 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available
```

Ok, so one of these is not like the other.  I tried all of them but was only able to get into one with no password.

```
(kali@kali)-$ smbclient \\\\10.10.10.134\\Backups
Password for [WORKGROUP\kali]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Tue Apr 16 06:02:11 2019
  ..                                  D        0  Tue Apr 16 06:02:11 2019
  note.txt                           AR      116  Tue Apr 16 06:10:09 2019
  SDT65CB.tmp                         A        0  Fri Feb 22 07:43:08 2019
  WindowsImageBackup                 Dn        0  Fri Feb 22 07:44:02 2019

                5638911 blocks of size 4096. 1178802 blocks available
```

So this one lets us in.  The notes talks about not downloading the backup cause it will take forever.  So that left me stumped, ngl.  And so I went to the walkthrough and was able to get some assistance.

`https://medium.com/@klockw3rk/mounting-vhd-file-on-kali-linux-through-remote-share-f2f9542c1f25` is the article that I was linked to and followed it, and here's what I came back with.

I went into a folder that I wanted to use for this box, cause we all love being organized, yea?  And then following the article, I did the following.  The article also had instructions on a few tools to install via `apt-get` but I'll leave that to you.

```
(kali@kali)-$ mkdir backups                                                      

(kali@kali)-$ sudo mount -t cifs -o 'rw,username=guest' //10.10.10.134/Backups backups

Password for guest@//10.10.10.134/Backups:
```

Here we used `mount` after making a directory to tie the share to, to mount the share.  -t was the type, cifs.  -o is the list of options.  Here we set it to rw (read,write) and supply the username of guest.  Then put in the address of the share and last, tell it the folder to mount to.

Now to mount the vhd file we found.

`guestmount -a '/home/kali/Documents/HTB Bastion/mount/backups/WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351/9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd' -m /dev/sda1 --ro /home/kali/Documents/HTB\ Bastion/bastion`

Lots to unpack here.

We used `guestmount` with -a to add image.  Then we filepathed out the image.  After that, we use the -m to mount it to `/dev/sda1`.  We give it --ro (read only) and then supply the folder for it to show up in.

And now we can see the contents of the vhd file.

![vhd file](/Images/HTB8Bastion/pic1.png)

And now we can see it also in the file explorer view.

![Explorer View](/Images/HTB8Bastion/pic2.png)

At this point, lots of enumeration.  We can look through all of this, and we should.  Check user desktop, documents, recycle bin, etc.  

After some looking, I didn't find anything in those but there are other things to think about.  I'm going to grab the SAM stuff, you know, the windows saved hashed passwords.  Those can be found here...

![SAM Files](/Images/HTB8Bastion/pic3.png)

You want the SAM, SECURITY, and SYSTEM files.  Move those to their own little folder to work with.  Note, do NOT grab the .log files, those will not work.  Don't worry about how I found that out lol.

Run the secretsdump.py command and you get the following output.

![secretsdump output](/Images/HTB8Bastion/pic4.png)

So we have a number of interesting items here.  We have some hashes, we have a couple keys, and we have a default password for an unknown user.

We can use hashcat to try and crack the hashes but it's no guarantee that either of them will still be relevant as this could be an older image.  Plus, we have a password to try so why not try it.

`(kali@kali)-$ ssh l4mpje@10.10.10.134`

Put the password in to give it a go and wouldn't ya know it.  It works.

![ssh](/Images/HTB8Bastion/pic5.png)

From here its a couple quick commands that we've run through many times before to get the user flag.  Now onto the escalation part.

The trail of enumeration and checking things.  `whoami /priv` didn't get anywhere and `systeminfo` got me a big ol fat access denied.  As of this second, I'm not really finding anything.  So time for some manual enumeration.

The challenge notes talk about mRemoteNG being part of this box so let's go find that.

Was able to find that in the Program Files (x86) folder.

![Program Files](/Images/HTB8Bastion/pic6.png)

Following the guided mode on HTB, it looks like we're looking for the file that holds the encrpyted passwords of users.  After some looking was able to find that file.

![nRemoteNG](/Images/HTB8Bastion/pic7.png)

I then took the contents of that file and copied it into a seperate file and used a tool called mRemoteNG-Decrypt and tried to do the file but that didn't work.  On the github there's an option for a string.  So I tried that way and looks like we got it going.  We can take the password part of the xml file and run it through the same program.

![xmlfile](/Images/HTB8Bastion/pic8.png)  

![Decrypted](/Images/HTB8Bastion/pic9.png)

And then we can use that password to ssh into the administrator account.

![root](/Images/HTB8Bastion/pic10.png)

And that's that.  This one was a little tougher and had to follow the video for it but it was good to see how this one works.  It'll be good to have this documented to look back on when I take my PNPT.

Thanks for coming along with me on this journey.  Until next time.
