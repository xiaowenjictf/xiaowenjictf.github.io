---
title: 'Paper - HTB Writeup'
categories: ["htb"]
tags: ["wordpress"]
cover: "https://i.ebayimg.com/00/s/MTA2NlgxNjAw/z/6T4AAOSwQkNgXJyd/$_57.JPG?set_id=8800005007"
date: 2025-09-05
lastmod: 2025-09-05
---

## Recon

```bash
nmap -sC -sV -p- 10.10.11.143 -v
````

{{< figure src="/images/htb/paper/nmap.png" alt="nmap scan output" >}}

---

## Enumeration

# Web (port 80)

We did not find anything manually with tools such as `ffuf`. Following a suggestion from ChatGPT, We used `whatweb` to attempt to obtain useful information.

As part of the output, a domain was discovered near the end:

{{< figure src="/images/htb/paper/whatweb.png" alt="whatweb output showing discovered domain office.paper" >}}

```
office.paper
```

We are dealing with a WordPress instance.

{{< figure src="/images/htb/paper/image1.png" alt="screenshot indicating WordPress instance or domain references" >}}

Reading the server content, We found a message that provided a hint indicating a possible vulnerability.

{{< figure src="/images/htb/paper/image2.png" alt="server message indicating potential vulnerability" >}}

Consulting the referenced article, We identified the vulnerability that was being referenced:

```
https://0day.work/proof-of-concept-for-wordpress-5-2-3-viewing-unauthenticated-posts/
```

We tested the behavior on the target server:

```
http://office.paper/?static=1
```

{{< figure src="/images/htb/paper/image3.png" alt="response from [http://office.paper/?static=1](http://office.paper/?static=1) showing exposed content" >}}

We obtained the following URL:

```
http://chat.office.paper/register/8qozr226AhkCHZdyY
```

We resolved that subdomain and accessed the secret registration area:

{{< figure src="/images/htb/paper/image4.png" alt="registration page on chat.office.paper" >}}

We created an account with the address `polar@polar.com` and the password `polar`.

While exploring, We observed a chat bot accessible from the application:

{{< figure src="/images/htb/paper/image5.png" alt="chat interface showing the bot" >}}

By inspecting user comments, We discovered a useful hint:

{{< figure src="/images/htb/paper/image6.png" alt="comment containing a hint" >}}

We proceeded to message the bot privately.

The initial approach was to attempt an XSS; however, the bot responded with a set of available commands:

{{< figure src="/images/htb/paper/image7.png" alt="bot response listing available commands" >}}

List of commands:

{{< figure src="/images/htb/paper/image-1.png" alt="list of bot commands screenshot" >}}

After further testing, We successfully achieved Local File Inclusion (LFI):

```
recyclops list ../../../../../../../home
```

{{< figure src="/images/htb/paper/image-2.png" alt="output of recyclops list showing directories under home" >}}

While using the bot's `list` and `file` commands, We discovered a `.env` file within the bot's directory that contained credentials:

{{< figure src="/images/htb/paper/image-3.png" alt="snippet of .env file containing credentials" >}}

From the file, We observed a password:

{{< figure src="/images/htb/paper/image-4.png" alt="revealed password from .env file" >}}

```
Queenofblad3s!23
```

The discovered credential permitted SSH access to the account `dwight`.

{{< figure src="/images/htb/paper/image-5.png" alt="ssh session showing successful login as dwight" >}}

We retrieved `user.txt`:

{{< figure src="/images/htb/paper/image-6.png" alt="contents of user.txt (user flag)" >}}

```
7d11932a1b6de6738b24c5ebc107ee7d
```

---

## Post Exploitation

# Root

Execution of LinPEAS revealed the following finding:

```
Vulnerable to CVE-2021-3560
```

To exploit the vulnerability, We referenced the following public exploit repository:

```
https://github.com/UNICORDev/exploit-CVE-2021-3560/blob/main/exploit-CVE-2021-3560.py
```

We transferred the exploit to the target and executed it:

{{< figure src="/images/htb/paper/image.png" alt="execution of CVE-2021-3560 exploit on the target host" >}}

The compromised user possessed sudo privileges:

{{< figure src="/images/htb/paper/image-7.png" alt="evidence of sudo privileges for the user" >}}

Consequently, We obtained the final flag:

{{< figure src="/images/htb/paper/image-8.png" alt="contents of root.txt (root flag)" >}}

```
682bc6dfe650b4a3c5eca6e5b20ffce5
```
