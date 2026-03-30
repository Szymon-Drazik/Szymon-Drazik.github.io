---
title: "Write-Up Skills Assessment - Password Attacks"
date: 2026-03-30 20:44:00 +0200
categories: [Cybersecurity, Writeups]
tags: [active-directory, password-cracking, pass-the-hash, lsass, impacket, Hack-The-Box]
image:
  path: /assets/img/posts/SkillsAssessmentPasswordAttacks/netexec.png
  alt: Próba uwierzytelnienia przez NetExec.
---

W dzisiejszym wpisie przejdziemy przez proces kompromitacji środowiska domenowego. Pokażę, jak zaczynając od skromnego dostępu z zewnątrz, poprzez umiejętny pivoting, łamanie sejfów haseł i zrzut pamięci LSASS, udało mi się zdobyć uprawnienia Administratora Domeny. 

Opisany atak to klasyczny przykład tzw. **"The Credential Theft Shuffle"** to systematycznego podejścia polegającego na przejmowaniu środowisk Active Directory poprzez wyciąganie poświadczeń z pamięci i wykorzystywanie ich do Lateral Movement.

## Scenariusz i Cel Zadania

Zadanie polegało na zinfiltrowaniu sieci firmy *Nexura LLC* i zdobyciu uprawnień pozwalających na wykonanie poleceń na kontrolerze domeny. 

**Punkt zaczepienia:** Wiedzieliśmy, że jedna z pracownic, Betty Jayde, ma tendencję do reużywania swojego prywatnego hasła (`Texas123!@#`) w różnych serwisach internetowych. Istniało spore prawdopodobieństwo, że używa go również w sieci korporacyjnej.

**Architektura i zakres:**
Mieliśmy do czynienia z klasyczną segmentacją sieci. Bezpośrednio z zewnątrz widoczna była tylko maszyna w strefie DMZ. Pozostałe hosty znajdowały się w sieci wewnętrznej, co wymagało skonfigurowania pivotingu.
* `DMZ01` – 10.129.*.* (Zewnętrzny) / 172.16.119.13 (Wewnętrzny)
* `JUMP01` – 172.16.119.7
* `FILE01` – 172.16.119.10
* `DC01` (Domain Controller) – 172.16.119.11

---

## 1. Rozpoznanie i początkowy dostęp

Standardowo rozpocząłem od skanowania portów za pomocą mojego autorskiego skryptu opartego na `nmap`, aby zidentyfikować wystawione usługi na maszynie brzegowej.

![Wynik skanowania nmap](/assets/img/posts/SkillsAssessmentPasswordAttacks/nmap.png)

Mając pewne punkty zaczepienia, zająłem się enumeracją użytkowników. Posiadając imię i nazwisko potencjalnego pracownika **Betty Jayde**. Wykorzystałem do tego narzędzie `username-anarchy`, aby wygenerować listę potencjalnych loginów korporacyjnych. 

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

Ponieważ były to "stare" hasła, założyłem, że użytkownicy mogli po prostu inkrementować cyfry na końcu to bardzo częsta, zgubna praktyka w korporacjach. Przetestowałem warianty z końcówkami od `1` do `9`. Okazało się, że użytkownik `bdavid` w ogóle nie zmienił hasła, a co więcej posiadał on uprawnienia lokalnego administratora na maszynie `JMP01`!

## 4. Eskalacja uprawnień i przejęcie Domeny

Mając uprawnienia administratora na maszynie `JMP01`, wykonałem zrzut pamięci procesu **LSASS** (Local Security Authority Subsystem Service). Analiza zrzutu pozwoliła mi na wyciągnięcie kolejnych poświadczeń bezpośrednio z pamięci RAM:

`stom:calves-warp-learning1`

Czas sprawdzić, kim jest `stom`. Używając `proxychains` i narzędzia `NetExec` przeciwko kontrolerowi domeny (`172.16.119.11`), otrzymałem piękny status **"Pwn3d!"**:

```zsh
proxychains nxc smb 172.16.119.11 -u 'stom' -p 'calves-warp-learning1'
```

Użytkownik `stom` okazał się być Administratorem Domeny (`nexura.htb`)!

## 5. Post-Eksploatacja i dostęp RDP po hashu (Pass-the-Hash)

Mając kontrolę nad domeną, postanowiłem wydobyć najważniejszy łup hash NT wbudowanego konta **Administratora**. Użyłem do tego skryptu `secretsdump.py` z pakietu **Impacket**, aby przeprowadzić atak **DCSync**:

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
> *Przedstawione techniki (m.in. zrzut pamięci LSASS, Pass-the-Hash, łamanie baz Password Safe) powinny być wykorzystywane tylko w ramach autoryzowanych testów penetracyjnych lub w bezpiecznych środowiskach laboratoryjnych. Autor nie ponosi odpowiedzialności za jakiekolwiek niezgodne z prawem wykorzystanie wiedzy zawartej w tym wpisie. Pamiętaj: nieautoryzowany dostęp do systemów informatycznych jest przestępstwem.*