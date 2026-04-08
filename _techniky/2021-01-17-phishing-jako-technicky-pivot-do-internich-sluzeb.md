---
layout: post
author: Tomáš Matějíček
title: "Phishing jako technický pivot do interních služeb"
date: 2021-01-17
tags: phishing email credentials internal-services foothold
---
## Úvod a kontext

O phishingu se často mluví hlavně sociologicky: jak napsat důvěryhodný e-mail, jak zvýšit šanci na kliknutí nebo jak školit uživatele. To je důležité, ale z technického hlediska často rozhoduje až to, co se stane potom. Jediné získané heslo obvykle nevede rovnou k plnému kompromisu. Vede ke grafu dalších služeb, které si navzájem důvěřují:

- pošta,
- FTP,
- vývojový vhost,
- interní repozitář,
- SSH,
- nebo jiný servisní workflow.

SneakyMailer je dobrý právě proto, že celý řetězec začíná phishingem, ale technická hodnota se ukáže až později. Zachycené heslo otevře poštu, pošta vydá další přístup, ten otevře FTP a vývojový web, a teprve potom vzniká skutečný foothold.

## O čem tenhle článek je a o čem není

Tenhle text není o copywritingu phishingu ani o psychologii oběti. Je o technickém modelu:

1. útočník získá jeden credential,
2. zkusí určit, pro které další služby může být platný,
3. z těchto služeb vytěží další interní informace,
4. a z nich postaví další pivot.

Jinými slovy: phishing tu nefunguje jako konečný útok. Funguje jako první identity foothold.

## Proč je jedno heslo tak cenné

V běžném prostředí není uživatelský účet izolovaný na jedné službě. Stejné nebo návazné přihlašovací údaje často propojují:

- webmail a IMAP,
- interní portál a FTP,
- poštu a helpdesk,
- uživatelský účet a vývojový vhost,
- nebo e-mail a interní package registry.

To je důležité. Hodnota hesla nespočívá jen v tom, že otevře jednu schránku. Hodnota spočívá v tom, že schránka, adresář kontaktů, staré zprávy a interní instrukce často ukážou, kam s tímto účtem pokračovat dál.

## SneakyMailer: phishing jako vstup, ne jako finální cíl

Na SneakyMailer byl první technický posun relativně jednoduchý: veřejně dostupný SMTP a firemní login vytvořily realistickou příležitost zachytit přihlašovací údaje z falešného formuláře.

Získaný účet:

```text
paulbyrd@sneakymailer.htb
```

a jeho heslo ještě samy o sobě neznamenaly shell. Znamenaly ale přístup do další informační vrstvy. Právě to je první důležitá lekce: po phishingu je často cennější pošta než samotný webový login.

## Co z pošty útočník skutečně získá

E-mailový účet není jen další login. V praxi často poskytne:

- další pověření poslaná v minulosti,
- onboarding instrukce,
- interní hostname,
- názvy vývojových systémů,
- informace o deploy procesu,
- nebo technické role dalších účtů.

Na SneakyMailer právě pošta ukázala další důležitý účet:

```text
Username: developer
Original-Password: m^AsY7vTKVT+dV1{WOU%@NaHkUAId3]C
```

Tím se z „kompromitované schránky“ stává technický pivot. Útočník už nepracuje jen s jedním uživatelem. Pracuje s účtem, který zjevně patří do vývojového workflow.

## Jak se z hesla stane přístup k další službě

Jakmile se objeví účet typu `developer`, dává smysl nezkoušet ho nahodile proti všemu, ale proti službám, které mají z provozního hlediska nejvyšší pravděpodobnost reuse:

- FTP,
- vývojový vhost,
- interní Git nebo package registry,
- SSH na dev host,
- CI/CD nebo upload rozhraní.

Na SneakyMailer vyšly právě dvě z nich:

- FTP přístup,
- a vývojová subdoména `dev.sneakycorp.htb`.

To je důležitý vzorec. Phishing často nevede k privilegiím přímo. Vede k účtu, který zná cestu do vývojové nebo interní vrstvy, kam by se útočník zvenku jinak nedostal.

## Vývojový vhost jako násobič dopadu

Jakmile účet `developer` dovolil upload přes FTP a současně existoval `dev.sneakycorp.htb`, vznikla zcela jiná situace než při prostém kompromitování webmailu.

Výsledkem už nebyla „jen cizí schránka“, ale:

- write přístup do prostředí,
- možnost nahrát jednoduchý webshell,
- a lokální pohled na další interní služby hostu.

Tady je dobře vidět, proč je phishing z technického pohledu tak nebezpečný. Útočník nemusí mít exploit na veřejný web. Stačí mu účet, který mu dá přístup do workflow, jež samo obsahuje execution nebo deployment cestu.

## Další vrstva: interní služby, které veřejně nejsou vidět

Na SneakyMailer vedl webshell k objevu interního PyPI:

```text
pypi.sneakycorp.htb
```

To je další opakující se pattern. Phishing často otevře ne tu službu, kterou chceš kompromitovat, ale službu, ze které uvidíš další vnitřní infrastrukturu:

- package registry,
- interní hostname,
- service účty,
- lokální konfigurační soubory,
- `.pypirc`, `.npmrc`, `.netrc`,
- nebo helper skripty.

Technický dopad phishingu tedy roste po vrstvách. Každá další interní služba přidává další důvěryhodný kontext.

## Jak podobné řetězce číst systematicky

Po získání jednoho hesla dává smysl jít po těchto otázkách:

### 1. K jakému typu účtu jsem se dostal

- běžný uživatel,
- support,
- developer,
- admin,
- servisní účet.

To určuje, které další systémy budou nejspíš relevantní.

### 2. Co mi účet ukáže o vnitřním prostředí

Pošta, konfigurace klienta nebo webmail mohou vydat:

- další účty,
- názvy interních hostů,
- konfigurační přílohy,
- deploy instrukce,
- a návazné identity.

### 3. Kde dává reuse technicky největší smysl

Nejde o slepé zkoušení. Jde o odhad důvěryhodných vazeb:

- developer -> FTP,
- developer -> dev vhost,
- poštovní účet -> IMAP/SMTP,
- lokální konfigurace -> package registry.

### 4. Která ze služeb už sama dovolí write nebo execute

Právě tady se z identity footholdu stává technický foothold:

- upload,
- webshell,
- package upload,
- deploy,
- nebo vzdálený shell.

## Proč je to obranně jiné než běžný password reuse

Na první pohled může phishingový řetězec vypadat jen jako další případ password reuse. Rozdíl je v tom, že phishing obvykle nezasáhne izolovaný účet, ale účet v silně propojeném workflow:

- zaměstnanec dostává technické instrukce e-mailem,
- vývojář dostává přístupy do vhostu a FTP,
- interní balíčkovací infrastruktura používá lokální configy,
- a každá další služba přidává další vnitřní kontext.

Proto je technický phishing často mnohem víc než „ukradené heslo do schránky“. Je to první klíč do mapy interních vztahů.

## Obrana a hardening

### 1. Nepředpokládat, že kompromitovaný e-mail je „jen e-mail“

Poštovní účet má být součást identity bezpečnostního modelu. Pokud jeho kompromitace otevře vývojové nebo provozní systémy, problém je v propojení služeb, ne jen v samotném mailu.

### 2. Oddělit uživatelské identity od vývojových a provozních účtů

Účet používaný pro e-mail nebo běžnou práci nemá zároveň otevírat FTP, deploy vhost nebo interní package registry bez další ochrany.

### 3. Chránit návazné služby silnější autentizací

MFA nebo jiný druh dodatečného ověření dává největší hodnotu právě tam, kde první kompromitovaná identita jinak vede k další technicky citlivé vrstvě.

### 4. Omezit citlivé informace v poště a klientských konfiguracích

Přístupové údaje, onboarding hesla, konfigurační přílohy a návody typu „přihlaš se na dev host tady“ výrazně zvyšují cenu kompromitované schránky.

### 5. Detekovat neobvyklé přechody mezi rolemi a službami

Po úspěšném phishingu bývá varovný hlavně přechod:

- mail -> FTP,
- mail -> interní vhost,
- mail -> package registry,
- nebo mail -> SSH.

Právě tyto skoky signalizují, že kompromitovaná identita začíná fungovat jako technický pivot.

## Shrnutí klíčových poznatků

- Phishing je z technického pohledu často jen první identity foothold, ne konečný útok.
- SneakyMailer ukazuje celý řetězec: získané heslo -> pošta -> účet `developer` -> FTP a dev vhost -> další interní služby.
- Největší hodnota kompromitované schránky neleží v samotném čtení pošty, ale v tom, jaké další identity, hostname a workflow vydá.
- Obrana musí řešit nejen phishing samotný, ale i to, jak daleko může kompromitovaný účet v infrastruktuře pokračovat.

## Co si odnést do praxe

- Po phishingu se vždy ptej, jaké další služby může kompromitovaný účet legitimně otevírat.
- E-mailový účet je často vstup do technického workflow, ne jen komunikační nástroj.
- Když jedna kompromitovaná schránka otevře vývojový vhost nebo interní package registry, problém neleží jen v uživateli. Leží v příliš široké důvěře mezi službami.
