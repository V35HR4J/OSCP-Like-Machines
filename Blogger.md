## Initial NMAP:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Blogger]
└─$ nmap 192.168.159.217 -T4
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-18 20:07 +0545
Nmap scan report for 192.168.159.217
Host is up (0.18s latency).
Not shown: 951 closed tcp ports (conn-refused), 47 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 16.97 seconds
```

Enemurating Port 80:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Blogger]
└─$ ffuf -u http://192.168.159.217/FUZZ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -c -ic -t 2000

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v1.3.1 Kali Exclusive <3
________________________________________________

 :: Method           : GET
 :: URL              : http://192.168.159.217/FUZZ
 :: Wordlist         : FUZZ: /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 2000
 :: Matcher          : Response status: 200,204,301,302,307,401,403,405
________________________________________________

                        [Status: 200, Size: 46199, Words: 21068, Lines: 986]
images                  [Status: 301, Size: 319, Words: 20, Lines: 10]
css                     [Status: 301, Size: 316, Words: 20, Lines: 10]
js                      [Status: 301, Size: 315, Words: 20, Lines: 10]
assets                  [Status: 301, Size: 319, Words: 20, Lines: 10]
                        [Status: 200, Size: 46199, Words: 21068, Lines: 986]
:: Progress: [87651/87651] :: Job [1/1] :: 846 req/sec :: Duration: [0:02:00] :: Errors: 3 ::
```

there's directory listing on `/assets` directory, checking inside `/assets/fonts` there's `blog` the site seems broken, checking the source code we can know that it's trying to load css and other files from blogger.thm, we need to add `blogger.thm` to `/etc/hosts` which makes the site useable. Now, doing so visiting `http://blogger.thm/assets/fonts/blog/` we can see a new wordpress site running with wappylyzer tool's detection, it's running wordpress 4.9.8, time to fire our wpscan :D

```bash

```

YET TO COMPLETE DUE TO PROVING GROUND TIME LIMITTT!! :(
    
# Takeaways:
- Try everything, i almost left checking out inside `/fonts` directory of `/assets` thinking it was a dead end, turns out it was the way to get initial access. (Maybe that's why they call it try harder :V )