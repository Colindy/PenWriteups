### Day 1

Got started on the knowledge check box on the getting started module in HTB.  So first things first, connected
to the VPN for HTB and away we go.

The system gen gave me a box at 10.129.42.249.  Ran first simple nmap:
```
sudo nmap -p- -sV -O 10.129.42.249
[sudo] password for kali: 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-12 20:26 EDT
Nmap scan report for 10.129.42.249
Host is up (0.095s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).
TCP/IP fingerprint:
```
So, nothing too crazy.  Scanned all the ports and looks like the only 2 open are 22 and 80.  I'm gonna
go at 80 first.  So run this next nmap command and get:
```
sudo nmap -sV -sC -p 80 10.129.42.249
Starting Nmap 7.94 ( https://nmap.org ) at 2023-07-12 20:28 EDT
Nmap scan report for 10.129.42.249
Host is up (0.037s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-robots.txt: 1 disallowed entry 
|_/admin/
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Welcome to GetSimple! - gettingstarted
```
Interesting, we have a robots.txt file.  Wonder what I'm "not supposed to see" lol.  So, open a browser and 
take a look.

At first glance it appears to be a startup page.  Looks like the default items weren't removed and/or setup is incomplete.
The robots.txt file...Disallow: /admin/
So of course, gotta see what /admin/ is all about, my guess is a login...and correct.  Hmm, since it's not set up, I wonder
if admin:admin works....HA, looks like I'm in.

Looks like the admin portal gives me a page with some tabs.  Pages, Files, Theme, Backups, and Plugins.  There are two plugins
but I want to see what I can get into with the files first.  I do see a Upload files and/or images but appears to do nothing once
I click on it.  Kinda sad.  No worries.  Maybe come back to that.

Let's see what dirb comes up with and what else is hiding on this box.  Real quick though, I doubt it works but still gotta
see if admin:admin works for the ssh....admin is a username but neither 'admin', 'getsimple', or 'getsimple!' work.  But I did
find something.  I got the following error:
```
admin@10.129.42.249's password: 
Permission denied, please try again.
admin@10.129.42.249's password: 
Permission denied, please try again.
admin@10.129.42.249's password: 
admin@10.129.42.249: Permission denied (publickey,password).
```
Looks like I can use a key instead of a password if need be.  Ok, back to dirb...
```
sudo dirb http://10.129.42.249 /usr/share/wordlists/dirb/small.txt 

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Wed Jul 12 20:53:53 2023
URL_BASE: http://10.129.42.249/
WORDLIST_FILES: /usr/share/wordlists/dirb/small.txt

-----------------

GENERATED WORDS: 959                                                           

---- Scanning URL: http://10.129.42.249/ ----
==> DIRECTORY: http://10.129.42.249/admin/                                                                                             
==> DIRECTORY: http://10.129.42.249/backups/                                                                                           
==> DIRECTORY: http://10.129.42.249/data/                                                                                              
                                                                                                                                       
---- Entering directory: http://10.129.42.249/admin/ ----
==> DIRECTORY: http://10.129.42.249/admin/inc/                                                                                         
==> DIRECTORY: http://10.129.42.249/admin/template/         
```
Nothing really here, maybe I'll come back to this later.  I wonder if there are any bugs for gettingstarted
v 3.3.15.  And there is, CVE-2019-11231.  Cool, there's a metasploit exploit for it.  Set my options for it 
and lets see where it goes.

After a quick check, there's the first flag!!  Woot woot!

Now to see if I can elevate privleges and get the root flag too.  Tried to see if I could get my linenum.sh
over to the device to see if it grabs anything interesting but wget doesn't work in metasploit.  Figures lol.

Nothing I tried so far works.
	
Tried hosting a python web server but I can't seem to get the file to download
from my local machine to the target.  Keep getting connection refused for that.

Tried poking around in meterpreter but didn't find anything.

I've got an idea, we know it hosts ssh, let's give hydra a try and see if we can brute force the root login
and get somewhere...nothing.

So, I can't get the upload function on the site to work.  I feel like the way to get it is to find privilege
escalations with either LinEnum.sh or LinPEAS.sh (because it says so in the hint).  In the lessons it had me
use the file upload to create a reverse shell to the system and then tried to load it that way using wget (had
some issues with this in the lesson too so I'll need to do more looking into that).

But it's zzz time for me!  I'll work on this more tomorrow after I look up some other writeups for this to
see where I may be going wrong with the file upload feature.

### Day 2

So I think I may have been going about getting my shell in the wrong way.  Stuck, was I, on doing in through an image
upload.  But I may also just be able to create a page and open the page. ;)
