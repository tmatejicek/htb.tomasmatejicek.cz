---
layout: post
author: Tomáš Matějíček
title: "kid/jku a vzdálené načítání klíčů"
date: 2021-01-24
tags: jwt web auth jwks kid jku
---
## Úvod a kontext

Článek o uniklém JWT signing secretu řeší situaci, kdy útočník získá klíč používaný pro podepisování a pak sám vytváří důvěryhodné tokeny. Téma `kid` a `jku` je jiné. Tady nemusí signing secret uniknout vůbec. Problém vzniká ve chvíli, kdy server přenechá klientovi rozhodnutí, odkud se má ověřovací klíč vzít.

To je zásadní rozdíl. Bezpečný návrh říká: aplikace má vlastní trust store a token jí maximálně pomůže vybrat správný klíč z už důvěryhodné sady. Nebezpečný návrh říká: token sám ukáže na URL, soubor nebo keyset, kterému pak server uvěří. V tu chvíli už neověřuje podpis server, ale útočník.

## Co `kid` a `jku` mají dělat správně

V bezpečném návrhu slouží hlavička JWT jen jako nápověda:

- `kid` vybírá konkrétní klíč z lokálního úložiště nebo z předem známého JWKS,
- `jku` nebo podobný odkaz směřuje jen na pevně definovaný, serverem vlastněný keyset,
- aplikace sama určuje, které issuery, algoritmy a keysety jsou důvěryhodné.

Klient tedy může říct `ověř mě klíčem s ID abc123`, ale nesmí říct `ověř mě klíčem, který si stáhneš z mé URL`.

Právě tenhle rozdíl bývá v implementaci rozmazaný. Vývojář chce podporu pro rotaci klíčů, několik issuerů nebo pohodlnou správu JWKS. Jenže pokud se tahle flexibilita přenese do klientem ovlivnitelných hlaviček bez pevného trust boundary, vzniká z ní přímý bypass celé tokenové důvěry.

## Kde se z flexibilního návrhu stává chyba

Rizikové jsou hlavně tyto vzory:

- server načítá klíč nebo certifikát z URL uvedené v tokenu,
- `kid` se interpretuje jako cesta, URL nebo jiný externí identifikátor místo lokálního indexu,
- aplikace přijímá `jku`, `x5u` nebo podobný odkaz bez pevného allowlistu a validace issueru,
- JWKS cache se váže na vstup z tokenu a ne na serverem určený issuer,
- aplikace smíchá legitimní key rotation s možností, aby klient dodal vlastní zdroj důvěry.

Všechny tyto varianty vypadají jinak, ale chyba je stejná: útočník už neovlivňuje jen payload tokenu. Ovlivňuje i to, čím se jeho podpis bude ověřovat.

## Příklad z praxe: TheNotebook

TheNotebook je čistá ukázka tohoto patternu. Aplikace používala asymetrické podepisování a vypadalo to jako bezpečnější varianta než klasické `HS256` se shared secretem. Problém ale nebyl v algoritmu. Problém byl v tom, že hlavička tokenu obsahovala `kid`, které ukazovalo na URL s klíčem:

```json
{
  "typ": "JWT",
  "alg": "RS256",
  "kid": "http://localhost:7070/privKey.key"
}
```

Server pak při ověřování slepě sáhl na adresu z tokenu, stáhl si odtud klíč a použil ho k validaci podpisu. Jakmile mohl útočník změnit `kid` na vlastní HTTP server, stačilo:

1. vygenerovat si vlastní pár klíčů,
2. vystavit veřejný klíč na útočníkem řízené URL,
3. podepsat token privátním klíčem,
4. změnit v payloadu třeba `admin_cap` na `true`.

Tím se ochrana podpisem nezlomila kryptograficky. Zlomila se architektonicky. Server neověřoval, zda token podepsal důvěryhodný issuer, ale zda ho dokáže ověřit klíčem, který mu sám token podstrčil.

## Proč je to jiné než uniklý signing secret

Je užitečné ten rozdíl držet ostře, protože v praxi se tyto problémy často slévají pod obecné „JWT bypass“.

### Uniklý signing secret

Na strojích jako Secret, Player nebo Cereal byl problém jiný:

- signing secret unikl z Git historie, backupu nebo zdrojáku,
- útočník pak podepsal vlastní token,
- aplikace věřila claimům, protože podpis odpovídal známému tajemství.

To je kompromitace klíče.

### `kid`/`jku` trust failure

Na TheNotebooku signing secret nebo privátní klíč serveru uniknout nemusel. Server sám přijal cizí zdroj ověřovacího klíče.

To je kompromitace trust modelu.

Oba scénáře mohou skončit stejným výsledkem, tedy podvrženým admin tokenem. Obrana je ale jiná:

- u uniklého secretu řešíš ochranu a rotaci klíče,
- u `kid`/`jku` řešíš to, kdo smí určovat, odkud se klíč bere.

## Jak vypadá bezpečný a nebezpečný model

### Bezpečný model

- server zná seznam issuerů,
- ke každému issueru zná vlastní JWKS endpoint nebo lokální keyset,
- `kid` vybírá klíč uvnitř této předem důvěryhodné sady,
- hlavička tokenu nerozhoduje o tom, na jaký host nebo soubor se sahá.

### Nebezpečný model

- token nese URL, cestu nebo jiný identifikátor, který server přímo použije,
- důvěryhodnost klíče se odvozuje od toho, že se ho podařilo stáhnout,
- issuer, keyset a klíč nejsou pevně svázané na serverové straně,
- key discovery se mění z interní konfigurace na klientem řízenou operaci.

Právě v tomto bodě vzniká chyba, kterou nejde opravit jen „silnějším algoritmem“. `RS256` ani `ES256` nepomůže, pokud si ověřovací klíč necháš vybrat útočníkem.

## Na co se zaměřit při auditu

Při review nebo testu je dobré projít několik konkrétních otázek:

### Odkud se ověřovací klíč bere

- je klíč nebo JWKS definovaný v serverové konfiguraci,
- nebo se zdroj odvozuje z hlavičky tokenu.

### Co přesně znamená `kid`

- je to jen interní identifikátor,
- nebo se převádí na cestu, URL nebo jiný externí lookup.

### Jak je svázaný issuer a keyset

- vybírá aplikace issuer ze serverového allowlistu,
- nebo jen vezme token a podle něj si sama „dohledá“, čemu má věřit.

### Jak se chová cache a fallback

- nebere aplikace neznámý `kid` jako důvod k novému fetchi na klientem řízený endpoint,
- neexistuje fallback logika, která při chybě sáhne po alternativním zdroji z tokenu.

### Jak silně aplikace věří claimům po validaci

I u `kid`/`jku` platí stejná druhá otázka jako u signing secretu: jak moc aplikace podle tokenu skutečně rozhoduje. Pokud payload nese roli, tenant nebo administrativní příznaky, dopad je okamžitý.

## Obrana a hardening

### 1. `kid` jen jako index do lokálního trust store

To je nejdůležitější pravidlo. `kid` smí vybírat z klíčů, které aplikace už zná jako důvěryhodné. Nesmí sloužit jako URL, cesta ani obecný klíč k dynamickému fetchi.

### 2. `jku` a podobné odkazy jen na pevně definované zdroje

Pokud aplikace opravdu potřebuje vzdálený JWKS, musí být endpoint svázaný se známým issuerem a kontrolovaný serverovou konfigurací. Ne tokenem.

### 3. Oddělit key discovery od klientského vstupu

Klient může předložit token. Ale issuer, JWKS a očekávaný algoritmus musí určit server podle vlastní konfigurace.

### 4. Striktně validovat issuer a algoritmus

I dobře navržený keyset se rozpadne, pokud server benevolentně přijímá cizí issuer, algoritmus nebo strukturu tokenu a teprve potom zjišťuje, čím ho ověří.

### 5. Monitorovat nečekané odchozí lookupy při validaci tokenů

Aplikace, která při zpracování JWT náhle navazuje HTTP spojení podle klientského vstupu, už bezpečnostně selhala. Právě to je praktický detekční signál podobných chyb.

## Shrnutí klíčových poznatků

- `kid` a `jku` nejsou problém samy o sobě. Problém vzniká ve chvíli, kdy klient začne určovat zdroj ověřovacího klíče.
- U `kid`/`jku` nejde primárně o únik tajemství, ale o chybu v trust modelu mezi tokenem, aplikací a keysetem.
- Asymetrické podepisování samo o sobě nepomůže, pokud server důvěřuje klíči dodanému nebo odkazovanému útočníkem.
- Bezpečný návrh drží key discovery na serverové straně a používá hlavičku tokenu jen jako nápovědu uvnitř už důvěryhodného kontextu.

## Co si odnést do praxe

- Když vidíš `kid`, ptej se, jestli je to jen interní identifikátor, nebo skrytý způsob, jak klient vybírá ověřovací klíč.
- JWT review nekončí u `alg=none` nebo u síly signing secretu. Stejně důležité je i to, kdo řídí lookup klíče.
- Pokud validace tokenu sahá na cizí URL podle hlavičky tokenu, nejde o elegantní key rotation. Jde o plně kompromitovaný model důvěry.
