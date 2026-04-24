---
title: "Write-Up: Skills Assessment - Using Web Proxies"
date: 2026-04-10 18:00:00 +0200
categories: [Cybersecurity, Writeups]
tags: [web-proxies, burp-suite, owasp-zap, fuzzing, metasploit, hack-the-box]
image:
  path: /assets/img/posts/SkillsAssessmentUsingWebProxies/fuzzing-results.png
  alt: Fuzzing hasha w OWASP ZAP.
---

# Zestaw narzędzi pentestera w praktyce: Fuzzing, Proxy i omijanie blokad

Będziemy omijać zabezpieczenia po stronie klienta, dekodować dane, pisać własne skrypty do fuzzingu i, co najciekawsze, zaglądać "pod maskę" Metasploita za pomocą Burp Suite i OWASP ZAP. Zaczynamy!

>**English speaker?** [Click here to skip straight to the English version!](#english-version)
{: .prompt-info }

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

---

## English Version {#english-version}
<details markdown="1">
<summary><b>Click here to expand the English version of this write-up!</b></summary>

# A Pentester's Toolkit in Practice: Fuzzing, Proxies, and Bypassing Restrictions

We will bypass client-side security measures, decode data, write custom fuzzing scripts, and, most interestingly, peek "under the hood" of Metasploit using Burp Suite and OWASP ZAP. Let's get started!

Solving CTF machines is not just about vulnerability knowledge, but also about efficiently juggling tools. Often, the same task can be solved in multiple ways, and choosing the right software can save a lot of time. In today's post, we will go through a short but comprehensive module summarizing web proxy skills based on *Skills Assessment - Using Web Proxies*, where theory gives way to practice.

---

## 1. The Illusion of Client-Side Security (Browser DevTools)

The first challenge was to grab the flag from the `/lucky.php` page. The catch? The button on the page was disabled and couldn't be clicked. This is a classic example of *Client-Side Control*, which in the web-security world is really just a visual illusion.

Instead of looking for complex exploits or sending crafted requests via a proxy, the fastest method was to use the browser's built-in **Developer Tools** (F12).

All it took was right-clicking the disabled element, selecting "Inspect" (or "Inspect Element"), and looking straight into the page's HTML code. There was a `disabled` attribute instructing the browser not to allow interaction.

![HTML code with highlighted disabled attribute](/assets/img/posts/SkillsAssessmentUsingWebProxies/disabled-button.png)
*Fig 1. Removing the 'disabled' attribute in DevTools immediately unlocks the button.*

Removing this single word on the fly from within DevTools instantly activated the button on the page. One click and the first flag appeared on the screen.

🚩 **Flag 1:** `HTB{d154bl3d_bu770n5_w0n7_570p_m3}`

---

## 2. CyberChef and the Mystery of the Truncated Cookie

The next stage (`/admin.php`) required looking at the HTTP headers, specifically the cookies assigned to us. The intercepted cookie looked like a very long string of characters in Hex format, suggesting multiple layers of encoding.

For quick, on-the-fly analysis of such strings, **CyberChef** is the best tool. I managed to figure out the right "recipe" for decoding: first, we strip the **Hex** format, and then **Base64**. The result was the following string: `3dac93b8cd250aa8c1a36fffc79a17a`.

![Decoding the cookie in CyberChef](/assets/img/posts/SkillsAssessmentUsingWebProxies/cyberchef-decode.png)
*Fig 2. The decoding process: Hex -> Base64.*

And here emerged the key problem of this task. The resulting string clearly looked like an MD5 hash, but it was **31 characters** long. A standard MD5 hash is always exactly 32 characters. The creator of the challenge intentionally removed the last character, forcing us to perform a brute-force (fuzzing) attack to guess it.

---

## 3. Hybrid Fuzzing (Python + Web Proxy)

The strategy was simple: take the 31-character base string, append every possible alphanumeric character (a-z, A-Z, 0-9) one by one, and then **re-encode** the entire new 32-character word (first Base64, then Hex) and send it to the server in the cookie.

Although advanced proxy tools can encode payloads on the fly (via *Payload Processors*), they can sometimes be finicky. In some situations, it's best to rely on flexible solutions. I wrote a short Python script that generated a ready-made list of 62 fully encoded cookie variants:

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
Armed with the ready `payloads.txt` file, I loaded it into the Fuzzer. This task could be solved in both **OWASP ZAP** (loading the list as a *File* type) and **Burp Suite** using the *Intruder* tool (*Sniper* attack type). 

After running the attack, all I had to do was sort the results by the server's response size. One request clearly stood out in size from the remaining 61 failed attempts – that's where the flag was hidden.

![Fuzzing attack results in proxy](/assets/img/posts/SkillsAssessmentUsingWebProxies/fuzzing-results.png)
*Fig 3. Hash fuzzing in OWASP ZAP.*

🚩 **Flag 2:** `HTB{burp_1n7rud3r_n1nj4!}`

---

## 4. The "Blind Script" Threat: Metasploit + Proxy

In the final stage, we had to use the `auxiliary/scanner/http/coldfusion_locale_traversal` module within the **Metasploit Framework**. However, automated scanners sometimes fail, and firing scripts without insight into what exactly they are sending is a dead end.

To diagnose the problem and answer the task's question (which hidden directory is Metasploit looking for?), I decided to route the MSF traffic through my local proxy. Following the rule *"when one tool strikes, fire up another"*, I switched to **Burp Suite**, which flawlessly intercepted the communication.

In the `msfconsole`, I just had to set the appropriate variable:

```bash
msf> set Proxies HTTP:127.0.0.1:8080
msf> run
```
After executing the module, a raw GET request appeared in Burp's HTTP history. I saw exactly the *Directory Traversal* attack structure and the path the task was asking about.

![Intercepted Metasploit request in Burp Suite](/assets/img/posts/SkillsAssessmentUsingWebProxies/metasploit-request.png)
*Fig 4. Analysis of the raw request sent by the automated tool.*

🚩 **Answer:** `CFIDE`

---

## Summary and Tool Insights

This short set of tasks is a great proving ground showing that in cybersecurity, understanding your tools is more important than memorizing exploits. In similar, real-world penetration testing scenarios, these same tools allow you to quickly identify vulnerabilities:

* **Browser DevTools:** Always verify the source code. Any front-end blocks like `disabled`, `hidden`, or character length limits are usually just an illusion of security that can be easily bypassed.
* **CyberChef + Python:** The perfect duo for detecting cryptographic anomalies. If you see a weird string (like a truncated MD5 hash), a few lines of script are enough to automate decoding and fuzzing, instead of doing it manually.
* **OWASP ZAP / Burp Suite:** These are your eyes and ears. Never trust scanners "blindly". Routing traffic from automated tools (like Metasploit) through a proxy lets you quickly spot whether the payload is actually hitting the right paths (like `/CFIDE/`) or getting lost along the way.

A good pentester does not cling to a single solution. Where one proxy struggles with unusual traffic, we smoothly switch to another. Flexibility and the ability to analyze raw traffic are the absolute foundation in our trade!

> **⚠️ Disclaimer and Task Information:**
> *This article is an original **Write-up** based on the **Skills Assessment - Using Web Proxies** task available on the [Hack The Box](https://www.hackthebox.com/) educational platform. The presented security analysis process is strictly for educational purposes.*
> 
> *The techniques presented should only be used during authorized penetration tests with the explicit consent of the infrastructure owner. The author is not responsible for any illegal use of the knowledge contained in this post.*

</details>