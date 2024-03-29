
<span style="color:red">PWNED</span>

# Initial Enumeration

![[Pasted image 20230620135725.png]]

Interesting ports:
- 8080 - Nagios NSCA. Unsure what that is...

The 8080 port is a website. I ran dirb against it to find directories:![[Pasted image 20230620135917.png]]
/upload looks interesting. 
Reviewing it, it is a simple uploader page. Trying to upload a test .txt file results in "Successful upload" but no other output. No links or any other indicators. 

# Foothold exploitation
Pointing Burp at it to see what's going on 
![[Pasted image 20230620140502.png]]
At the bottom, I can see the file I'm sending it, but nothing happens. If I mess with the data and put some random below it in that box, I do get a reply back on the response that says  `Only image files are accepted!`

I'll try sending the file with a .png at the end... 
Sent:
![[Pasted image 20230620140743.png]]
Response:
![[Pasted image 20230620140708.png]]

Okay, now we are getting somewhere!
Theoretically, I should be able to send a get request to that address and get that page as a response. Let's see if I can use this to traverse the directories...
![[Pasted image 20230620140933.png]]
Success! Can I see `/etc/passwd`?
Sent:
![[Pasted image 20230620141014.png]]
Receive:
![[Pasted image 20230620141031.png]]
Ding ding ding!
Now I'll look for interesting files on the system...
Found something in a home directory. There is a user named frank who has a home directory I can read. in a hidden folder, I can see a settings.xml file.
Sent:
![[Pasted image 20230620141224.png]]
Results:
![[Pasted image 20230620141528.png]]
A password! Let's try it in a few places...
After trying several places, I wasn't able to find where to point the credentials. 
Doing some more digging with the earlier directory traversal technique, I was able to find a file under `/var/www/webApp/HELP.md` which gives information about the webapp that the server is running. I was able to find `CVE-2022-22963 - Spring Cloud Function SpEL RCE` which looks juicy. 
It has a Metasploit module (`multi/http/spring_cloud_function_spoel_injection`) which I am able to run against the system. 
![[Pasted image 20230620142055.png]]
Yay! I am frank. Unfortunately I cannot retrieve the user flag. It looks to be owned by phil. Maybe that password *is* his password, but he just can't use ssh?
![[Pasted image 20230620142210.png]]
And there's the user flag!

# PrivEsc
Did some digging and found files in the `/opt/` directory called automation. Reviewing the files in the directory show they are .yml playbooks which are used by ansible for automation. 
![[Pasted image 20230620142540.png]]
I may be able to create my own playbook and use it to escalate my privelege...
![[Pasted image 20230620142632.png]]

> NOTE: the syntax uses spaces and NOT tabs! if you use tab for spacing, it will fail. 

I uploaded the file to the directory with the others using [[Python Simple Server]] and `wget` on the victim machine.
Since the playbooks look to be run via a scheduled task, it should automatically run my program. If it does run, I should be able to see that `/bin/bash` has a setuid binary added. After that, running `bash -p` should be spawn a root shell. 
![[Pasted image 20230620143304.png]]
**PWNED**

# What I learned:

- A few new tricks with burp. Paying attention to outputs and messing with things in the repeater is a great thing
- Just because you find a password, doesn’t mean it is for ssh, and it also doesn’t mean it ISN'T the password. In this case, they just can’t ssh
- Look at everything and read about what it is. Find out what framework a website is using especially if it isn’t obvious.


