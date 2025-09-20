---
layout: post
title: THM Watcher - Writeup
date: 2025-09-20 16:58 -0400
tags: ["thm","writeup","lfi","web"]
---

In this occasion we will be solving TryHackMe's room
URL: https://tryhackme.com/room//assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher

## recon
```shell
# Nmap 7.94SVN scan initiated Fri Sep 19 15:47:51 2025 as: nmap -sCV --min-rate=1500 -n -p- --open --max-retries=1 -oA /tmp//assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher -vv 10.201.75.132
Host: 10.201.75.132 ()	Status: Up
Ports: 
	21/open/tcp//ftp//vsftpd 3.0.5/, 
	22/open/tcp//ssh//OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)/, 
	80/open/tcp//http//Apache httpd 2.4.41 ((Ubuntu))
```

Now we start inspecting the web application hosted in the machine using Nuclei.

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250919.png)

From the output we can see 2 interesting paths, accessing, we can retrieve the first flag here `flag_1.txt`, the second path is protected but lets keep this path for later.

```
/flag_1.txt
/secret_file_do_not_read.txt
```
**Local File Inclusion**
Inspecting the post section we can test for Local File Inclusion, a simple test confirms this.

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250919-1.png)

```
#passwd file
root:x:0:0:root:/root:/bin/bash
will:x:1000:1000:will:/home/will:/bin/bash
mat:x:1002:1002:,#,,:/home/mat:/bin/bash
toby:x:1003:1003:,,,:/home/toby:/bin/bash
ubuntu:x:1004:1005:Ubuntu:/home/ubuntu:/bin/bash
```
**Available users**
```
root
will
mat
toby
ubuntu
```
Now we access the protected path and we can see the contents of the text file

```bash
/var/www/html/secret_file_do_not_read.txt
```

Here, we retrieve the FTP credentials, and also take note on where are the files are being stored after uploading.

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250919-2.png)

```
ftpuser:givemefiles777
```
## Remote code Execution

using FTP upload a php reverse shell, we have the path where this will be placed /home/ftpuser/ftp/files, and after this we can access the reverse shell via the LFI vulnerability as follows.

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250919-3.png)

Got a reverse shell as www-data user

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250919-4.png)

Proceed to find the third flag: flag_3.txt

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250919-5.png)

flag_4.txt

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250919-6.png)
## Privilege Escalation

means user www-data can switch to user toby without password so, lets execute the following

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250919-7.png)

```
sudo -u toby bash
```

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250919-8.png)

Examining with pspy we can see user uid 1002 (mat) is executing a cronjob every minute.

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250920.png)

Proceed to edit the job cow.sh with a reverse shell and receive a reverse shell as mat.

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250920-1.png)

Get flag_5.txt here.

From mat user check sudo rights, we can see that a script can impersonate will, so let's dig into this.

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250920-2.png)

We have write permissions over cmd.py file, so edit this and include the following lines:

```python
import os
os.system("/bin/bash")
```

Resulting the final file like this.

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250920-3.png)

Now execute the script to escalate privileges

```bash
sudo -u will /usr/bin/python3 /home/mat/scripts/will_script.py "1"
```

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250920-4.png)

Inside /opt/backups we can see a key.b64 file, which contains a base64 data encoded.

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250920-5.png)
Decode the file and save it on you host machine, assign permissions to 0400.


![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250920-8.png)

Finally, use the private key to ssh into the system. Get the last flag here.

```bash
ssh -i key root@$IP
```

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250920-6.png)

![](/assets/img/posts/2025-09-20-thm-watcher-writeup/Watcher-20250920-7.png)
