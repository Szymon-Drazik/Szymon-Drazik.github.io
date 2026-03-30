---
title: "Write-Up: Skills Assessment - Password Attacks (PL/EN)"
date: 2026-03-30 20:44:00 +0200
categories: [Cybersecurity, Writeups]
tags: [active-directory, password-cracking, pass-the-hash, lsass, impacket, Hack-The-Box]
image:
  path: /assets/img/posts/SkillsAssessmentPasswordAttacks/netexec.png
  alt: Authentication attempt via NetExec.
---

W dzisiejszym wpisie przejdziemy przez proces pełnej infiltracji środowiska domenowego. Pokażę, jak zaczynając od skromnego dostępu z zewnątrz, poprzez umiejętny pivoting, łamanie sejfów haseł i zrzut pamięci LSASS, udało mi się zdobyć uprawnienia Administratora Domeny.

> 🇬🇧 **English speaker?** [Click here to skip straight to the English version!](#english-version)
{: .prompt-info }

Opisany atak to klasyczny przykład tzw. **"The Credential Theft Shuffle"** – systematycznego podejścia polegającego na przejmowaniu środowisk Active Directory poprzez wyciąganie poświadczeń z pamięci i wykorzystywanie ich do Lateral Movement.

## Scenariusz i Cel Zadania

Zadanie polegało na zinfiltrowaniu sieci firmy *Nexura LLC* i zdobyciu uprawnień pozwalających na wykonanie poleceń na kontrolerze domeny. 

**Punkt zaczepienia:** Wiedzieliśmy, że jedna z pracownic, Betty Jayde, ma tendencję do reużywania swojego prywatnego hasła (`Texas123!@#`) w różnych serwisach internetowych. Istniało spore prawdopodobieństwo, że używa go również w sieci korporacyjnej.

**Architektura i zakres:**
Mieliśmy do czynienia z klasyczną segmentacją sieci. Bezpośrednio z zewnątrz widoczna była tylko maszyna w strefie DMZ. Pozostałe hosty znajdowały się w sieci wewnętrznej, co wymagało skonfigurowania pivotingu.
* `DMZ01` – 10.129.*.* (Zewnętrzny) / 172.16.119.13 (Wewnętrzny)
* `JUMP01` – 172.16.119.7
* `FILE01` – 172.16.119.10
* `DC01` (Domain Controller) – 172.16.119.11

## 1. Rozpoznanie i początkowy dostęp

Standardowo rozpocząłem od skanowania portów za pomocą mojego autorskiego skryptu opartego na `nmap`, aby zidentyfikować wystawione usługi na maszynie brzegowej.

![Wynik skanowania nmap](/assets/img/posts/SkillsAssessmentPasswordAttacks/nmap.png)

Mając pewne punkty zaczepienia, zająłem się enumeracją użytkowników. Posiadając imię i nazwisko potencjalnego pracownika **Betty Jayde**, wykorzystałem narzędzie `username-anarchy`, aby wygenerować listę potencjalnych loginów korporacyjnych. 

![Generowanie loginów przez username-anarchy](/assets/img/posts/SkillsAssessmentPasswordAttacks/username-anarchy.png)

Mając listę loginów, użyłem `Hydry` do ataku słownikowego na wystawioną usługę SSH, wykorzystując znane nam hasło.

![Atak Hydra na SSH](/assets/img/posts/SkillsAssessmentPasswordAttacks/hydra.png)

Sukces! Udało się wyłuskać poprawne poświadczenia:
`jbetty:Texas123!@#`

## 2. Pivoting i wewnętrzna enumeracja

Po zalogowaniu się przez SSH na konto `jbetty` do maszyny DMZ01, moim kolejnym celem było rozeznanie się w sieci wewnętrznej. Skonfigurowałem dynamiczny port forwarding (pivoting) na port `9051`, aby uzyskać dostęp do odizolowanej podsieci `172.16.119.0/24`. 

Spróbowałem użyć zdobytych poświadczeń `jbetty` za pomocą narzędzia `NetExec` poprzez Proxychains przeciwko kontrolerowi domeny, a także wykonałem Password Spraying z pełną listą użytkowników, ale bez rezultatu.

![Próba uwierzytelnienia przez NetExec](/assets/img/posts/SkillsAssessmentPasswordAttacks/netexec.png)

Musiałem poszukać głębiej na już zdobytej maszynie. Przeszukując pliki w katalogu domowym użytkownika `jbetty`, natrafiłem na klasyczny błąd w pliku `.bash_history` czyli zapisane hasło w plain-text podczas logowania przez `sshpass`:

```text
sshpass -p "dealer-screwed-gym1" ssh hwilliam@file01
```
Mamy to! Nowe poświadczenia: `hwilliam:dealer-screwed-gym1`. Logowanie do maszyny `FILE01` przebiegło pomyślnie.

![Logowanie na konto hwilliam](/assets/img/posts/SkillsAssessmentPasswordAttacks/login.png)

## 3. Lateral Movement i łamanie Password Safe

Na maszynie `FILE01` zacząłem sprawdzać dostępne zasoby sieciowe i udziały (SMB shares).

![Przeszukiwanie zasobów na FILE01](/assets/img/posts/SkillsAssessmentPasswordAttacks/file01search.png)

W jednym z archiwów znalazłem stary plik bazy haseł: `Employee-Passwords_OLD.psafe3`. 

To idealny cel do ataku offline. Skopiowałem plik na swoją maszynę i przystąpiłem do łamania hasła głównego. Najpierw wyciągnąłem hash, a następnie użyłem narzędzia **John the Ripper** ze słownikiem `rockyou.txt`:

```zsh
pwsafe2john Employee-Passwords_OLD.psafe3 > psafe_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt psafe_hash.txt
```

Hasłem głównym do sejfu okazało się: `michaeljackson`.

Otworzyłem bazę za pomocą narzędzia **Password Gorilla**:

```zsh
password-gorilla Employee-Passwords_OLD.psafe3
```

![Zawartość pliku psafe3](/assets/img/posts/SkillsAssessmentPasswordAttacks/psafe3.png)

Wewnątrz znalazłem stare poświadczenia pracowników:

```text
jbetty:xiao-nicer-wheels5
bdavid:caramel-cigars-reply1
stom:fails-nibble-disturb4
hwilliam:warned-wobble-occur8
```

Ponieważ były to "stare" hasła, założyłem, że użytkownicy mogli po prostu inkrementować cyfry na końcu (bardzo częsta, zgubna praktyka w korporacjach). Przetestowałem warianty z końcówkami od `1` do `9`. Okazało się, że użytkownik `bdavid` w ogóle nie zmienił hasła, a co więcej posiadał on uprawnienia lokalnego administratora na maszynie `JMP01`!

## 4. Eskalacja uprawnień i przejęcie Domeny

Mając uprawnienia administratora na maszynie `JMP01`, wykonałem zrzut pamięci procesu **LSASS** (Local Security Authority Subsystem Service). Analiza zrzutu pozwoliła mi na wyciągnięcie kolejnych poświadczeń bezpośrednio z pamięci RAM:

`stom:calves-warp-learning1`

Czas sprawdzić, kim jest `stom`. Używając `proxychains` i narzędzia `NetExec` przeciwko kontrolerowi domeny (`172.16.119.11`), otrzymałem piękny status **"Pwn3d!"**:

```zsh
proxychains nxc smb 172.16.119.11 -u 'stom' -p 'calves-warp-learning1'
```

Użytkownik `stom` okazał się być Administratorem Domeny (`nexura.htb`)!

## 5. Post-Eksploatacja i dostęp RDP po hashu (Pass-the-Hash)

Mając kontrolę nad domeną, postanowiłem wydobyć najważniejszy łup, czyli hash NT wbudowanego konta **Administratora**. Użyłem do tego skryptu `secretsdump.py` z pakietu **Impacket**, aby przeprowadzić atak **DCSync**:

```zsh
proxychains python3 /usr/share/doc/python3-impacket/examples/secretsdump.py nexura.htb/stom:calves-warp-learning1@172.16.119.11 -just-dc-user Administrator
```

Zdobyty Hash NT Administratora: `36e09e1e6ade94d63fbcab5e5b8d6d23`

Dla pełnej satysfakcji chciałem zalogować się graficznie przez RDP, wykorzystując zjawisko **Pass-the-Hash**. Początkowo napotkałem jednak błąd:

![Błąd RDP](/assets/img/posts/SkillsAssessmentPasswordAttacks/errorrdp.png)

Aby to obejść, zalogowałem się najpierw na kontroler domeny za pomocą narzędzia **Evil-WinRM**, które natywnie obsługuje logowanie samym hashem:

```zsh
proxychains evil-winrm -i 172.16.119.11 -u Administrator -H 36e09e1e6ade94d63fbcab5e5b8d6d23
```

Będąc w sesji **PowerShell** na DC, zmieniłem klucz rejestru wyłączający tryb `RestrictedAdmin`. Pozwala to na uwierzytelnianie RDP za pomocą samego hasha NTLM, bez znajomości hasła w postaci jawnej:

```powershell
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

Po tej prostej zmianie mogłem swobodnie wejść przez RDP na kontroler domeny. Z ciekawości przepuściłem jeszcze hash Administratora przez narzędzie **Hashcat** z użyciem słownika `rockyou.txt`, ale nie został złamany. Jak jednak widać w środowisku Windows sam hash często jest równowartością hasła.

## Podsumowanie i Mitygacje

Ten łańcuch ataku doskonale ilustruje, dlaczego **Defense in Depth** jest tak ważna. Aby zapobiec atakom typu **Credential Theft Shuffle**, organizacja z tego scenariusza powinna wdrożyć następujące środki zabezpieczające:

* **LAPS (Local Administrator Password Solution):** Wdrożenie LAPS zapobiegłoby sytuacji, w której lokalne hasła administratorów są współdzielone lub łatwe do odgadnięcia, co znacząco utrudniłoby **Lateral Movement**.
* **Ochrona procesu LSASS:** Włączenie funkcji takich jak **RunAsPPL** (Protected Process Light) dla usługi LSA znacząco utrudniłoby zrzucenie pamięci i wydobycie haseł w czystym tekście.
* **Zasada najmniejszych uprawnień (PoLP) i Tiering:** Ograniczenie uprawnień administracyjnych. Konta o wysokich przywilejach (Domain Admin) nigdy nie powinny logować się na stacje robocze lub serwery niższych warstw (jak `JUMP01`), gdzie ich poświadczenia mogą zostać łatwo skradzione z pamięci.
* **Zarządzanie sekretami:** Edukacja użytkowników i administratorów. Hasła zapisane w otwartym tekście w plikach takich jak `.bash_history` czy w starych archiwach to wymarzony prezent dla każdego pentestera czy hakera.

---

> **⚠️ Zastrzeżenie i Informacja o zadaniu:**
> *Niniejszy artykuł jest autorskim opracowaniem typu **Write-up** z zadania **Skills Assessment - Password Attacks** dostępnego na platformie edukacyjnej [Hack The Box](https://www.hackthebox.com/). Zaprezentowany proces zdobycia domeny `nexura.htb` ma charakter wyłącznie edukacyjny.*
> 
> *Przedstawione techniki powinny być wykorzystywane tylko w ramach autoryzowanych testów penetracyjnych. Autor nie ponosi odpowiedzialności za jakiekolwiek niezgodne z prawem wykorzystanie wiedzy zawartej w tym wpisie.*

<br><br>
---
---
<br><br>

>## English Version {#english-version}

<details markdown="1">
<summary><b>Click here to expand the English version of this write-up!</b></summary>

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

</details>