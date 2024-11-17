### THM UltraTech

So time for a new box.  This is tied to the docker lesson.  So let's have at it.

First, ran through the nmaps.  Not going to paste them as I ran the same commands I always run.  What I found though was some intereting items.

```
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
8081/tcp  open  blackice-icecap
31331/tcp open  unknown
```

And with the -A on the specific ports

```
PORT      STATE SERVICE VERSION
21/tcp    open  ftp     vsftpd 3.0.3
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 dc:66:89:85:e7:05:c2:a5:da:7f:01:20:3a:13:fc:27 (RSA)
|   256 c3:67:dd:26:fa:0c:56:92:f3:5b:a0:b3:8d:6d:20:ab (ECDSA)
|_  256 11:9b:5a:d6:ff:2f:e4:49:d2:b5:17:36:0e:2f:1d:2f (ED25519)
8081/tcp  open  http    Node.js Express framework
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
|_http-cors: HEAD GET POST PUT DELETE PATCH
31331/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: UltraTech - The best of technology (AI, FinTech, Big Data)
```

So looks like two ports hosting a web page.  Let's take a look at that.  So I need to get into the habit of opening burpsuite every time I go to a webpage.  We start that up here just kinda roam the page.  Click the links, see where things go, etc.

But, we check that this time and don't really get a whole lot on first check.  Hmm.  What else.  No mention of robots.txt file in the nmap scan but never hurts to check manually.  And wouldn't ya know it.  We do have a robots.txt file.

```
Allow: *
User-Agent: *
Sitemap: /utech_sitemap.txt
```

So, go check out `/utech_sitemap.txt` and we get the following.

```
/index.html
/what.html
/partners.html
```

In combing through and looking we already saw index.html and what.html.  The new one here is the `/partners.html`.

Go check out that page and we've got a log in.  admin/admin doensn't work, the 'fogot your password' link does nothing.  Check the page source and all the way down at the bottom is a javascript.  If we look at the javascript file there's no obfuscation of any sort, its plainly readable.

![js code](/Images/THM5UltraTech/pic1.png)

This is the part we're looking for.  Above that, you can see the function being made to grab the hostname and append port 8081 and that's what gets called into the `const url` command.  Well, we have that, its the IP of the machine itself.  Let's copy that url into the address bar, replace the machine IP and give it a go to see what it does.

`http://10.10.131.192:8081/ping?ip=127.0.0.1`

![code execution](/Images/THM5UltraTech/pic2.png)

Looks like we can execute a ping in here.  Tried adding other commands on to the end, nothing there either.  Turns out you can put backticks (the button right below the esc without shift, "`") and that will cause the command to take precedence in that instance.  It will run the command inside that and then whatever command on top of that.

![code injection](/Images/THM5UltraTech/pic3.png)

So take that utech.db.sqlite and cat that out and we get this.

![passwords](/Images/THM5UltraTech/pic4.png)

Looks like we have `r00t` and `admin` along with a couple of password hashes.  We know what to do with those.  Run them through hashcat.  So get those hashes, put them in my hash.txt file and run it through hashcat.

The first runthrough it kicks back that it needs the mode.  Lot of MD5 variations showed up so just try basic MD5 with this command.

`hashcat.exe -m 0 hash.txt rockyou.txt`

And that gave us the following output.

![hash output](/Images/THM5UltraTech/pic5.png)

Give the both of those a try with ssh and admin didn't work, but the r00t user did.

Now that we've got a foothold, upload linpeas.sh and run that and almost immediately they pinpoint docker.

![linpeas output](/Images/THM5UltraTech/pic6.png)

Go check gtfobins and we see the command here.

`CONTAINER_ID="$(docker run -d alpine)"` 

We change it though cause we're not running alpine, we want to run bash.

```
r00t@ultratech-prod:~$ docker run -v /:/mnt --rm -it bash chroot /mnt sh
# whoami
root
```

![whoami](/Images/THM5UltraTech/pic7.png)


