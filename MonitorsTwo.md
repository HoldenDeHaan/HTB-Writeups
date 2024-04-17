![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/3ca5dd16-5871-4c55-859a-8e41588d189b)

Nmap:
```
Nmap scan report for 10.129.228.231
Host is up (0.029s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.21 seconds
```
Open ports:
- 22 - SSH - Not likely but useful if I find credentials
- 80 - HTTP - Web page. Likely the place to start.

Navigating to the website shows a login page:
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/e9e7a915-0a06-4a09-b8c1-bdb5f5999162)
It also contains version information about the application. The usual stuff like default credentials and simple blind SQL injection does nothing. 

# Foothold

Researching the version of the app reveals a metasploit script for it! 
After getting a shell, I found an interesting file:
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/ddb4c8ce-d9ca-42bf-b694-30fd9a1ca1af)

Checking /etc/passwd shows there aren’t any other users on this box. 

Analyzing the script, it look slike it is used to initiate a docker container running the cacti website. Might be useful, but unsure. Will want to run an enumeration script next I think. It doesn’t look like there is a user flag? 
Maybe it’s because I’m actually inside a docker container right now? One simple way to check is to look for system files related to docker:
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/b3b446d0-2ffd-498b-a765-ce3309c843a2)

Yep, this is a docker container. This would explain why there was a lack of /home/ directory. 

# Docker Fun!
Now I need to get out of docker into the main system running docker. [Hacktricks](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/docker-security/docker-breakout-privilege-escalation) has some information. [Additional steps I found helpful can be found here](https://book.hacktricks.xyz/linux-hardening/privilege-escalation/linux-capabilities).

Looks like I'll need to get root on the docker container before I can break out. Following standard privesc practices makes this extremely easy. I was eventually able to get root on the container.
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/f8836686-c93b-4b0b-ae9d-005bd81b1a8d)

Now to escape. 

# Escape Docker Jail
Now that I am root for the container, I need to escape to the host box. Should be fun. 

Looks like the container is pretty secure from what I can tell…

Looking over the code again, it is sending a sql query about databases. Running the script lets me see the databases available:
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/73ec8d6c-b61a-4da4-be54-371bc75e0206)

User auth seems like a good choice to look into. 
Reading through the script again, everything in the mysql command after `-e` is used to query the database. If I change the command from `show tables` to `select * from user_auth` I should be able to view info inside of the table… 
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/39c5809c-b8a8-408d-b91e-fb17b94785aa)

Bingo! Those look like password hashes. I have one for admin and Marcus. Marcus probably used the same username on the server for login as the database. 
Hash:
```
$2y$10$vcrYth5YcCLlZaPDj6PwqOYTw68W1.3WeKlBn70JonsdW/MhFYK4C
```
Using John the Ripper, I was able to decode the hash:
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/1aef67b1-6d60-4679-8e6f-bb5b51a30da9)

Using SSH, I'm able to get in with the follwoing credentials:
```
Marcus : funkymonkey
```

# Escalation
After some snooping, Marcus has some mail! These are messages from the admin. It details three CVEs that they need to look out for:
- CVE-2021-33033 - Kernel exploit <5.11.14
- CVE-2020-25706 - XSS vuln for Cacti
- CVE-2021-41091 - Docker exploit
  
That last one seems juicy. I was able to find a proof of concept: https://github.com/UncleJ4ck/CVE-2021-41091
Following the instructions, setuid for /bin/bash and then ran the script:
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/b49ef3f9-0270-4216-b544-f0eab3f2ae10)

MonitorsTwo has been PWNED!
