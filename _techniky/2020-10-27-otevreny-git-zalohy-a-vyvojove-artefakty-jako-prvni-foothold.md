---
layout: post
author: Tomáš Matějíček
title: "Otevřený .git, zálohy a vývojové artefakty jako první foothold"
date: 2020-10-27
tags: web git backups sourcecode
---
## Úvod a kontext

Spousta útoků nezačíná exploitem, ale čtením. Veřejně dostupný `.git`, archiv zdrojáků, starý backup webu nebo testovací soubor ponechaný ve webrootu často prozradí víc než samotná aplikace. Ne proto, že by šlo o "zranitelnost vyšší úrovně", ale proto, že v těchto artefaktech bývají přesně ty vztahy mezi službami, účty a tajemstvími, které veřejné rozhraní skrývá.

Je užitečné tenhle vzorec pojmenovat explicitně. První foothold si mnoho lidí automaticky spojuje s RCE, ale v praxi často přijde jinak: nejdřív unikne kontext, potom přístupové údaje nebo interní logika a teprve z toho vznikne skutečný shell. Právě na tenhle pattern se článek zaměřuje.

## Proč jsou tyto artefakty tak cenné

Živá aplikace obvykle ukazuje jen to, co chce ukázat. Zdroják, záloha nebo repozitář naopak často odhalí:

- neveřejné endpointy,
- interní hostname a služby,
- komentáře a dočasné obchvaty,
- přístupové údaje a tokeny,
- historii změn, ve které jsou i smazaná tajemství,
- vazbu mezi webem, shell účtem, databází a pomocnými skripty.

Jinými slovy: artefakt útočníkovi zkracuje cestu od "vidím login formulář" k "chápu, jak systém opravdu funguje".

## Co všechno sem patří

Když se řekne otevřený `.git`, většina lidí si představí jen jeden konkrétní problém. Ve skutečnosti jde o širší rodinu artefaktů:

- veřejně čitelný `.git/`,
- zálohy typu `files.zip`, `html.tar.gz`, `backup.zip`, `appserver.zip`,
- editorové nebo deployment kopie typu `index.php~`, `.bak`, `.old`,
- exporty databáze nebo aplikačních konfigurací,
- testovací nebo migrační skripty ponechané ve webrootu,
- archiv domácího adresáře nebo aplikace přiložený k issue, dokumentaci nebo downloadu.

Společný jmenovatel je jednoduchý: systém zveřejní něco, co nevzniklo pro veřejný provoz, ale přesto zůstalo dosažitelné.

## Co v takových artefaktech hledat

Nejhorší strategie je stáhnout archiv a bezcílně v něm listovat. Prakticky se vyplatí hledat několik konkrétních tříd informací.

### 1. Tajemství a klíče

Nejde jen o hesla v plaintextu. Hledejte i:

- JWT signing secret,
- databázové connection stringy,
- SMTP hesla,
- API tokeny,
- SSH klíče,
- privátní certifikáty,
- interní účty pro deployment a automatizaci.

Právě tady se často láme rozdíl mezi "jen jsem stáhl zdroják" a "mám přímý přístup do další vrstvy".

### 2. Neveřejné endpointy a hostname

Zdrojáky a konfigurace často obsahují:

- interní subdomény,
- localhost-only endpointy,
- cesty k admin rozhraním,
- odkazy na vedlejší služby typu Redis, registry, FTP nebo API gateway.

To bývá klíčové hlavně ve chvíli, kdy veřejný web sám vypadá chudě nebo vrací jen `403`, zatímco uniklá konfigurace prozradí další směr útoku.

### 3. Historii a smazané stopy

Obzvlášť u `.git` je důležité neomezit se na aktuální stav. Starý commit může obsahovat:

- tajemství, která už byla "odstraněná",
- bezpečnostní obchvat ponechaný během vývoje,
- starší implementaci endpointu s lepším vysvětlením logiky,
- dočasnou diagnostiku, která později zmizela z UI, ale ne z historie.

Tohle je častý zdroj omylu na straně obrany: secret se smaže z hlavní větve a tým má pocit, že problém je vyřešený. Není.

### 4. Vazby mezi účty a službami

Při hledání footholdu je mimořádně cenné pochopit, jak jsou propojené:

- webový účet a shell účet,
- databázový uživatel a systémový uživatel,
- vývojový vhost a FTP deploy,
- záloha aplikace a SSH klíč v domácím adresáři.

Jeden izolovaný credential leak ještě nemusí stačit. Jakmile ale artefakt prozradí i to, kde credential použít, stane se z něj přímý vstup.

## Typický řetězec od artefaktu k footholdu

Praktický postup často vypadá takto:

1. veřejný web neukáže nic zásadního,
2. vedlejší soubor nebo `.git` odhalí zdroják,
3. zdroják prozradí secret, interní hostname nebo backup cestu,
4. další vrstva otevře platný účet, administraci nebo server-side primitivum,
5. teprve pak vznikne shell.

To je přesně důvod, proč je chyba hodnotit tyto artefakty jen jako informační leak. Informace totiž často nejsou konečný dopad, ale mezikrok k plnému kompromisu.

Na [Playeru](/player) se to ukazuje velmi čistě. Nejdůležitější indicie nepřišla z hlavního webu, ale ze záložního souboru `dee8dc8a47256c64630d803a4c40786c.php~`, který vydal JWT signing secret i logiku launcheru. Jeden jediný backup artefakt tak otevřel další aplikační větev a zkrátil cestu k plnohodnotnému footholdu.

## Proč staré a "už nepoužívané" soubory pořád bolí

Obrana často spoléhá na dvě mylné domněnky:

- "ten soubor je starý, takže už nevadí",
- "to heslo už v aplikaci není, protože jsme ho odstranili z kódu".

V praxi to nefunguje z několika důvodů:

### Tajemství se často nerevokují

Smazat secret z kódu není totéž jako secret změnit. Pokud heslo nebo token zůstal platný i po smazání ze zdrojáku, starý commit je pořád aktuální kompromis.

### Stará konfigurace vysvětluje novou logiku

I když staré tajemství nefunguje, může starý commit vysvětlit:

- názvy claimů v JWT,
- jména interních endpointů,
- strukturu requestu,
- způsob deploye nebo logování,
- vazbu mezi více hosty nebo službami.

Starý artefakt tak může pomoci zneužít zcela současnou chybu.

### Backup často obsahuje víc než produkce

Archiv aplikace občas neobsahuje jen to, co běží venku, ale i:

- `.env`,
- lokální helper skripty,
- testovací routy,
- komentáře,
- přiložené utility,
- exporty dalších služeb.

Záloha proto může mít vyšší informační hodnotu než běžící instance.

## Jak s nálezem pracovat disciplinovaně

Jakmile se k takovému artefaktu dostanete, je dobré nezkoušet hned všechno na všechno. Větší hodnotu má mapování:

- které tajemství patří ke které službě,
- které hostname nebo endpointy jsou veřejné a které interní,
- které účty působí provozně a které uživatelsky,
- které části aplikace pracují s lokálními soubory nebo shellovými příkazy.

Právě to odděluje náhodné střílení credentialů od smysluplné analýzy. Dobrý foothold často vznikne ne z množství nálezů, ale ze správného propojení dvou nebo tří z nich.

## Jak tomu předcházet

Obrana musí řešit víc než jen "zakázat listing adresářů".

### 1. Zabránit publikaci artefaktů

Web server a reverse proxy musí explicitně blokovat:

- `.git/`,
- backup přípony,
- editorové kopie,
- migrační a pomocné soubory mimo nutný runtime.

To je první vrstva, ale ne jediná.

### 2. Oddělit build a runtime

Do webrootu nebo image nemá proudit celý repozitář se vším všudy. Čím čistší artefakt se nasazuje, tím menší šance, že se ven dostane `.git`, test data nebo vývojové utility.

### 3. Po úniku rotovat, ne jen mazat

Jakmile se secret objevil v repozitáři, commitu nebo záloze, musí se považovat za kompromitovaný. Smazání souboru nestačí.

### 4. Auditovat historii i exporty

Týmy často kontrolují jen produkční konfiguraci. Prakticky stejně důležité je ale:

- co je v historii repozitáře,
- co se ukládá do backupů,
- co se přikládá do exportů a support archivů,
- co se dostává do obrazů a release balíčků.

### 5. Omezit reuse mezi vrstvami

I když nějaký artefakt unikne, dopad výrazně roste až tehdy, když stejné tajemství funguje v SSH, databázi, aplikaci i deploy nástroji. Segmentace credentialů proto zůstává klíčová.

## Shrnutí klíčových poznatků

- Otevřený `.git`, záloha nebo vývojový artefakt nejsou jen pasivní informační leak; často představují první skutečný foothold.
- Největší hodnotu mívají tajemství, interní hostname, historie commitů a vazby mezi službami.
- Staré nebo "smazané" informace zůstávají nebezpečné tak dlouho, dokud nejsou i revokované nebo architektonicky oddělené.
- Praktický dopad nevzniká z jedné nalezené položky, ale z propojení artefaktu s další vrstvou systému.

## Co si odnést do praxe

- Při auditu webu se vyplatí hledat i to, co na web nepatří: `.git`, backupy, `~` soubory, exporty a vývojové utility.
- Při obraně nestačí tajemství "uklidit" z aktuální verze. Pokud se někdy objevilo v repozitáři nebo archivu, je potřeba ho rotovat.
- Největší riziko těchto artefaktů není v tom, že prozradí jeden detail, ale že útočníkovi vysvětlí celý vztah mezi aplikací, účty a interní infrastrukturou.
