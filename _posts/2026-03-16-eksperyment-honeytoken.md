---
title: "Eksperyment: Kto skanuje Twój kod? 48h z Honeytokenem na GitHubie"
date: 2026-03-16 18:00:00 +0100
categories: [Cybersecurity, Research]
tags: [honeytokens, aws, github, opsec, forensics]
image:
  path: /assets/img/posts/honeytoken/honeytoken.png
  alt: Mapa globalnych alertów wygenerowana przez CanaryTokens.
---

## Wstęp: Po co w ogóle to zrobiłem?
Wielu z nas żyje w przekonaniu, że szybkie usunięcie pliku z GitHuba po przypadkowym commicie załatwia sprawę. "Nikt chyba nie zdążył zauważyć, prawda?". Postanowiłem to sprawdzić w praktyce, tworząc całkowicie kontrolowany wyciek danych. 

Moim celem było zbadanie, jak szybko i z jakich kierunków nadejdą próby użycia "skradzionych" poświadczeń.

**Narzędzie:** [CanaryTokens](https://canarytokens.org). Wygenerowałem tam fałszywe klucze AWS. Nie dają one dostępu do absolutnie niczego, ale pełnią rolę cyfrowej pułapki (IDS) – każda próba ich użycia natychmiast wysyła mi maila z adresem IP i danymi napastnika.

---

## Przebieg eksperymentu
Zrobiłem klasyczną "wpadkę" początkującego dewelopera:
1. **Wyciek:** Wrzuciłem klucze prosto do publicznego repozytorium w pliku `scripts/deploy.sh`.
2. **"Panika":** Odczekałem 5 minut i usunąłem plik, robiąc nowy commit. Zniknął z głównego widoku, ale oczywiście został w historii.
3. **Nasłuch:** Zostawiłem pułapkę samą sobie i zacząłem zbierać logi.

---

## Rezultaty: Anatomia zmasowanego ataku
Gdy zacząłem analizować powiadomienia, uderzyła mnie jedna rzecz: totalna automatyzacja. To nie byli ludzie wklepujący klucze w terminal. To były zoptymalizowane skrypty wykonujące całe sekwencje komend.

### 1. Pierwsza krew: Skanery i "głębokie kopanie"
Zaledwie sekundy po publikacji uaktywniły się profesjonalne skanery bezpieczeństwa. Co jednak ciekawe, narzędzia takie jak **TruffleHog** wyciągnęły moje klucze bezpośrednio z historii commitów – już po tym, jak rzekomo "posprzątałem" repozytorium.

### 2. Globalny zasięg wycieku
Mój fałszywy klucz rozszedł się po świecie w mgnieniu oka. Z logów wyczytałem kilka głównych kierunków uderzenia:

* **USA (Chicago, New York, Wilmington):** Zmasowany ruch z serwerów takich gigantów jak **Comcast**, **Verizon** czy **AT&T**. Z logów wynikało, że boty używały `aws-sdk-go-v2` – a więc profesjonalnych, szybkich narzędzi napisanych w języku Go.
* **Kanada (OVH SAS - Quebec):** Skrypty z tej lokalizacji były najbardziej upierdliwe. Przez kilka dni w regularnych odstępach czasu uderzały komendą `GetCallerIdentity`, sprawdzając, czy klucze wciąż żyją.
* **Niemcy/Francja (Contabo GmbH):** Tutaj odnotowałem dość zaawansowane próby rekonesansu z użyciem klasycznego `aws-cli`.

### 3. Czego dokładnie szukano? (Komendy API)
Boty nie ograniczyły się do weryfikacji tożsamości. Logi zarejestrowały całą serię prób dobrania się do konkretnych usług AWS:
* **Secrets Manager (`ListSecrets`):** Klasyka. Próba znalezienia innych haseł i kluczy do baz danych.
* **EC2 (`DescribeInstances`):** Poszukiwanie aktywnych serwerów, które można by przejąć (np. do odpalenia koparek kryptowalut).
* **S3 (`ListBuckets`):** Próba dobrania się do prywatnych plików i backupów.
* **SES (`GetAccountSendingEnabled` / `ListIdentities`):** Weryfikacja, czy z mojego konta da się rozsyłać miliony maili z phishingiem.

### 4. Anonimowość pod przykryciem Tora 🕵️‍♂️
Najciekawszy rodzynek w logach to trafienie z **Tor Exit Node** w Austrii (`tor-exit-anonymizer.appliedprivacy.net`). To jasny sygnał: ktoś (lub bardzo zaawansowany botnet) po cichu weryfikował uprawnienia moich kluczy używając `botocore`, robiąc wszystko, by całkowicie ukryć swój prawdziwy adres IP.

---

## Analiza Danych w Pigułce

Poniższa mapa przedstawia wizualizację aktywności tych wszystkich botów i researcherów, którzy "połknęli haczyk":

![Mapa alertów](/assets/img/posts/honeytoken/honeytoken.png)
_Każda kropka to inna organizacja i inna, zautomatyzowana próba dobrania się do danych._

**Statystyki eksperymentu:**
* **Ulubione narzędzie botów:** `Boto3` (Pythonowa biblioteka do AWS).
* **Najczęstsza akcja:** `GetCallerIdentity` (klasyczny rekonesans).
* **Czas pierwszej reakcji:** Poniżej 1 minuty od pusha.

---

## Wnioski na przyszłość
1. **GitHub to dziki zachód:** Jeśli wrzuciłeś klucz do publicznego repozytorium, uznaj go za spalony w ciągu 60 sekund. Nie ma wyjątków.
2. **Historia Git nigdy nie zapomina:** Robienie commita z dopiskiem "usunięto hasło" to proszenie się o kłopoty. Boty przeszukują historię równie chętnie, co aktualny kod.
3. **Honeytokeny robią robotę:** To świetny sposób na wczesne ostrzeganie. Pozwalają wykryć włamanie na długo zanim napastnik dotrze do realnych, wrażliwych danych.

Jeśli wydaje ci się, że monitorowanie logów to paranoja to ten eksperyment udowadnia, że to po prostu konieczność.

---

## Technical Deep Dive: Zajrzyjmy w surowe logi (JSON)

Dla tych z was, którzy lubią analizować surowe dane, wyciągnąłem dwa najciekawsze hity z logów CanaryTokens. To one najlepiej pokazują, jak działają boty.

#### Przykład 1: Świadoma anonimizacja (Sieć Tor)
Napastnik z Austrii. Użył oficjalnego narzędzia **aws-cli**, ale starannie ukrył się za węzłem wyjściowym sieci Tor.
```json
{
  "src_ip": "109.70.100.9",
  "geo_info": {
    "org": "Foundation for Applied Privacy",
    "hostname": "tor-exit-anonymizer.appliedprivacy.net",
    "country": "AT"
  },
  "useragent": "aws-cli/1.22.34 Python/3.10.12 botocore/1.23.34",
  "additional_info": {
    "aws_key_log_data": {
      "eventName": ["GetCallerIdentity"]
    }
  }
}
```
#### Przykład 2: Szukanie darmowego serwera do spamu (SES)
To trafienie z infrastruktury Verizon (USA) świetnie pokazuje automatyzację "biznesową". Bot od razu celuje w usługę mailową (SES), aby sprawdzić, czy może z mojego konta wysyłać darmowy spam.
```json
{
  "src_ip": "98.118.135.128",
  "geo_info": {
    "org": "AS701 Verizon Business",
    "city": "Hamburg",
    "country": "US"
  },
  "useragent": "aws-sdk-go-v2/1.41.2 lang/go#1.26.0",
  "additional_info": {
    "aws_key_log_data": {
      "eventName": ["GetAccountSendingEnabled"]
    }
  }
}
```