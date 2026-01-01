---
title: Paper - HTB Machine
published: 2025-09-05
description: Paper writeup
tags: [Wordpress, LFI, CVE-2021-3560]
category: HTB
image: "images/cover.png"
draft: false
---

## Recon

```bash
nmap -sC -sV -p- 10.10.11.143 -v
```

![nmap scan output](images/nmap.png)

---

## Enumeration

### Web (port 80)

We did not find anything manually with tools such as ffuf, so we used whatweb to attempt to obtain useful information. As part of the output, a domain was discovered near the end:

![whatweb output showing discovered domain office.paper](images/whatweb.png)

```
office.paper
```

We are dealing with a WordPress instance.

![screenshot indicating WordPress instance or domain references](images/image1.png)

Reading the server content, We found a message that provided a hint indicating a possible vulnerability.

![server message indicating potential vulnerability](images/image2.png)

Consulting the referenced article, We identified the vulnerability that was being referenced:

https://0day.work/proof-of-concept-for-wordpress-5-2-3-viewing-unauthenticated-posts/

We tested the behavior on the target server:

```
http://office.paper/?static=1
```

![response from http://office.paper/?static=1 showing exposed content](images/image3.png)

We obtained the following URL:

```
http://chat.office.paper/register/8qozr226AhkCHZdyY
```

We resolved that subdomain and accessed the secret registration area:

![registration page on chat.office.paper](images/image4.png)

We created an account with the address polar@polar.com and the password polar. While exploring, We observed a chat bot accessible from the application:

![chat interface showing the bot](images/image5.png)

By inspecting user comments, We discovered a useful hint:

![comment containing a hint](images/image6.png)

We proceeded to message the bot privately. The initial approach was to attempt an XSS; however, the bot responded with a set of available commands:

![bot response listing available commands](images/image7.png)

List of commands:

![list of bot commands screenshot](images/image-1.png)

After further testing, We successfully achieved Local File Inclusion (LFI):

```
recyclops list ../../../../../../../home
```

![output of recyclops list showing directories under home](images/image-2.png)

While using the bot's list and file commands, We discovered a .env file within the bot's directory that contained credentials:

![snippet of .env file containing credentials](images/image-3.png)

From the file, We observed a password:

![revealed password from .env file](images/image-4.png)

```
Queenofblad3s!23
```

The discovered credential permitted SSH access to the account dwight.

![ssh session showing successful login as dwight](images/image-5.png)

We retrieved user.txt:

![contents of user.txt (user flag)](images/image-6.png)

```
7d11932a1b6de6738b24c5ebc107ee7d
```

---

## Post Exploitation

### Root

Execution of LinPEAS revealed the following finding:

```
Vulnerable to CVE-2021-3560
```

To exploit the vulnerability, We referenced the following public exploit repository:

https://github.com/UNICORDev/exploit-CVE-2021-3560/blob/main/exploit-CVE-2021-3560.py

We transferred the exploit to the target and executed it:

![execution of CVE-2021-3560 exploit on the target host](images/image.png)

The compromised user possessed sudo privileges:

![evidence of sudo privileges for the user](images/image-7.png)

Consequently, We obtained the final flag:

![contents of root.txt (root flag)](images/image-8.png)

```
682bc6dfe650b4a3c5eca6e5b20ffce5
```
