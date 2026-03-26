---
layout: post
author: Tomáš Matějíček
title: "Repozitář, historie konfigurace a deployment trust"
date: 2020-11-06
tags: git repo deployment ci-cd config secrets
---
## Úvod a kontext

Článek o otevřeném `.git` řeší první fázi problému: jak se útočník dostane ke zdrojákům, zálohám nebo starým commitům. Tady jde o druhou fázi. Co se stane ve chvíli, kdy repozitář nebo historická konfigurace nejsou jen zdroj informací, ale přímo řídí nasazení, správu nebo privilegovanou automatiku.

Právě to je důvod, proč kompromitace vývojového prostředí často nekončí u úniku tajemství. Jakmile produkce čte konfiguraci z pracovního adresáře, administrátor spouští `git pull` pod `sudo`, Tomcat Manager používá heslo z dávno smazaného commitu nebo provozní konfigurace pod `/etc` pořád obsahuje `.git`, mění se verzovací historie v aktivní útokovou plochu.

## Kdy repozitář přestává být jen archiv zdrojáků

Bezpečnostně jsou nejrizikovější hlavně tyto vzory:

- běžící aplikace čte soubory přímo z working tree,
- nasazení nebo synchronizace sahá do repozitáře, který může měnit neprivilegovaný uživatel,
- produkční host drží `.git` i s historií konfigurace,
- staré commity stále obsahují platná hesla, klíče nebo servisní účty,
- hooky, playbooky nebo build kroky se spouští s vyššími právy, než má uživatel, který může měnit jejich vstup.

Klíčová otázka tedy nezní jen `unikl repozitář?`, ale hlavně `co všechno mu prostředí věří`.

## Typické scénáře zneužití

### 1. Repozitář přímo ovládá běžící aplikaci

Na Bitlabu nestačilo, že GitLab odhalil zdrojáky. Důležité bylo až to, že kompromitovaný uživatel mohl změnit soubor `profile/index.php` a aplikace tuto změnu prakticky sama převzala. Ve chvíli, kdy je repo současně zdroj pravdy i zdroj běžícího kódu, není potřeba hledat samostatné RCE v aplikaci. Stačí změnit to, co se má spustit.

To je jeden z nejnebezpečnějších provozních antipatternů: vývojový přístup se tím přímo mění v produkční code execution.

### 2. Historie pořád obsahuje živé tajemství

Seal ukazuje jiný problém. Uniklý `tomcat-users.xml` nebyl v aktuálním stavu aplikace, ale v historii projektu. Bezpečnostně je to ale téměř totéž, jako kdyby tam ležel dnes. Jakmile stejné heslo stále funguje v Tomcat Manageru, historie repozitáře není „starý odpad“, ale aktivní credential store.

Stejný princip se opakoval i jinde:

- v Secret zůstalo v Git historii produkčně použitelné JWT tajemství,
- v Zettě ležela pod `/etc/rsyslog.d/.git` stará databázová konfigurace s heslem `postgres`,
- v řadě prostředí se v commitech objevují `.env`, helper skripty nebo servisní konfigurace, které už z hlavní větve zmizely, ale z hlediska útočníka pořád fungují.

To je důležitý rozdíl proti běžnému úniku zdrojáku. Historie neukazuje jen to, jak vývoj probíhal. Často uchovává přesně ta tajemství, která už administrátor považuje za dávno odstraněná.

### 3. Privilegovaná automatika důvěřuje writable repozitáři

Na Bitlabu byl root ve skutečnosti až důsledkem toho, že administrativní `sudo git pull` běžel nad repozitářem, do kterého mohl zapisovat kompromitovaný uživatel. Jakmile je writable i `.git/hooks`, nejde o nevinný update, ale o spuštění cizí logiky s vyššími právy.

Podobný princip se objevuje i mimo čistý Git:

- Seal měl zálohovací playbook a později příliš široké `sudo` nad `ansible-playbook`,
- Zetta těžila z toho, že historická konfigurační vrstva v `/etc` pořád nesla důvěryhodná data pro další privilegovaný subsystém,
- jakmile deployment nebo maintenance úloha přebírá vstup ze souborů, které může ovlivnit méně privilegovaný účet, vzniká z ní velmi přímočarý privesc vektor.

Nejde tedy jen o hooky. Jde o celý vzorec `mohu měnit to, čemu později věří privilegovaná automatika`.

## Co při auditu skutečně hledat

U podobných prostředí je užitečné jít po konkrétních otázkách.

### Kde všude je přítomná historie

- leží `.git` v deploynuté aplikaci,
- existují repozitáře i pod `/etc`, v zálohách nebo v administračních adresářích,
- jsou v hostu checkouty vývojových větví nebo staré exporty release.

### Co z historie zůstává platné

- fungují stará hesla pro Tomcat, Splunk, JWT, databázi nebo servisní účet,
- obsahují commity interní URL, které jsou dostupné jen z hostu,
- jsou v historii playbooky, hooky nebo skripty, které se stále používají.

### Kdo může měnit vstup do deploymentu

- běží `git pull`, `ansible-playbook`, synchronizace nebo build pod `sudo`,
- pracuje automatika s working tree místo s immutable artefaktem,
- může kompromitovaný uživatel zapisovat do hooků, playbooků, šablon nebo konfiguračních fragmentů.

### Jaká je skutečná hranice mezi vývojem a produkcí

- liší se vůbec přihlašovací údaje mezi vývojovým a produkčním prostředím,
- používá se stejný repozitář jako zdroj pro review i pro reálný provoz,
- má produkce vlastní tajemství mimo verzovací systém, nebo je bere z commitnutých souborů.

## Praktické signály, že je prostředí nebezpečně navržené

Mezi varovné indikátory patří hlavně:

- produkční host umí `git status`,
- v nasazeném adresáři je `.git`,
- administrátor opravuje produkci ručním `git pull`,
- tajemství se „odstraňují“ jen novým commitem bez rotace,
- deployment účty mají přístup k více rolím, než nutně potřebují,
- konfigurace infrastruktury je verzovaná přímo na cílovém hostu bez oddělení od runtime.

Každý jednotlivý bod ještě nemusí znamenat kompromitaci. Ale jejich kombinace obvykle znamená, že vývojová vrstva už není oddělená od produkční důvěry.

## Obrana a hardening

### 1. Oddělit repozitář od runtime

Produkce by ideálně neměla běžet z working tree. Bezpečnější model je buildnout artefakt mimo cílový host a nasazovat jen to, co má skutečně běžet.

### 2. Nenechávat `.git` v produkci

To platí i pro konfigurační adresáře. `.git` pod `/etc`, v backup adresáři nebo ve webrootu znamená, že na cíli leží nejen současný stav, ale i historie změn a často i stará tajemství.

### 3. Rotovat tajemství po každém úniku z historie

Smazání z posledního commitu nic neřeší. Pokud se jednou heslo objevilo v repozitáři a existuje šance, že se k němu někdo dostal, je potřeba ho považovat za kompromitované.

### 4. Zakázat privilegované operace nad writable repozitářem

`sudo git pull`, root build, automatické načítání hooků nebo playbooků z adresáře, který může měnit neprivilegovaný účet, je potřeba považovat za designovou chybu, ne za administrativní pohodlí.

### 5. Auditovat deployment trust chain

Nestačí kontrolovat aplikaci samotnou. Je potřeba projít i to, odkud se berou:

- konfigurační šablony,
- hooky a post-deploy skripty,
- servisní účty,
- tajemství v CI/CD,
- maintenance playbooky a synchronizační úlohy.

Právě tudy často vede kratší cesta k privilegovanému kódu než přes samotnou aplikaci.

## Shrnutí klíčových poznatků

- Repozitář není jen zdroj kódu. V mnoha prostředích je i zdrojem důvěry pro deployment, konfiguraci a správu.
- Historie commitů zůstává bezpečnostně relevantní tak dlouho, dokud v ní leží platná tajemství nebo stále používané provozní informace.
- Největší dopad vzniká ve chvíli, kdy privilegovaná automatika přebírá vstup z repozitáře nebo working tree, který může útočník ovlivnit.
- Obrana nezačíná až u web serveru. Začíná už u toho, jak ostře jsou oddělené vývoj, deploy a produkční runtime.

## Co si odnést do praxe

- Pokud útočník získá přístup k repozitáři, nehledej jen hesla. Hledej i to, co se z repozitáře samo nasazuje, spouští nebo synchronizuje.
- Starý commit s tajemstvím je stále únik. Bez rotace credentialů se z historie stává plnohodnotný credential store.
- `git pull`, hooky, playbooky a konfigurační repozitáře jsou součást bezpečnostní hranice. Jakmile do nich může zapisovat kompromitovaný účet, hranice se rozpadá.
