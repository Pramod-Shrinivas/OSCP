---
description: 'OSCP Preparation: TJNULL List —Kioptrix Level 1 Walkthrough'
---

# Kioptrix Level 1 w/o Metasploit

Kioptrix Series on Vulnhub: [Kioptrix](https://www.vulnhub.com/?q=kioptrix) consists of five machines, each with increasing difficulty. The challenges are somewhat outdated, yet seem pertinent.

Let’s get started with the first box. I downloaded the .rar file, and spun up the Virtual Machine in [Virtual Box](https://www.virtualbox.org/). For the attacker machine, i will be using [Kali Linux](https://www.kali.org/get-kali/#kali-platforms)

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*9c3jFN07WvV9xansTShnFg.png" alt="" height="456" width="700"><figcaption></figcaption></figure>

**Scanning and Enumeration Phase:**

Attacker Machine: Kali Machine IP: 192.168.1.7

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*Rd7KQzbPi7n4VrBwQuccJw.png" alt="" height="84" width="700"><figcaption></figcaption></figure>

Finding the Target IP Address (Kioptrix VM). The host grabs an address via DHCP as it boots. We can detect the IP Address either via [netdiscover](https://www.kali.org/tools/netdiscover/) reconnaissance tool or any network scanner tools like [Nmap](https://nmap.org/) to identify live hosts.

Netdiscover Command:

```
sudo netdiscover -i eth0 -r 192.168.1.0/24
                  or
nmap -T4 -sn 192.168.1.0/24
```

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*H_in2ku2Bs7k_HiYdyRBDQ.png" alt="" height="257" width="700"><figcaption></figcaption></figure>

Mac Vendor: “_PCS Systemtechnik GmbH” belongs to VirtualBox. Here 192.168.1.8 belongs to our machine “Kioptrix Level 1”._

**Service Enumeration:** Detailed scanning of the target with Nmap.

```
sudo nmap -sV -sC -Pn -sS -A 192.168.1.8
```

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*YVDORPcj-JMLbv0reCJW9Q.png" alt="" height="604" width="700"><figcaption></figcaption></figure>

Interesting services worth exploring based on above scan results.

* Port 22 — Running OpenSSH 2.9p2.
* Port 80 — Running Apache Web server (1.3.20, Red-Hat/Linux; mod\_ssl/2.8.4).
* Port 111 — Running RPC.
* Port 139 — Running Samba.
* Port 443 — Running Apache Web server (1.3.20, Red-Hat/Linux; mod\_ssl/2.8.4).
* Port 1024 — Running RPC.
* Port 32768 — Unknown.

**Exploitation 1:** Apache and mod\_ssl

Lets explore the application hosted.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*a3v83IGweHfroKiCSA_GIQ.png" alt="" height="461" width="700"><figcaption></figcaption></figure>

Ran Dirbuster to reveal any hidden directories. Nothing of interest except a standard apache installation.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*btvfxbJE3VbrKc3UNo7jIA.png" alt="" height="548" width="700"><figcaption></figcaption></figure>

Ran Nikto against the web server to reveal any vulnerabilities. Nikto revealed a few outdated versions of software running including outdated mod\_ssl version vulnerable to remote buffer overflow.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*uvY1g-36CZUnB4FXj7f7EA.png" alt="" height="623" width="700"><figcaption></figcaption></figure>

Searching for Apache and mod\_ssl vulnerabilities / exploits confirms the above.

```
searchsploit apache mod_ssl
```

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*uBhZOa_jxQO478tgZsAxkQ.png" alt="" height="71" width="700"><figcaption></figcaption></figure>

Create a local copy of the exploit 47080.c as exploit.c. Tried to compile the exploit but failed.

<figure><img src="https://miro.medium.com/v2/resize:fit:573/1*8bvqiyomktb8Aha7vcCNxQ.png" alt="" height="56" width="573"><figcaption></figcaption></figure>

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*cXhbF2BnFjsS-oqjzspC8A.png" alt="" height="405" width="700"><figcaption></figcaption></figure>

Found an interesting article by Paul Sec about [updating this exploit.](https://paulsec.github.io/posts/updating-openfuck-exploit/) Validated the exploit as described. (Warnings can be ignored !)

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*3OLcwEO0YHGTIXhEKStv4g.png" alt="" height="558" width="700"><figcaption></figcaption></figure>

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*JvZKqqcub265kxcOJTDgdA.png" alt="" height="308" width="700"><figcaption></figcaption></figure>

Launched the exploit successfully. Was able to login as root.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*KE07MhRwD3HGVhuE8I_08w.png" alt="" height="499" width="700"><figcaption></figcaption></figure>

Finally found the flag (or rather email from root) in var/mail/root.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*QuXFBlI5l9BNwd9lFV-Vbg.png" alt="" height="371" width="700"><figcaption></figcaption></figure>

**Exploitation 2:** SMB Service

Enumerating SMB identified from the nmap scans earlier (port 139) on our target machine. Scans show that null session is allowed on the host. (Anonymous Login)

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*gvACINL0MMyH7EzRH6eOqA.png" alt="" height="462" width="700"><figcaption></figcaption></figure>

Enumerating SMB by connecting to the target machine (Anonymous Login) to list all the shares available.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*g83tuKuwGQrZieHwjxj19w.png" alt="" height="234" width="700"><figcaption></figcaption></figure>

Let’s check if IPC$ and ADMIN$ shares can be accessed using [smbclient](https://www.kali.org/tools/samba/#libsmbclient).

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*hno8XDiFbk4aPqMivd7blQ.png" alt="" height="385" width="700"><figcaption></figcaption></figure>

Admin shares are accessible. File listing and other permissions are denied. Lets identify the smbversion running on the target machine. Tried with the below but no success. Perhaps there is a different way to identify ? Was trying to avoid metasploit as such as possible !

```
nmap -sV -p 139 192.168.1.8
enum4linux -a 192.168.1.8
smbmap -H 192.168.1.8
```

Last resort -> Using auxiliary module in metasploit for smbversion enumeration :(

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*qC4NvRPdOHxnx0PCxZyBBw.png" alt="" height="430" width="700"><figcaption></figcaption></figure>

Lets search for any exploits available using searchsploit. Remote Code Execution seems interesting. (Avoiding metasploit usage here!)

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*HresporoDntw7sgLtltBFw.png" alt="" height="128" width="700"><figcaption></figcaption></figure>

Copying a local copy of the exploit, compiling and executing the exploit.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*DSmAPulscXWUkZq5jwXH1w.png" alt="" height="152" width="700"><figcaption></figcaption></figure>

Finally was successful in getting root access to the machine.

<figure><img src="https://miro.medium.com/v2/resize:fit:700/1*sB8grAbqqiuse9pSBIhnEA.png" alt="" height="518" width="700"><figcaption></figcaption></figure>

Interesting caption observed. “JE MOET JE MUIL HOUWE” which means “YOU MUST SHUT YOUR MOUTH” and an reference to a famous [neophyte song ](https://www.youtube.com/watch?v=BcpirI2BcWI).
