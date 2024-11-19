### THM Convert My Video

So this one is a tougher one.  When we run our nmap, we get back port 22 and 80.  So first things first, let's check the website.

Do that and it's a youtube video conversion site.

![site](/Images/HTB9ConvertVideo/pic1.png)

Check the source code, nothing.  Check dirbuster, nothing there either.  The only noticeable avenue is via the input box on the site.  So, fire up burpsuite and let's see what this thing does.

We fire up burpesuite and try a couple submissions and we see that it's building a URL from the input on the site.  From our initial scan we can see this is likely a linux box.  I wonder if we can do some command stuff.

Send the packet to repeater and try a couple things.  Learn something new as well.  In the course of trying different things, we find that it doesn't really like the spaces " " or some other things.  The space though is what is killer.  Try html code for the space (`%20`) and that doesn't really work either.  We're told about another way to enter a space, `${IFS}`.  This will allow us to add in spaces.  Give that a try with a ping command and it hangs.  Looks like we might have something here.

Let's give it another try and see if we can get something to work for us.  So going to try a reverse shell and it took some trys but was finally able to get it.  There's more than one way to run a command.

First, we create our shell file and name it `rev.sh`.  Then put it into a folder where we're hosting via port 80 and use wget in order to grab the file.

![burp1](/Images/HTB9ConvertVideo/pic2.png)

This was the one that finally kicked off the download of the file.  We can see that cause it's in the reply in burp and we also see the connection log where we started the http server.

Ok, so now to getting the file to run.

```
yt_url=`chmod${IFS}+x${IFS}rev.sh`
```

Tried this but it didn't take because of the "+" sign.  How else can we change the file permissions?

```
yt_url=`chmod${IFS}777${IFS}rev.sh`
```

Do that and the output we get talks about the command but nothing about an error.

```
yt_url=`./rev.sh`
```

Try this and it doesn't like the "." so we gotta try something else again.

```
yt_url=`bash${IFS}rev.sh`
```

That works as we now have our reverse shell.

![www-data shell](/Images/HTB9ConvertVideo/pic3.png)

Ok, so now we have a bit of a foothold.  So looking at our enumeration, fire off `linpeas.sh` and it doesn't really kick back anything.  Time for manual mode.  Check a couple things but `ps aux` is what gives us the best info for this box.

We checked `crontabs` and didn't see anything but when we run `ps aux` we can see that `cron` is being utilized by root.

![cron](/Images/HTB9ConvertVideo/pic4.png)

Here we can use a tool called `pspy64` to check the cron jobs even with the user we are currently.  So, go out and download that grabbing the 64 bit version since we're attacking a 64 bit linux box.

Run that and let's take a look.

![pspy64](/Images/HTB9ConvertVideo/pic5.png)

Look at that.  Looks like when that cron runs, it runs a file called `clean.sh`.  When we look at that file we see that `www-data` is the user that owns that file.  That's the user we're on.  We can modify that file.

![clean.sh](/Images/HTB9ConvertVideo/pic6.png)

Now that we've updated that, start our listener and wait.

![root](/Images/HTB9ConvertVideo/pic7.png)

And we have our root!!  This one was way tougher!!  Learned lots of new tricks in this one.  Let's keep it going.  One more before my night is done.
