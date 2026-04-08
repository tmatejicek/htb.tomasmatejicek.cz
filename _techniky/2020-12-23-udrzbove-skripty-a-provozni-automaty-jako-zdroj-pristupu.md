---
layout: post
author: Tomáš Matějíček
title: "Údržbové skripty a provozní automaty jako zdroj přístupů"
date: 2020-12-23
tags: privesc automation cron backup maintenance ops
---
## Úvod a kontext

Některé privilege escalation cesty nevznikají z exploitu v tradičním smyslu, ale z toho, že systém už sám něco pravidelně dělá: resetuje heslo, kopíruje logy, spouští kontrolní skripty, komunikuje se zálohovacím backendem nebo přijímá vlastní pluginy a checky. Útočník pak nemusí vymýšlet nový mechanismus. Stačí pochopit a zneužít ten, který už v prostředí existuje.

To je důležité odlišit od jiných linuxových a windowsových témat:

- není to totéž jako `sudo` nad příkazem, který si spouští uživatel ručně,
- není to totéž jako `PATH` nebo `PYTHONPATH` hijack,
- a není to totéž jako běžná admin platforma typu dashboard nebo webový manager.

Tady jde o automatiku: o proces, který běží s vyšší důvěrou, často periodicky a často nad vstupem, který někdo podcenil.

## Co dělá automatizaci nebezpečnou

Údržbový skript nebo provozní helper se mění v útokovou plochu ve chvíli, kdy se sejdou tři vlastnosti:

- běží s vyššími právy nebo v citlivějším kontextu než útočník,
- pracuje s cestou, backendem nebo souborem, který může útočník ovlivnit,
- dělá něco mocného: kopíruje data, resetuje hesla, spouští příkazy nebo importuje obsah.

Každá z těchto vlastností samostatně ještě nemusí stačit. Ale dohromady z nich vzniká velmi stabilní privesc nebo pivot pattern.

## Typické scénáře zneužití

### 1. Automatické resetování nebo nastavování přístupů

Omni je čistý příklad. Skrytý `r.bat` neběžel jako náhodný pomocný skript, ale jako provozní rutina, která:

- resetovala heslo `administrator`,
- spravovala členství ve skupině `administrators`,
- a dělala to opakovaně bez další interakce správce.

Jakmile útočník tento skript našel, nevznikla potřeba hledat další kernel exploit nebo složitý bypass. Už existující maintenance logika sama vydala privilegovaný přístup.

To je důležitá lekce: automatické „dočasné“ recovery nebo provisioning mechaniky často otevřou systém mnohem spolehlivěji než původní zranitelnost.

### 2. Kopírování nebo synchronizace z útočníkem ovlivnitelné cesty

Tentacle ukazuje jiný vzor. Cron job `log_backup.sh` pravidelně kopíroval obsah `/var/log/squid/` do domovského adresáře `admin`. Samotný skript nevypadal dramaticky. Nebyl v něm `eval`, shell injection ani zjevný exploit. Problém byl v tom, že zdrojovou cestu mohl ovlivnit méně privilegovaný uživatel.

Jakmile šlo do log adresáře vložit `.k5login`, skript tento skrytý soubor poslušně zkopíroval do `/home/admin/`. Tím se z běžného backup helperu stal autentizační bridge mezi dvěma účty.

Právě to je klasický provozní antipattern:

- skript věří obsahu cesty, kterou nepovažuje za nebezpečnou,
- kopíruje i skryté nebo strukturálně významné soubory,
- a výsledkem je změna bezpečnostního kontextu v jiném uživatelském profilu.

### 3. Zálohovací nástroje s útočníkem řízeným backendem

Registry ukazuje variantu, kde automatizace sama neplánuje běh periodicky, ale systém už má připravený provozní workflow a důvěru k zálohovacímu nástroji. `restic` zde nebyl náhodný binární soubor v `sudoers`, ale součást skutečného backup modelu s REST backendem.

To je podstatné. Jakmile má privilegovaný proces číst data a posílat je na backend, musí být:

- pevně dané, co se smí zálohovat,
- pevně dané, kam se to smí poslat,
- a pevně dané, kdo backend určuje.

Pokud může útočník backend podstrčit nebo přesměrovat, helper z něj udělá delegovaný root file-read.

### 4. Provozní agenti, kteří umí nahrát a spustit skript

ServMon stojí na NSClient++, což není klasický cron job, ale stále jde o provozní automatizační komponentu. Umí přijímat konfigurační změny, uploadnout skript a spustit check v privilegovaném kontextu. Jakmile z lokální konfigurace unikne heslo a útočník se k API dostane přes port forwarding, agent sám provede zbytek.

Tady je důležité, že chyba neleží jen v jednom hesle. Leží v celém modelu:

- agent má vysoká práva,
- je navržený k vykonávání provozních úloh,
- a jeho „legitimní“ administrativní funkce jsou po compromise hostu přesně tím, co útočník potřebuje.

## Na co se při analýze zaměřit

U těchto situací se vyplatí jít po konkrétních otázkách.

### Co se spouští pravidelně nebo bez další kontroly

- cron,
- systemd timer,
- backup wrapper,
- helper agent,
- batch nebo PowerShell maintenance skript,
- synchronizační úloha.

### Čemu ten proces věří

- log adresáři,
- backup zdroji,
- backend URL,
- lokálním konfiguračním souborům,
- nahraným skriptům nebo checkům,
- skrytým souborům v kopírovaném stromu.

### Jaká je síla jeho dopadu

- resetuje hesla,
- kopíruje soubory do cizího homediru,
- čte privilegované cesty,
- provádí upload a execute,
- komunikuje se zálohovacím serverem,
- nebo zapisuje do autentizačně významného umístění.

Tohle je důvod, proč nestačí jen najít „nějaký cron“. Je potřeba pochopit celý tok důvěry: kdo ovládá vstup, kdo spouští akci a jak mocná ta akce je.

## Rozdíl proti jiným privesc vzorům

Je užitečné to držet odlišené od blízkých témat.

### Není to wrapper hijack

U `PATH` nebo `PYTHONPATH` hijacku útočník podstrkuje binárku nebo modul do lookup řetězce. Tady naopak často žádný lookup problém není. Skript dělá přesně to, co dělat měl, jen nad špatně vybraným vstupem.

### Není to obyčejné `sudo`

U `sudo` článku řešíš situaci, kdy uživatel sám ručně spustí privilegovaný nástroj. Tady může být rozhodující právě to, že proces běží sám, periodicky nebo skrz jiný operační workflow.

### Není to admin dashboard

U monitoringu a admin platforem je primární problém v API nebo rozhraní. Tady je primární problém v automatizaci a implicitní důvěře ke vstupu.

## Obrana a hardening

### 1. Omezit vstupy, které automatika zpracovává

Kopírovat logy, zálohy nebo uploady do privilegovaného kontextu lze jen tehdy, když je přesně definované:

- které soubory jsou povolené,
- zda se kopírují i skryté soubory,
- zda se zachovávají symlinky,
- a kdo může zdrojový strom ovlivnit.

### 2. Nepouštět resety hesel a recovery logiku v plaintext skriptech

Jakmile maintenance skript obsahuje skutečné produkční heslo nebo rutinu na reset přístupu, je to plnohodnotný credential store.

### 3. Oddělit backend od klientem volitelné konfigurace

Zálohovací a synchronizační nástroj nesmí slepě přijmout libovolný backend, URL nebo repozitář. Jinak se z něj stane pohodlný transport privilegovaných dat k útočníkovi.

### 4. Provozní agenty držet jako citlivé execution engine

Nástroje typu NSClient++, restic helper nebo jiné orchestrátory je potřeba brát jako vysoce privilegované komponenty. Jakmile umí uploadnout, spustit nebo přenášet data, musí mít stejně tvrdý model přístupu jako root shell.

### 5. Auditovat maintenance logiku stejně jako aplikaci

Pomocné skripty, batch soubory a backup helpery nesmějí stát mimo review jen proto, že jsou „provozní“. V praxi právě tam často končí nejcitlivější zkratky a nejméně promyšlené trust assumptions.

## Shrnutí klíčových poznatků

- Údržbové skripty a provozní automaty jsou nebezpečné tehdy, když s vyššími právy pracují nad vstupem, který může ovlivnit méně privilegovaný účet.
- Nejčastější dopad nevzniká shell injection, ale důvěrou: v kopírovaný adresář, resetované heslo, backend zálohy nebo privilegovaný agent.
- Omni, Tentacle, Registry i ServMon ukazují různé varianty stejného vzoru: systém už má mechanismus, který umí udělat něco mocného, a útočník ho jen přesměruje nebo nechá proběhnout nad vlastním vstupem.
- Obrana spočívá hlavně v tom, aby automatika dělala méně, věřila menšímu množství vstupů a neběžela nad zbytečně mocnými kontexty.

## Co si odnést do praxe

- Při lokální enumeraci se neptej jen `co můžu spustit`. Ptej se i `co se spouští samo` a `čemu to věří`.
- Každý backup helper, reset skript nebo provozní agent je potenciální privesc kandidát, pokud jeho vstup může ovlivnit jiný účet.
- U automatizace je potřeba auditovat celý tok důvěry. Ne jen příkaz, ale i zdroj dat, cílový kontext a to, kdo určuje backend nebo kopírovanou cestu.
