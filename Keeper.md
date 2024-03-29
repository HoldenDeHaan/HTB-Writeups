![Pasted image 20230815085357](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/8bffcd8c-b771-437d-966b-2f6b99a26148)

# Enumeration
nmap scan:
```
Nmap scan report for 10.129.208.137
Host is up (0.086s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3539d439404b1f6186dd7c37bb4b989e (ECDSA)
|_  256 1ae972be8bb105d5effedd80d8efc066 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Supported Methods: GET HEAD
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two ports open: 22 (not likely) and 80 (http)

Checking the website, I can see that there is a link asking me to go to ticketets.keeper.htb/rt/ so now I have a hostname for /etc/hosts as well as a subdomain. 

# Foothold

Going to the site shows a login:

![Pasted image 20230815090351](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/22ecb299-ad85-4380-ac8e-5be51b43372d)

Given the difficulty of the box, this is likely my point of entry.

Tried a few XSS vulnerabilites that I read about, but they don't seem to work. would probably need to find more functionality for this site before that, but maybe I can find the default credentials?
Default password for root is still in place and am able to log in:
```
username: root
password: password
```

From here I can see that there is a single ticket that was already created previously. It talks about a crash and log files for it being included. The admin makes mention of removing the attachment after saving for security reasons... 
Checking the original ticket email, I get a user named `Lise` and another possible username of `Inorgaard`

It also mentions an issue with keepass client on Windows. This is a password storage program like lastpass or similar apps. Looks pretty bare bones overall. 

After some digging around in the web interface, I can view users and their comments. The admin left a password for the user in the comments. 

```
username: lnorgaard
password: Welcome2023!
```

Trying to use these credentials for ssh:
```
┌─[eu-release-1]─[10.10.14.73]─[htb-th1stle@htb-4hana8jksq]─[~]
└──╼ [★]$ ssh lnorgaard@10.129.208.137
The authenticity of host '10.129.208.137 (10.129.208.137)' can't be established.
ECDSA key fingerprint is SHA256:apkh696g2/uAeckIXd6eFvgmvmPqoEj41w4ia45OfrI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.208.137' (ECDSA) to the list of known hosts.
lnorgaard@10.129.208.137's password: 
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-78-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
You have mail.
Last login: Tue Aug  8 11:31:22 2023 from 10.10.14.23
lnorgaard@keeper:~$ pwd
/home/lnorgaard
lnorgaard@keeper:~$ cat user.txt 
4XXXXXXXXXXXXXXXXXXXXXXXXXXXXXX2
lnorgaard@keeper:~$ 
```
Tasty user flag!

# Priv-Esc
Within the home directory is a zip file. 
Extracting the zip file gives me two new files:
```
-rwxr-x--- 1 lnorgaard lnorgaard 253395188 May 24 12:51 KeePassDumpFull.dmp
-rwxr-x--- 1 lnorgaard lnorgaard      3630 May 24 12:51 passcodes.kdbx
```
Looks like these are files from KeePass. Researching these file names, the `.kdbx` file is the backup of the password vault essentially and the`.dmp` file is a snapshot of the running process as it crashed. The dmp file might contain passwords for the vault itself. Unfortunately it contains a lot of junk characters that make it nearly impossible to sort through even with the strings command. I'll need to dig around to see how I can open it. 
I did find a poc program on github that might do the trick: 
https://github.com/CMEPW/keepass-dump-masterkey

It did come back with something strange:

![Pasted image 20230815102451](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/79bf4bd9-1a24-4b97-826c-779d50c3db4e)

Looks like it contains special characters... I did see that the user was Danish so it might be using nordic characters. 

After some googling, I get a Danish berry pudding called Rødgrød Med Fløde

That might be the password for the kdbx file.

On a windows machine, I installed KeePass and opened the file. The password was, `rødgrød med fløde` with the special characters. 
> Unrelated, but looks to be a Danish berry pudding. Lovely!


Root password looks to be `F4><3K0nd!`, but doesn't work to SSH....

Inside the KeePass notes section is an RSA private key file. I'll need to make an RSA key using this private key text and then I should be able to SSH into the box as root!
The key is for PuTTY RSA and not OpenSSH. Putty does have a tool called Puttygen that can convert it, but the private key is configured using a newer standard and the repository on my linux box isn't current enough to support it... boo. 
I need to use putty key generator to convert it to something the version on linux can use as it is written in version 3 while the linux distro version is version 2. Thankfully this can be done on the windows machine as I can just grab the latest version from the main website very easily. Once I convert it from version 3 to 2, I exported it to my linux machine to convert. 
From there I used the following command to convert it from putty ssh key to openssh key:
```bash
$ puttygen id_rsa.ppk -O private-openssh -o id_rsa

$ chmod 600 id_rsa
```

Now I should be able to ssh as root:
```
$ ssh root@keeper.htb -i id_rsa
The authenticity of host 'keeper.htb (10.129.208.137)' can't be established.
ECDSA key fingerprint is SHA256:apkh696g2/uAeckIXd6eFvgmvmPqoEj41w4ia45OfrI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'keeper.htb' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-78-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

You have new mail.
Last login: Tue Aug  8 19:00:06 2023 from 10.10.14.41
root@keeper:~# whoami && id
root
uid=0(root) gid=0(root) groups=0(root)
root@keeper:~# 
```

That's root!
