![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/3f3c7e07-cdf2-46c4-ad3b-c7c77dada83e)

# Enumeration

![Pasted image 20230620135725](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/1b1fcb1a-f09e-4424-9639-936e8efb65f3)

Interesting ports:
- 8080 - Nagios NSCA. Unsure what that is...

The 8080 port is a website. I ran dirb against it to find directories:

![Pasted image 20230620135917](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/077f5151-36ea-4707-8210-0c66830545f9)

`/upload` looks interesting. 
Reviewing it, it is a simple uploader page. Trying to upload a test .txt file results in "Successful upload" but no other output. No links or any other indicators. 

# Foothold exploitation
Pointing Burp at it to see what's going on 

![Pasted image 20230620140502](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/68c2742a-7d1c-4bd3-a702-289acf3ab91c)

At the bottom, I can see the file I'm sending it, but nothing happens. If I mess with the data and put some random value below it in that box, I do get a reply back on the response that says  `Only image files are accepted!`

I'll try sending the file with a .png at the end... 
Sent:

![Pasted image 20230620140502](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/bce1e45e-2a64-4b04-96e1-882d15283c55)

Response:

![Pasted image 20230620140708](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/d4bafe17-6b11-44da-a075-a94c99af6adc)

Okay, now we are getting somewhere!
Theoretically, I should be able to send a GET request to that address and get that page as a response. Let's see if I can use this to traverse the directories...

![Pasted image 20230620140933](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/f752b702-3889-47ba-b5a6-d925eb769296)

Success! Can I see `/etc/passwd`?
Sent:

![Pasted image 20230620141014](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/be6cdce0-5149-4c08-af04-15cbbfe3f1d2)

Receive:

![Pasted image 20230620141031](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/430cf230-d448-4101-8da6-86bddac9676e)

Ding ding ding!

Now I'll look for interesting files on the system...
After some snooping I found something in a home directory. There is a user named frank who has a home directory I can read. in a hidden folder, I can see a settings.xml file.
Sent:

![Pasted image 20230620141224](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/51acfc67-d561-430e-82e0-5e54f1e3ef1b)

Results:

![Pasted image 20230620141528](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/550e8a64-7a61-4915-aba8-34be484f0735)

A password! 
After trying several places, I wasn't able to find anywhere obvious to point the credentials. 
Doing some more digging with the earlier directory traversal technique, I was able to find a file under `/var/www/webApp/HELP.md` which gives information about the webapp that the server is running. I was able to find `CVE-2022-22963 - Spring Cloud Function SpEL RCE` which looks juicy. 
It has a Metasploit module (`multi/http/spring_cloud_function_spoel_injection`) which I am able to run against the system. 

![Pasted image 20230620142055](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/75794b83-79f9-44e6-b422-b6abd2cc4508)

Yay! I am frank. Unfortunately I cannot retrieve the user flag. It looks to be owned by phil. Maybe that password *is* his password, but he just can't use ssh?

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/4799343c-f150-4d8c-befd-583ec467f25a)


And there's the user flag!

# PrivEsc
Did some digging and found files in the `/opt/` directory called automation. Reviewing the files in the directory show they are .yml playbooks which are used by ansible for automation.

![Pasted image 20230620142540](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/a93fa757-83b3-4b8a-9183-cf0edf371017)

I may be able to create my own playbook and use it to escalate my privelege...

![Pasted image 20230620142632](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/230502b7-08d2-4d6b-9c02-287bb5562d8d)


> NOTE: the syntax uses spaces and NOT tabs! if you use tab for spacing, it will fail. 

I uploaded the file to the directory with the others using [[Python Simple Server]] and `wget` on the victim machine.
Since the playbooks look to be run via a scheduled task, it should automatically run my program. If it does run, I should be able to see that `/bin/bash` has a setuid binary added. After that, running `bash -p` should be spawn a root shell. 

![image](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/25bc8fcc-4ad8-4e20-88cc-382e19170e2f)


Inject has been PWNED!
