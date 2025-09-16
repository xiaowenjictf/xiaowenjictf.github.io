---
title: 'Nahamcon CTF 2025 - Crypto Clock writeup'
categories: ["ctf"]
tags: ["crypto"]
cover: "/images/ctfs/nahamcon2025/crypto_clock.jpg"
date: 2025-09-12T12:00:00-04:00
lastmod: 2025-09-12T12:00:00-04:00
---

# Challenge Overview

The Crypto Clock challenge from Nahamcon 2025 provides an encrypted flag and some source code that generates a pseudo-random key using the current timestamp as a seed. The goal is to recover the original flag by replicating the key generation logic.

## Initial Analysis

We are given the following encrypted flag:


{{< figure src="/images/ctfs/nahamcon2025/nc.png" alt="nc output" >}}


```python
>>> flag = "a6717b705265d3b9d810736c72b27bc8bcc5d245e68ccada61f8c16277d5a8bb5655db3cc7d0"
>>> print(len(flag))
76
>>> print(len(flag)/2)
38.0
```
> The hex-encoded string is 76 characters long, which corresponds to 38 bytes.

Our task is to replicate the key generation, then XOR it with the ciphertext to retrieve the plaintext flag.

Moving on, lets analyze the source code:

{{< figure src="/images/ctfs/nahamcon2025/challenge.png" alt="challenge code" >}}

In the `generate_key` function, they will generate the same key if the same seed is provided. The seed that they gave was the current time, using python. We need to replicate the same behavior to get the key, than XOR it with the ciphertext to get the flag.

Finally, we created our final exploit.

## Exploit

```python
import time
import random
import socket
from pwn import xor

HOST = 'challenge.nahamcon.com'
PORT = 31909

def bruteKey(size,seed):
    """Challenge Function used to generate the key."""
    if seed is not None:
        random.seed(int(seed))
    return bytes(random.randint(0, 255) for _ in range(size))

with socket.create_connection((HOST, PORT)) as s:
    # We retrieve the current time right after connecting to the port.
    current_time = int(time.time())
    print("Now it is currently:",current_time)

    # Then we retrieve only the flag part of the output.
    data = s.recv(1024).decode()
    enc_flag = data[47:123]

    # Finally, we get the size of the flag and pass it to the function that
    # generates the key.
    size = (len(enc_flag)) 
    key = bruteKey(size,current_time)

    # We need to conver the flag to bytes, to xor it with the key(that is already bytes)
    flag_bytes = bytes.fromhex(enc_flag)

    # XOR
    final_flag = xor(flag_bytes,key)

    # Clean the output
    flag = final_flag.decode('utf-8',errors='ignore')[:38]
    print(flag)
```

Flag:

```
flag{0e42ba180089ce6e3bb50e52587d3724}
```