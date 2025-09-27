---
layout: post
title: THM Biblioteca - Writeup
date: 2025-09-26 20:46 -0400
categories: ["Writeup"]
tags: ["thm","sqli","web","python"]
---

# 🏴 THM Writeup – Biblioteca

## 📌 Challenge Info

- **CTF Name:** Biblioteca
- **Url**: https://tryhackme.com/room/biblioteca
- **Category:** Web
- **Points:** 60
- **Difficulty:** Medium


---
## 📝 Description
> This is a medium difficulty box with basic enumeration and an interesting privilege escalation vector.

## 🧠 Enumeration / Recon


```bash
# Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)

Host: 10.201.1.150 ()	Status: Up
Host: 10.201.1.150 ()	Ports: 22/open/tcp//ssh//OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)/, 8000/open/tcp//http//Werkzeug httpd 2.0.2 (Python 3.8.10)/

# Nmap done at Thu Sep 25 22:30:08 2025 -- 1 IP address (1 host up) scanned in 66.58 seconds
```

We identified 2 ports, lets start with the HTTP service in port 8000, there is a simple login and sign up page.

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-6.png)

---

## 🔍 Inititial Foothold

After registering a test user, only a simple authenticated page is shown.

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-7.png)

From the login requests we will try some sqlinjection testing.

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-8.png)

Before launching the intruder attack, add a payload processing option, in this case replace `__TIME__` with 10, to specify 10 seconds of waiting on our blind sqli tests.

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-9.png)

The following request confirms a blind sql injection.

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-10.png)

Sending this to repeater we can confirm the request waits 10 seconds before sending a response confirming a SQL injection vulnerability.

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-11.png)

Send this request to sqlmap and proceed to dump the database website.

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-1.png)

From the database we extracted a set of credentials.

```
smokey@email.boop | <redacted> | smokey
```

Then, access via ssh with this credentials.

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926.png)

We found 4 working users, doing some search we can find where is `user.txt` is located, actually is under hazel home directory. Therefore we need to find a way to escalate privileges to this user.

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-2.png)

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-3.png)

After doing a lot of enumeration in the box I couldn't find anything useful to switch the user to hazel. So the last try is to test some obvious passwords directly.

So trying username as password worked! `hazel:hazel` (sometimes this is not straightforward), so I noticed this password is also contained in many seclists wordlists, so there would be a little slower way to get the right password brute forcing too.

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-12.png)
## 🏁 Rooting

Checking sudo rights with  `sudo -l`, we can see that this user is able to run the script as root, so this must be the way to escalate privileges to root.

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-13.png)

The file contains, a script that generates a hash representation using the library `hashlib.py`, at first sight there is nothing unusual or vulnerable in the code.
```python
import hashlib

def hashing(passw):
    md5 = hashlib.md5(passw.encode())
    print("Your MD5 hash is: ", end ="")
    print(md5.hexdigest())
    sha256 = hashlib.sha256(passw.encode())
    print("Your SHA256 hash is: ", end ="")
    print(sha256.hexdigest())
    sha1 = hashlib.sha1(passw.encode())
    print("Your SHA1 hash is: ", end ="")
    print(sha1.hexdigest())

def main():
    passw = input("Enter a password to hash: ")
    hashing(passw)

if __name__ == "__main__":
    main()
```

After doing some research, I found a really consistent article about this kind of vulnerabilities: [Here](https://medium.com/analytics-vidhya/python-library-hijacking-on-linux-with-examples-a31e6a9860c8).

> The **_PYTHONPATH_** environment variable indicates a directory (or directories), where Python can search for modules to import.
> It can be abused if the user got privileges to set or modify that variable, usually through a script that can run with **_sudo_** permissions and got the **_SETENV_** tag set into **_/etc/sudoers_** file. -

Now to exploit this we need to create a fake `hashlib.py` ,  inside the `/tmp/` directory, 

```
touch /tmp/hashlib.py
```

Edit the file the the following content

```python
import os
os.system("/bin/bash")
```

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-14.png)

Finally execute the following command to get a root shell

```bash
sudo PYTHONPATH=/tmp/ /usr/bin/python3 /home/hazel/hasher.py
```

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-15.png)

Get the flag!

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-16.png)

---
## 🏆 Win

![](/assets/img/posts/2025-09-26-thm-biblioteca-writeup/biblioteca-20250926-5.png)

---
## 🤔 Lessons Learned

- Check libraries permissions
- Brute force only as last resort, sometimes the obvious and easiest works better.


---
## 📚 References
- https://medium.com/analytics-vidhya/python-library-hijacking-on-linux-with-examples-a31e6a9860c8