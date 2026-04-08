---
layout: post
author: Tomáš Matějíček
title: "Message brokery a interní fronty jako zdroj tajemství"
date: 2020-12-29
tags: message-broker mqtt redis queues secrets
---
## Úvod a kontext

Message broker nebo interní fronta se v architektuře často bere jako technický detail. Aplikace něco pošle do Redis queue, MQTT topicu nebo jiného interního kanálu a někdo jiný to zpracuje. Jenže právě tím se z brokeru stává bezpečnostní hranice. Nenese jen data. Nese provozní instrukce, stavy workflow, někdy i privátní klíče, tokeny nebo celé job payloady.

PlayerTwo, Ready a Postman ukazují tři různé podoby stejného problému:

- jednou interní MQTT topic prozradí privátní klíč,
- podruhé Redis queue umožní vložit útočníkův job do GitLabu,
- a potřetí otevřený Redis ukáže, že stejný produkt může být zároveň cache, queue i konfigurační vrstva s velmi vysokou hodnotou.

Tento text záměrně nenavazuje na storage článek přes "špatně vystavená data". Tady je důležité něco jiného: co všechno se v interních zprávách a frontách přenáší a proč se z transportní vrstvy tak snadno stane zdroj tajemství nebo rovnou řídicí rozhraní.

## Proč jsou message brokery a fronty bezpečnostně tak citlivé

### Nesou víc než business data

Ve zprávách často necestuje jen uživatelský obsah. Cestuje v nich i:

- podpisový materiál,
- interní eventy,
- názvy jobů a workerů,
- parametry dávkových úloh,
- přihlašovací údaje k dalším službám,
- nebo provozní metadata, která nikdo nečekal mimo backend.

### Bývají považované za "interní"

Redis, MQTT a podobné systémy často běží:

- jen na `127.0.0.1`,
- na vedlejším portu bez veřejné dokumentace,
- nebo za předpokladu, že je používá jen aplikace sama.

Jakmile se ale útočník dostane dovnitř hostu, do kontejneru nebo k SSRF, mění se broker z interního pomocníka v plnohodnotnou attack surface.

### Jsou to řídicí kanály, ne jen úložiště

Rozdíl proti prostému storage je zásadní. Když aplikace čte soubor, je to statický artefakt. Když čte queue nebo topic, očekává:

- že zprávě může důvěřovat,
- že ji zpracuje jiná privilegovaná komponenta,
- a že obsah nese provozní význam.

Právě to dává message brokerům tak vysokou hodnotu.

## Jak tenhle vzorec vypadal v praxi

### PlayerTwo: MQTT topic s privátním klíčem

PlayerTwo je nejčistší ukázka toho, že interní broker umí nést rovnou tajemství. Po prvním footholdu jako `www-data` bylo možné připojit se na lokální Mosquitto broker pomocí [mosquitto_sub](/nastroje/mosquitto-sub):

```bash
mosquitto_sub -h localhost -t '$SYS/#'
```

Na tématu `$SYS/internal/firmware/signing` se periodicky objevoval privátní klíč používaný k podepisování firmwaru. Ten byl zároveň použitelný jako SSH klíč pro účet `observer`.

To je důležitá lekce. Problém neležel v MQTT jako protokolu. Problém ležel v tom, že provozní tajemství cestovalo v topicu, který byl považovaný za interní a bezpečný jen proto, že broker běžel lokálně.

### Ready: Redis queue jako cesta ke spuštění jobu

Na Ready šel GitLab zneužít přes SSRF a CRLF injection tak, aby poslal vlastní příkazy do lokálního Redis. Praktický dopad nebyl "přečtu si klíč z Redis". Dopad byl vyšší: Redis sloužil jako queue backend pro GitLab joby a útočník tak mohl vložit vlastní job pro `GitlabShellWorker`.

To je dobré rozlišovat. Redis zde nebyl zajímavý jako key-value store s tajemstvím. Byl zajímavý jako řídicí kanál, přes který aplikace spouštěla vlastní pracovní úlohy. Jakmile se do něj dalo zapisovat, nešlo už o leak, ale o přímý vliv na chování backendu.

### Postman: Redis jako hranice mezi storage, queue a správou

Postman je užitečný kontrastní případ. Otevřený Redis zde primárně nevedl přes interní fronty, ale přes `CONFIG SET`, `dbfilename` a `SAVE` k zápisu `authorized_keys`. Na první pohled to vypadá spíš jako storage chyba než message-broker problém.

Právě to je ale didakticky cenné. V reálném prostředí jeden Redis často plní více rolí najednou:

- cache,
- session backend,
- queue,
- pomocné úložiště konfigurace,
- nebo provozní koordinaci.

Postman proto připomíná důležitou otázku: když najdete Redis, nemá smysl řešit jen "co v něm je teď". Smysl má řešit i "co všechno přes něj aplikace a okolní systém dělají".

## Co je na těchto případech společné

### 1. Broker nese implicitní důvěru

Aplikace a pracovníci kolem ní předpokládají, že:

- topic je interní,
- frontu plní jen trusted producer,
- a obsah zprávy je provozně validní.

Jakmile tento předpoklad padne, dopad bývá větší než u obyčejného datového leaku.

### 2. Tajemství se v provozu šíří dál, než by měla

PlayerTwo ukazuje privátní klíč v MQTT. Ready ukazuje job payload vedoucí ke spuštění příkazu. V obou případech nejde o "tajemství uložené v DB". Jde o tajemství nebo instrukci šířenou provozním kanálem, který nikdo neauditoval jako citlivou vrstvu.

### 3. Interní port neznamená malý dopad

Tyto systémy často neběží veřejně. Přesto mají po footholdu velmi vysokou hodnotu, protože sedí přesně mezi aplikací a její orchestrací. Odtud už bývá blízko:

- k dalším identitám,
- k workerům,
- k podpisovým klíčům,
- nebo k úplnému převzetí backendového workflow.

## Jak podobné problémy hledat po footholdu

Po prvním shellu nebo po úspěšné SSRF dává smysl cíleně zjišťovat:

- jestli běží Redis, MQTT nebo jiný broker,
- kdo je na něj připojený,
- která témata nebo fronty existují,
- jestli jimi putují tajemství, podpisový materiál nebo job payloady,
- a jestli producent skutečně ověřuje, kdo do kanálu zapisuje.

U lokálně běžících služeb typicky pomůže:

```bash
ss -lntp
ps aux
```

A pak už záleží na konkrétním systému:

- subscription na MQTT témata,
- čtení front nebo konfigurace Redis,
- a hlavně pochopení, jak broker zapadá do zbytku aplikace.

## Jak si to nesplést se storage a localhost články

Tohle téma se přirozeně dotýká i jiných oblastí, ale je užitečné ho držet samostatně.

- Oproti článku o lokálních službách jde tady méně o to, že něco běží na `127.0.0.1`, a víc o to, co se přes to přenáší.
- Oproti článku o storage službách jde méně o veřejně čitelné objekty a víc o řídicí nebo provozní význam zpráv.

Jednoduchá otázka pomáhá dobře: je hlavní hodnota v tom, že služba něco ukládá, nebo v tom, že přes ni někdo někomu říká, co má udělat? Pokud platí druhé, je to spíš message broker než storage problém.

## Obrana a bezpečnější návrh

### 1. Neposílat tajemství v provozních zprávách

Privátní klíče, dlouhodobé tokeny a jiné citlivé artefakty nemají cestovat přes MQTT topic, Redis queue nebo podobný interní kanál. Jakmile takový kanál někdo zpřístupní nebo odposlechne, dopad je okamžitý.

### 2. Oddělit producenty a konzumenty pomocí autentizace a ACL

Broker nemá důvěřovat každému klientovi jen proto, že je "lokální" nebo "ze správné aplikace". Témata a fronty potřebují:

- autentizaci,
- autorizaci,
- a omezení, kdo smí číst a kdo smí zapisovat.

### 3. Hlídat, co přes queue skutečně spouští backend

Ready dobře ukazuje, že queue není neutrální vrstva. Pokud worker interpretuje zprávu jako job s vysokou důvěrou, je potřeba přísně kontrolovat, kdo a jak takovou zprávu může vytvořit.

### 4. Nenechávat Redis a podobné systémy v "interním defaultu"

Postman připomíná, že vystavený Redis je problém i tehdy, když aplikace nikde nepíše slovo "message queue". Jeden produkt může být současně cache, fronta i admin backend. Bez autentizace nebo segmentace je jeho dopad příliš široký.

### 5. Monitorovat nečekané subscription a zápisy

Na produkci je podezřelé:

- když se někdo přihlásí k interním MQTT tématům,
- když do Redis queue zapisuje nečekaný producer,
- nebo když se v brokeru začnou objevovat payloady, které neodpovídají běžnému workflow.

## Shrnutí klíčových poznatků

- Message broker nebo interní fronta jsou bezpečnostní hranice, ne jen transportní detail.
- PlayerTwo ukazuje tajemství unikající přes MQTT, Ready řídicí joby v Redis queue a Postman připomíná, že stejný backend často plní víc rolí najednou.
- Hlavní riziko neleží jen v tom, že se data ukládají. Leží v tom, že broker přenáší důvěru, instrukce a někdy i klíče.

## Co si odnést do praxe

- Když po footholdu najdete MQTT nebo Redis, neřešte jen obsah. Řešte i to, kdo přes něj komunikuje, co se přes něj řídí a jakou důvěru na něj navazuje zbytek systému.
- Interní fronta je často blíž ke spuštění kódu nebo k úniku tajemství než běžná databáze. Dopad proto bývá vyšší, než napovídá název služby.
- Pokud architektura spoléhá na to, že broker je "jen interní detail", je velmi pravděpodobné, že mu zároveň svěřila víc důvěry, než je bezpečné.
