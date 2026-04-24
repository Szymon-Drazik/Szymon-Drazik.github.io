---
title: "Write-Up: Skills Assessment - Using Web Proxies"
date: 2026-04-10 18:00:00 +0200
categories: [Cybersecurity, Writeups]
tags: [web-proxies, burp-suite, owasp-zap, fuzzing, metasploit, hack-the-box]
lang: pl
ref: web-proxies-skills-assessment
image:
  path: /assets/img/posts/SkillsAssessmentUsingWebProxies/fuzzing-results.png
  alt: Fuzzing hasha w OWASP ZAP.
---

# Zestaw narzędzi pentestera w praktyce: Fuzzing, Proxy i omijanie blokad

Będziemy omijać zabezpieczenia po stronie klienta, dekodować dane, pisać własne skrypty do fuzzingu i, co najciekawsze, zaglądać "pod maskę" Metasploita za pomocą Burp Suite i OWASP ZAP. Zaczynamy!

Rozwiązywanie maszyn CTF to nie tylko wiedza o podatnościach, ale też sprawne żonglowanie narzędziami. Często to samo zadanie można rozwiązać na kilka sposobów, a wybór odpowiedniego softu potrafi zaoszczędzić mnóstwo czasu. W dzisiejszym wpisie przejdziemy przez krótki, ale treściwy moduł podsumowujący umiejętności pracy z serwerami proxy na podstawie *Skills Assessment - Using Web Proxies*, w którym teoria ustępuje miejsca praktyce. 

---

## 1. Iluzja bezpieczeństwa po stronie klienta (Browser DevTools)

Pierwsze wyzwanie polegało na zdobyciu flagi ze strony `/lucky.php`. Haczyk? Przycisk na stronie był zablokowany i nie dało się go kliknąć. To klasyczny przykład zabezpieczenia po stronie klienta (*Client-Side Control*), które w świecie web-security tak naprawdę jest tylko wizualną ułudą.

Zamiast szukać skomplikowanych exploitów lub wysyłać spreparowane żądania przez proxy, najszybszą metodą było użycie wbudowanych **narzędzi deweloperskich przeglądarki** (F12).

Wystarczyło kliknąć zablokowany element prawym przyciskiem myszy, wybrać opcję "Zbadaj element" (Inspect) i zajrzeć prosto w kod HTML strony. Znajdował się tam atrybut `disabled`, który instruował przeglądarkę, aby nie pozwalała na interakcję.

![Kod HTML z podświetlonym atrybutem disabled](/assets/img/posts/SkillsAssessmentUsingWebProxies/disabled-button.png)
*Rys 1. Usunięcie atrybutu 'disabled' w narzędziach deweloperskich natychmiast odblokowuje przycisk.*

Usunięcie tego jednego słowa w locie z poziomu DevToolsów natychmiast uaktywniło przycisk na stronie. Jedno kliknięcie i na ekranie pojawiła się pierwsza flaga.

🚩 **Flaga 1:** `HTB{d154bl3d_bu770n5_w0n7_570p_m3}`

---

## 2. CyberChef i zagadka uciętego ciasteczka

Kolejny etap (`/admin.php`) wymagał przyjrzenia się nagłówkom HTTP, a dokładniej przypisanym nam ciasteczkom. Przechwycone ciasteczko wyglądało na bardzo długi ciąg znaków w formacie Hex, co sugerowało wielokrotne kodowanie.

Do szybkiej analizy takich ciągów w locie najlepiej sprawdza się **CyberChef**. Udało mi się ustalić odpowiedni "przepis" na dekodowanie: najpierw zdejmujemy format **Hex**, a następnie **Base64**. Wynikiem był następujący ciąg: `3dac93b8cd250aa8c1a36fffc79a17a`.

![Dekodowanie ciasteczka w CyberChef](/assets/img/posts/SkillsAssessmentUsingWebProxies/cyberchef-decode.png)
*Rys 2. Proces dekodowania: Hex -> Base64.*

I tutaj pojawił się kluczowy problem zadania. Otrzymany ciąg wyglądał jednoznacznie jak hash MD5, ale miał **31 znaków**. Standardowy hash MD5 ma ich zawsze dokładnie 32. Autor zadania celowo usunął ostatni znak, zmuszając nas do przeprowadzenia ataku brute-force (fuzzingu), aby go odgadnąć.

---

## 3. Fuzzing hybrydowy (Python + Web Proxy)

Strategia była prosta: wziąć 31-znakowy bazowy ciąg, doklejać do niego po kolei każdy możliwy znak alfanumeryczny (a-z, A-Z, 0-9), a następnie całe nowe 32-znakowe słowo **zakodować z powrotem** (najpierw Base64, potem Hex) i wysłać na serwer w ciasteczku.

Choć zaawansowane narzędzia proxy potrafią kodować payloady w locie (tzw. *Payload Processors*), czasami bywają kapryśne. W niektórych sytuacjach najlepiej polegać na rozwiązaniach elastycznych. Napisałem krótki skrypt w Pythonie, który wygenerował mi gotową listę 62 w pełni zakodowanych wariantów ciasteczka:

```python
import base64
import string

base_cookie = "3dac93b8cd250aa8c1a36fffc79a17a"
characters = string.ascii_letters + string.digits 

with open("payloads.txt", "w") as file:
    for char in characters:
        full_cookie = base_cookie + char
        b64_encoded = base64.b64encode(full_cookie.encode()).decode()
        hex_encoded = b64_encoded.encode().hex()
        file.write(hex_encoded + "\n")
```
Mając gotowy plik `payloads.txt`, załadowałem go do Fuzzera. Zadanie to można było rozwiązać zarówno w **OWASP ZAP** (wczytując listę jako typ *File*), jak i w **Burp Suite** przy pomocy narzędzia *Intruder* (atak typu *Sniper*). 

Po przepuszczeniu ataku wystarczyło posortować wyniki po rozmiarze odpowiedzi z serwera. Jedno żądanie wyraźnie odstawało rozmiarem od pozostałych 61 błędnych prób – to tam znajdowała się flaga.

![Wyniki ataku Fuzzing w proxy](/assets/img/posts/SkillsAssessmentUsingWebProxies/fuzzing-results.png)
*Rys 3. Fuzzing hasha w OWASP ZAP.*

🚩 **Flaga 2:** `HTB{burp_1n7rud3r_n1nj4!}`

---

## 4. Zagrożenie "Ślepego Skryptu": Metasploit + Proxy

W ostatnim etapie należało użyć modułu `auxiliary/scanner/http/coldfusion_locale_traversal` w środowisku **Metasploit Framework**. Czasami jednak automatyczne skanery zawodzą, a odpalanie skryptów bez wglądu w to, co dokładnie wysyłają, to ślepa uliczka. 

Aby zdiagnozować problem i odpowiedzieć na pytanie z zadania (jakiego ukrytego katalogu szuka Metasploit?), postanowiłem przepuścić ruch z MSF przez moje lokalne proxy. Zgodnie z zasadą *"gdy jedno narzędzie strajkuje, odpalaj drugie"*, przerzuciłem się na **Burp Suite**, który bezbłędnie przechwycił komunikację.

W konsoli `msfconsole` wystarczyło ustawić odpowiednią zmienną:

```bash
msf> set Proxies HTTP:127.0.0.1:8080
msf> run
```
Po wykonaniu modułu, w historii HTTP w Burpie pojawiło się surowe żądanie GET. Zobaczyłem w nim dokładnie strukturę ataku *Directory Traversal* oraz ścieżkę, o którą pytało zadanie.

![Przechwycone żądanie z Metasploita w Burp Suite](/assets/img/posts/SkillsAssessmentUsingWebProxies/metasploit-request.png)
*Rys 4. Analiza surowego żądania wysłanego przez automat.*

🚩 **Odpowiedź:** `CFIDE`

## Podsumowanie i wnioski narzędziowe

Ten krótki zestaw zadań to świetny poligon doświadczalny pokazujący, że w cyberbezpieczeństwie zrozumienie narządzi jest ważniejsze niż uczenie się exploitów na pamięć. W podobnych, rzeczywistych scenariuszach testów penetracyjnych te same narzędzia pozwalają błyskawicznie zidentyfikować luki:

* **Browser DevTools:** Zawsze weryfikuj kod źródłowy. Wszelkie blokady typu `disabled`, `hidden` czy ograniczenia długości znaków na froncie to zazwyczaj tylko iluzja bezpieczeństwa, którą można łatwo obejść.
* **CyberChef + Python:** To idealny duet do wykrywania anomalii kryptograficznych. Jeśli widzisz dziwny ciąg znaków (np. ucięty hash MD5), kilka linijek skryptu wystarczy, aby zautomatyzować dekodowanie i fuzzing, zamiast robić to ręcznie.
* **OWASP ZAP / Burp Suite:** To Twoje oczy i uszy. Nigdy nie ufaj skanerom "w ciemno". Przepuszczanie ruchu z automatów (jak Metasploit) przez proxy pozwala szybko wykryć, czy payload faktycznie uderza we właściwe ścieżki (jak `/CFIDE/`), czy też gubi się po drodze.

Dobry pentester nie przywiązuje się kurczowo do jednego rozwiązania. Tam, gdzie jedno proxy nie radzi sobie z nietypowym ruchem, płynnie przesiadamy się na drugie. Elastyczność i umiejętność analizy surowego ruchu to absolutny fundament w naszym fachu!

> **⚠️ Zastrzeżenie i Informacja o zadaniu:**
> *Niniejszy artykuł jest autorskim opracowaniem typu **Write-up** z zadania **Skills Assessment - Using Web Proxies** dostępnego na platformie edukacyjnej [Hack The Box](https://www.hackthebox.com/). Zaprezentowany proces analizy zabezpieczeń ma charakter wyłącznie edukacyjny.*
> 
> *Przedstawione techniki powinny być wykorzystywane tylko w ramach autoryzowanych testów penetracyjnych za wyraźną zgodą właściciela infrastruktury. Autor nie ponosi odpowiedzialności za jakiekolwiek niezgodne z prawem wykorzystanie wiedzy zawartej w tym wpisie.*
