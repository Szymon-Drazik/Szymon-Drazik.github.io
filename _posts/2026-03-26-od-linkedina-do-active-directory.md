---
title: "Od LinkedIna do Active Directory: Jak przestępcy zgadują Twój login?"
date: 2026-03-23 18:00:00 +0100
categories: [Cybersecurity, Red Teaming]
tags: [osint, active-directory, kerbrute, pentesting, username-anarchy]
image:
  path: /assets/img/posts/kerbrute_username/Kerbrute.png
  alt: Konsola z wynikiem działania narzędzia Kerbrute weryfikującego loginy.
---
# Jak przestępcy zgadują Twój login? Od LinkedIna do Active Directory

W świecie cyberbezpieczeństwa często skupiamy się na zaawansowanych exploitach i lukach 0-day. Tymczasem w realnych atakach i podczas testów penetracyjnych , najtrudniejszy bywa pierwszy krok: zdobycie poprawnego loginu do firmowej sieci. 

W tym wpisie pokażę, jak z połączenia otwartych źródeł i mechanizmów protokołu Kerberos można zbudować gotową listę celów do ataku, i to bez blokowania kont w Active Directory.

## Kopalnia wiedzy na LinkedIn

Zanim atakujący dotknie firmowej infrastruktury, zaczyna od białego wywiadu (OSINT). Najlepszym źródłem informacji o strukturze zatrudnienia jest LinkedIn. 

Wyszukując profil docelowej firmy (na potrzeby tego wpisu użyjemy fikcyjnego *Inlanefreight*), atakujący przechodzi do zakładki "Osoby". Zbiera stamtąd imiona i nazwiska pracowników. Często używa się do tego automatycznych scraperów. Mamy bazę prawdziwych nazwisk i co dalej?

## Generowanie formatów z Username-anarchy

Mamy już listę pracowników, na której znajdują się na przykład: *John Marston*, *Carol Johnson* i *Jennifer Stapleton*. Jak jednak wygląda ich login w domenie? Czy będzie to `john.marston`, `cjohnson`, a może `stapletonj`?

Zamiast zgadywać w ciemno, używamy narzędzia **username-anarchy**. Pozwala ono na wygenerowanie potencjalnych formatów loginów na podstawie naszej listy. Przepuszczamy zebrane dane (zapisane w pliku `names.txt`) przez narzędzie i otrzymujemy plik tekstowy z kombinacjami:

![Wynik działania username-anarchy - wygenerowana lista potencjalnych loginów](/assets/img/posts/kerbrute_username/username-anarchy.png)
*Username-anarchy potrafi wygenerować wszystkie najpopularniejsze schematy nazewnictwa w kilka sekund.*

## Weryfikacja bezszelestna, czyli Kerbrute w akcji

Mamy plik z tysiącami loginów. Teraz trzeba sprawdzić, które z nich faktycznie istnieją w firmowym Active Directory. Próba logowania się na każdy z nich przez panel WWW czy usługę RDP zostawiłaby ogromny ślad w logach i szybko zablokowała konta (*Account Lockout Policy*).

Tutaj na scenę wkracza **Kerbrute**.

Uruchomienie narzędzia w trybie enumeracji użytkowników `userenum` jest bardzo proste. Podajemy adres IP kontrolera domeny `--dc`, nazwę domeny (w naszym przypadku `ILF.local`) oraz ścieżkę do wygenerowanej wcześniej listy loginów:

```bash
$ ./kerbrute_linux_amd64 userenum --dc 10.129.202.85 --domain ILF.local ~/username.txt 
```
Narzędzie to komunikuje się bezpośrednio z usługą Kerberos na kontrolerze domeny. Wysyła żądanie `AS-REQ` (Authentication Service Request) **bez pre-autentykacji** czyli bez podawania hasła. 

Jak to działa w praktyce?
* Jeśli login **nie istnieje**, serwer zwraca błąd: `KDC_ERR_C_PRINCIPAL_UNKNOWN`.
* Jeśli login **istnieje**, serwer prosi o hasło, zwracając: `KDC_ERR_PREAUTH_REQUIRED`.

Dzięki temu atakujący "czyta" z błędów serwera, które konta są aktywne, nie wysyłając ani jednego błędnego hasła!

![Wynik działania Kerbrute - znalezione poprawne loginy w domenie](/assets/img/posts/kerbrute_username/Kerbrute.png)
*Kerbrute bezbłędnie i "cicho" weryfikuje poprawne loginy bazując na specyficznych odpowiedziach Kerberosa.*

## Jak obrońcy (Blue Team) mogą to wykryć?

Atak wydaje się niewidzialny, ale zostawia ślady, jeśli wiesz, gdzie szukać:
1. **Monitorowanie logów (Event ID 4768):** Żądanie biletu TGT przez Kerbrute generuje logi na kontrolerze domeny. Złota zasada: jeśli widzisz setki zdarzeń `Event ID 4768` z jednego adresu IP w ułamku sekundy, to znaczy, że ktoś prawdopodobnie przeprowadza enumerację. Warto ustawić odpowiednie alerty w systemie SIEM.
2. **Polityka haseł i Password Spraying:** Enumeracja to dopiero początek. Kolejnym krokiem atakującego będzie *Password Spraying* czyli testowanie jednego popularnego hasła. Silna polityka haseł i MFA (Multi-Factor Authentication) to podstawa.
3. **Świadomość OSINT:** Warto edukować pracowników, by ograniczali widoczność swoich profili w social mediach dla osób spoza sieci kontaktów.

## Użyte narzędzia

Jeśli chcesz zgłębić temat i przetestować te mechanizmy we własnym środowisku testowym, poniżej znajdziesz linki do projektów omówionych w artykule:

* **[username-anarchy](https://github.com/urbanadventurer/username-anarchy)**: Świetne narzędzie napisane w języku Ruby do generowania list potencjalnych nazw użytkowników. Posiada wbudowane reguły obsługujące formaty loginów z ponad 40 000 różnych forów, systemów i korporacji. Wystarczy podać listę imion i nazwisk, a skrypt zrobi resztę.
* **[Kerbrute](https://github.com/ropnop/kerbrute)**: Niezwykle szybkie narzędzie napisane w Go, stworzone specjalnie do interakcji z protokołem Kerberos w Active Directory. W przeciwieństwie do innych narzędzi brute-force, Kerbrute nie wykorzystuje zapytań SMB, NTLM czy LDAP, ale natywną pre-autentykację Kerberosa, co czyni go szybszym i trudniejszym do wykrycia.

## Podsumowanie

Łączenie prostych narzędzi i technik tworzy groźne scenariusze. To, co zaczyna się jako niewinne przeglądanie LinkedIna, przy użyciu `username-anarchy` i weryfikacji przez `Kerbrute`, szybko zamienia się w precyzyjną listę celów. Zrozumienie tego łańcucha ataków to pierwszy krok do budowania skutecznej obrony.

---

> **⚠️ Zastrzeżenie:**
> *Niniejszy artykuł ma charakter wyłącznie edukacyjny i demonstracyjny. Zaprezentowane techniki oraz narzędzia (takie jak Kerbrute czy username-anarchy) powinny być wykorzystywane tylko i wyłącznie w ramach autoryzowanych testów penetracyjnych, legalnych działań audytowych lub w bezpiecznych, własnych środowiskach laboratoryjnych. Nieautoryzowany dostęp do systemów informatycznych, infrastruktury firmowej oraz łamanie zabezpieczeń jest przestępstwem. Autor nie ponosi odpowiedzialności za jakiekolwiek niezgodne z prawem wykorzystanie wiedzy zawartej w tym wpisie.*