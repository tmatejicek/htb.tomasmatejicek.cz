---
layout: post
author: Tomáš Matějíček
title: "Špatně vystavené datové, storage a cloud-like služby"
date: 2020-11-13
tags: storage redis nfs registry s3
---
## Úvod a kontext

Mnoho prostředí obsahuje služby, které se neberou jako "hlavní aplikace". Jsou to úložiště, registry, cache, exporty, backup share nebo interní cloud-like API. Právě proto bývají podceněné. Tým je vnímá jako pomocnou vrstvu, která přece nemá být z internetu důležitá. Jenže útočník to vidí opačně: právě tato vrstva často drží tajemství, cizí data nebo přímo možnost něco zapsat do prostoru, který už aplikace sama považuje za důvěryhodný.

To je důvod, proč má smysl spojit Redis, NFS backupy, Docker registry, S3-like storage, DynamoDB endpointy nebo rsync do jednoho článku. Ne proto, že by šlo o stejnou technologii. Ale proto, že sdílejí stejný bezpečnostní vzorec: interní nebo pomocná datová služba se ocitla ve špatné zóně, bez autentizace nebo s většími právy, než odpovídá jejímu účelu.

## Proč jsou tyto služby tak cenné

Veřejná aplikace často ukáže jen omezený výřez systému. Datové a storage služby naopak obsahují:

- skutečná provozní data,
- zálohy a historické artefakty,
- tajemství a konfigurační soubory,
- image vrstvy nebo build výstupy,
- možnost zápisu do prostoru, který dál používá aplikace nebo SSH.

Právě proto bývá špatně vystavené storage rozhraní často hodnotnější než samotná titulní stránka webu.

## Tři základní typy dopadu

Ne každá vystavená storage služba znamená totéž. Prakticky se vyplatí rozlišit tři úrovně.

### 1. Read-only leak

Služba sice nepovolí zápis, ale vrátí:

- databázi,
- logy,
- image vrstvu,
- historické konfigurace,
- seznam bucketů nebo tabulek,
- nebo cizí domácí adresář.

To stačí na tajemství, hashe, klíče a velmi často i na přímý přístup do jiné služby.

### 2. Write primitive

Ještě nebezpečnější je situace, kdy lze do úložiště něco zapisovat:

- veřejný bucket přijme soubor,
- rsync export dovolí podstrčit `.ssh/authorized_keys`,
- Redis dovolí změnit cestu a zapsat `authorized_keys`,
- storage vrstva uloží obsah, který pak web nebo renderer sám načte.

V tu chvíli už nejde jen o leak. Jde o možnost vložit data do důvěryhodného workflow.

### 3. Nepřímý administrativní přístup

Některé služby působí jako datová vrstva, ale ve skutečnosti jsou velmi blízko správě:

- Docker registry,
- management Webmin za reuse heslem získaným přes storage artefakt,
- cloud-like API, které samo rozhoduje o publikační cestě souborů,
- nebo share s backupem, z něhož vypadne přímý admin credential.

Tady se storage mění z pasivního úložiště v most k plnohodnotnému administrativnímu přístupu.

## Jak tenhle vzorec vypadal v různých případech

Na [Bucketu](/bucket) běželo S3-like a DynamoDB rozhraní bez autentizace. To samo o sobě stačilo k vyčtení uživatelských hesel z tabulky a zároveň k zápisu souboru do bucketu `adserver`, odkud ho hlavní aplikace sama publikovala. Je to učebnicový příklad spojení read a write primitiva v jedné "pomocné" službě.

Na [Remote](/remote) stál jiný případ na veřejném NFS exportu `/site_backups`. Nebylo třeba obcházet aplikaci ani získávat první login. Záloha sama vydala databázi Umbraco a provozní logy, z nichž šlo složit aktuální admin heslo. NFS tedy neposkytlo shell přímo, ale poskytlo přesně ta data, která shell umožnila.

Na [Registry](/registry) nebyla hlavní slabina na webu, ale v Docker registry odhalené TLS certifikátem. Stažení image vrstev odhalilo shell history a passphrase k soukromému klíči. Tady je dobře vidět, že registry není jen přepravní sklad pro image. Je to archiv provozních artefaktů, které mohou být bezpečnostně cennější než běžící kontejner.

Do stejné rodiny patří i [Postman](/postman), kde veřejný Redis bez autentizace nevydal data pasivně. Dovolil přímý zápis do `/var/lib/redis/.ssh/authorized_keys`, a tím okamžitý SSH foothold. Je to krásný příklad služby, kde je write primitive ve skutečnosti rychlejší než jakýkoli read.

Na [Zettě](/zetta) další případ využil rsync export domácího adresáře uživatele. Jakmile se podařilo získat heslo k modulu, nešlo jen o čtení souborů. Bylo možné do home directory i zapisovat a podstrčit vlastní `authorized_keys`. To už není "sdílený souborový prostor". To je de facto vzdálený SSH onboarding pro útočníka.

## Co mají tyto služby společné

Na povrchu jsou rozdílné, ale spojuje je několik vzorců.

### Služba je vnímaná jako pomocná, ne bezpečnostní hranice

Právě proto bývá méně chráněná než veřejná aplikace:

- běží bez autentizace,
- zůstane v defaultní konfiguraci,
- publikuje interní data,
- nebo je dostupná z širší sítě, než by měla být.

### Obsah služby už někdo jiný považuje za důvěryhodný

To je klíčové hlavně u write primitive:

- web obslouží soubor z bucketu,
- SSH přijme klíč z domácího adresáře,
- kontejner nebo provozní tým důvěřuje image vrstvě,
- renderer čte obsah z tabulky nebo storage prostoru.

Storage tedy sama nemusí umět spouštět kód. Stačí, že jí věří jiná vrstva.

### Data jsou historická, ale stále platná

Zálohy, image vrstvy, logy nebo staré klíče se často podceňují, protože "už nejsou aktivně používané". Jenže:

- tajemství nemuselo být revokováno,
- passphrase je znovu použitá jinde,
- log stále obsahuje aktuální heslo,
- nebo obraz vrstvy pořád vysvětluje interní architekturu.

## Jak takové služby hodnotit při analýze

Při průzkumu i při review se vyplatí projít tři otázky.

### Co přesně jde číst

- tajemství,
- konfigurační soubory,
- zdrojáky,
- logy,
- obrazové vrstvy,
- domácí adresáře,
- nebo jen metadata.

### Co přesně jde zapisovat

- obyčejný soubor,
- objekt do bucketu,
- klíč do home directory,
- databázový záznam, který dál čte jiná služba,
- nebo konfiguraci samotné storage služby.

### Kdo této vrstvě dál věří

- SSH,
- webserver,
- PDF renderer,
- CMS,
- build/deploy systém,
- nebo administrativní nástroj.

Třetí otázka bývá rozhodující. Bez ní je storage jen leak. S ní se z ní stává přímý execution nebo credential pivot.

## Obrana a bezpečnější návrh

### 1. Helper služby chránit stejně přísně jako hlavní aplikaci

NFS export, Redis, rsync, registry nebo S3-compatible endpoint nejsou vedlejší technická drobnost. Jsou to produkční rozhraní s vlastním bezpečnostním dopadem.

### 2. Oddělit read a write přístup

Tam, kde není zápis nutný, nesmí existovat. Write primitive má téměř vždy výrazně vyšší dopad než samotný read-only leak.

### 3. Nepublikovat artefakty určené pro vnitřní provoz

Image vrstvy, zálohy, shell history, logy a exporty musejí být chráněné stejně jako produkční databáze. Často v nich leží přesně ta tajemství, která už z běžící aplikace zmizela.

### 4. Nepovažovat uložený obsah automaticky za důvěryhodný

To platí hlavně pro:

- bucket obsluhovaný webem,
- rsync export domácího adresáře,
- tabulku čtenou rendererem,
- nebo Redis/registry vrstvu, kterou dál používá jiný privilegovaný proces.

### 5. Segmentovat sítě a identity

Storage služba nemá být dostupná ze stejné zóny jako veřejný web, pokud to není nezbytné. A tajemství získané z jedné vrstvy nemá automaticky fungovat i na SSH, WinRM nebo admin nástroji.

## Shrnutí klíčových poznatků

- Špatně vystavené datové a storage služby často představují rychlejší cestu k tajemstvím než veřejná aplikace.
- Praktický dopad je potřeba hodnotit podle toho, co lze číst, co lze zapisovat a kdo této vrstvě dál důvěřuje.
- Redis, NFS, Docker registry, S3-like endpointy nebo rsync nejsou nahodilá směs. Jsou to různé projevy stejného problému: pomocná datová vrstva se ocitla ve špatné zóně nebo bez správné autentizace.
- Obrana stojí na segmentaci, omezení write přístupů a na tom, že backup, registry a storage artefakty nejsou považované za méně citlivé než produkční data.

## Co si odnést do praxe

- Při průzkumu infrastruktury se vždy ptejte, jaké vedlejší datové služby jsou dostupné mimo hlavní aplikaci. Často právě tam leží skutečný první foothold.
- Read-only leak je vážný, ale write primitive bývá rozhodující. Jakmile lze do storage něco podstrčit a jiná vrstva tomu věří, dopad dramaticky roste.
- Storage a registry služby nesmějí stát na implicitní důvěře. Pokud jejich obsah používá web, SSH nebo deploy nástroj, je potřeba je chránit stejně přísně jako samotné administrativní rozhraní.
