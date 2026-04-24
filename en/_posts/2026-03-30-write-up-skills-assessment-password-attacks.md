---
layout: post
title: "Write-Up: Skills Assessment - Password Attacks"
date: 2026-03-30 20:44:00 +0200
categories: [Cybersecurity, Writeups]
tags: [active-directory, password-cracking, pass-the-hash, lsass, impacket, Hack-The-Box]
lang: en
ref: password-attacks-skills-assessment
image:
  path: /assets/img/posts/SkillsAssessmentPasswordAttacks/netexec.png
  alt: Authentication attempt via NetExec.
---

In today's post, we will walk through the process of compromising a domain environment. I will demonstrate how, starting from modest external access, through skillful pivoting, cracking password safes, and dumping LSASS memory, I managed to obtain Domain Administrator privileges.

The described attack is a classic example of the **"Credential Theft Shuffle"** a systematic approach to taking over Active Directory environments by extracting credentials from memory and utilizing them for Lateral Movement.

### Scenario and Objective

The objective was to infiltrate the *Nexura LLC* network and gain privileges that allow command execution on the Domain Controller.

**Foothold:** We knew that one of the employees, Betty Jayde, tended to reuse her private password (`Texas123!@#`) across various online services. There was a high probability she was also using it within the corporate network.

**Architecture and Scope:**
We were dealing with classic network segmentation. Only the machine in the DMZ zone was visible from the outside. The remaining hosts were located in the internal network, requiring the configuration of pivoting.
* `DMZ01` – 10.129.*.* (External) / 172.16.119.13 (Internal)
* `JUMP01` – 172.16.119.7
* `FILE01` – 172.16.119.10
* `DC01` (Domain Controller) – 172.16.119.11

### 1. Reconnaissance & Initial Access

I started as usual by scanning ports using my custom `nmap` based script to identify exposed services on the edge machine.

![Nmap scan results](/assets/img/posts/SkillsAssessmentPasswordAttacks/nmap.png)

With some initial foothold points, I moved on to user enumeration. Having the first and last name of a potential employee **Betty Jayde** I used the `username-anarchy` tool to generate a list of potential corporate usernames.

![Generating usernames via username-anarchy](/assets/img/posts/SkillsAssessmentPasswordAttacks/username-anarchy.png)

Once I had the list of usernames, I used `Hydra` for a dictionary attack against the exposed SSH service, utilizing the known password.

![Hydra attack on SSH](/assets/img/posts/SkillsAssessmentPasswordAttacks/hydra.png)

Success! I managed to extract valid credentials:
`jbetty:Texas123!@#`

### 2. Pivoting & Internal Enumeration

After logging into the `DMZ01` machine via SSH using the `jbetty` account, my next goal was to map out the internal network. I configured dynamic port forwarding (pivoting) on port `9051` to access the isolated `172.16.119.0/24` subnet.

I attempted to use the obtained `jbetty` credentials with the `NetExec` tool via Proxychains against the Domain Controller, and I also performed Password Spraying with the full user list, but to no avail.

![Authentication attempt via NetExec](/assets/img/posts/SkillsAssessmentPasswordAttacks/netexec.png)

I had to dig deeper on the already compromised machine. Searching through the files in the home directory of user `jbetty`, I came across a classic mistake in the `.bash_history` file: a password saved in plain-text during an `sshpass` login:

```text
sshpass -p "dealer-screwed-gym1" ssh hwilliam@file01
```
We got it! New credentials: `hwilliam:dealer-screwed-gym1`. Logging into the `FILE01` machine was successful.

![Logging into the hwilliam account](/assets/img/posts/SkillsAssessmentPasswordAttacks/login.png)

### 3. Lateral Movement & Cracking Password Safe

On the `FILE01` machine, I started checking available network resources and SMB shares.

![Searching resources on FILE01](/assets/img/posts/SkillsAssessmentPasswordAttacks/file01search.png)

In one of the archives, I found an old password database file: `Employee-Passwords_OLD.psafe3`.

This was a perfect target for an offline attack. I copied the file to my machine and proceeded to crack the master password. First, I extracted the hash, and then used **John the Ripper** with the `rockyou.txt` dictionary:

```zsh
pwsafe2john Employee-Passwords_OLD.psafe3 > psafe_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt psafe_hash.txt
```

The master password for the safe turned out to be: `michaeljackson`.

I opened the database using the **Password Gorilla** tool:

```zsh
password-gorilla Employee-Passwords_OLD.psafe3
```

![Content of the psafe3 file](/assets/img/posts/SkillsAssessmentPasswordAttacks/psafe3.png)

Inside, I found old employee credentials:

```text
jbetty:xiao-nicer-wheels5
bdavid:caramel-cigars-reply1
stom:fails-nibble-disturb4
hwilliam:warned-wobble-occur8
```

Since these were "old" passwords, I assumed that users might have simply incremented the digits at the end—a very common and dangerous practice in corporate environments. I tested variants with endings from `1` to `9`. It turned out that the user `bdavid` had not changed his password at all, and moreover, he possessed local administrator privileges on the `JMP01` machine!

### 4. Privilege Escalation & Domain Takeover

Having administrator privileges on the `JMP01` machine, I performed a memory dump of the **LSASS** (Local Security Authority Subsystem Service) process. Analyzing the dump allowed me to extract more credentials directly from RAM:

`stom:calves-warp-learning1`

Time to check who `stom` is. Using `proxychains` and the `NetExec` tool against the Domain Controller (`172.16.119.11`), I received the beautiful **"Pwn3d!"** status:

```zsh
proxychains nxc smb 172.16.119.11 -u 'stom' -p 'calves-warp-learning1'
```

User `stom` turned out to be a Domain Administrator (`nexura.htb`)!

### 5. Post-Exploitation & RDP Access via Hash (Pass-the-Hash)

With control over the domain, I decided to extract the ultimate prize—the NT hash of the built-in **Administrator** account. I used the `secretsdump.py` script from the **Impacket** suite to conduct a **DCSync** attack:

```zsh
proxychains python3 /usr/share/doc/python3-impacket/examples/secretsdump.py nexura.htb/stom:calves-warp-learning1@172.16.119.11 -just-dc-user Administrator
```

Obtained Administrator NT Hash: `36e09e1e6ade94d63fbcab5e5b8d6d23`

For full satisfaction, I wanted to log in graphically via RDP using the **Pass-the-Hash** technique. Initially, however, I encountered an error:

![RDP Error](/assets/img/posts/SkillsAssessmentPasswordAttacks/errorrdp.png)

To bypass this, I first logged into the Domain Controller using the **Evil-WinRM** tool, which natively supports logging in with just the hash:

```zsh
proxychains evil-winrm -i 172.16.119.11 -u Administrator -H 36e09e1e6ade94d63fbcab5e5b8d6d23
```

While in a **PowerShell** session on the DC, I changed the registry key that disables `RestrictedAdmin` mode. This allows RDP authentication using the NTLM hash itself, without needing the clear-text password:

```powershell
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

After this simple change, I could freely RDP into the Domain Controller. Out of curiosity, I also ran the Administrator's hash through the **Hashcat** tool using the `rockyou.txt` dictionary, but it was not cracked. As shown, however, in a Windows environment, the hash itself is often the equivalent of a password.

### Summary & Mitigations

This attack chain perfectly illustrates why **Defense in Depth** is so vital. To prevent **Credential Theft Shuffle** attacks, the organization in this scenario should implement the following security measures:

* **LAPS (Local Administrator Password Solution):** Implementing LAPS would prevent scenarios where local administrator passwords are shared or easily guessed, significantly hindering **Lateral Movement**.
* **LSASS Process Protection:** Enabling features such as **RunAsPPL** (Protected Process Light) for the LSA service would make dumping memory and extracting clear-text passwords significantly more difficult.
* **Principle of Least Privilege (PoLP) and Tiering:** Restricting administrative privileges. High-privileged accounts (Domain Admin) should never log into lower-tier workstations or servers (like `JUMP01`), where their credentials can be easily stolen from memory.
* **Secrets Management:** Education for users and administrators. Plain-text passwords saved in files like `.bash_history` or old archives are a dream gift for any pentester.

---

> **⚠️ Disclaimer & Task Information:**
> *This article is an original **Write-up** of the **Skills Assessment - Password Attacks** task available on the [Hack The Box](https://www.hackthebox.com/) educational platform. The presented process of gaining control over the `nexura.htb` domain is for educational purposes only.*
> 
> *The techniques presented should only be used as part of authorized penetration tests. The author is not responsible for any illegal use of the knowledge contained in this post.*