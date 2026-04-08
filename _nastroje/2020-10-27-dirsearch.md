---
layout: post
author: Tomáš Matějíček
title: "Dirsearch"
date: 2020-10-27
tags: nastroje web enumeration content-discovery directories files
---
## Úvod a kontext

`dirsearch` je v tomto projektu praktický hlavně ve chvíli, kdy už je známý správný host, vhost nebo cesta a je potřeba rychle zjistit, co dalšího na webu existuje. Není to nástroj na pochopení celé aplikace. Je to nástroj na objevování vstupních bodů, vedlejších cest a zapomenutých souborů.

Na [Magicu](/magic) ukáže `upload.php` a tím rovnou naznačí, že aplikace umí upload a za ním drží login. Na [Previsi](/previse) odhalí `accounts.php`, `files.php` nebo `download.php`, tedy cesty, které na první pohled nejsou z webu vidět. Na [Cerealu](/cereal) pomůže na vedlejším hostu `source.cereal.htb` najít `/.git`, což pak úplně změní směr útoku. To je jeho praktická role: nepřináší exploit, ale vytáhne na povrch věci, které se dál dají číst nebo zneužít.

## Co `dirsearch` v praxi řeší

`dirsearch` je content discovery nástroj. Prakticky odpovídá na otázky:

- jaké cesty, soubory nebo endpointy aplikace vystavuje,
- které z nich nejsou v navigaci,
- kde je login, upload, admin rozhraní, backup nebo technický artefakt,
- a zda vedle hlavního webu neexistuje další výrazně zajímavější povrch.

Jeho hodnota neleží v samotném množství nalezených cest, ale v tom, že rychle odfiltruje ty, které stojí za další analýzu.

## Nejčastější scénáře využití v tomto projektu

### Nalezení relevantní funkce aplikace

Na [Magicu](/magic) je první praktický výsledek velmi přímočarý. Zvenku jsou jen Apache a SSH, takže dává smysl systematicky projít web a hledat něco, co nese reálný dopad:

```bash
./dirsearch/dirsearch.py -u http://$IP -e php -x 403 -r
```

Výstup ukáže:

```text
/upload.php  ->  login.php
```

To je velmi cenná informace už sama o sobě:

- aplikace umí upload,
- upload je schovaný za autentizací,
- a první foothold bude téměř jistě webový, ne přes SSH nebo jinou službu.

Tady `dirsearch` neřeší exploit. Jen přesně označí funkci, která má vysokou návratnost dalšího času. Širší kontext upload řetězců je i v článku [Nebezpečné uploady: polygloty, WAR deploy a plugin upload](/techniky/nebezpecne-uploady-polygloty-war-deploy-a-plugin-upload).

### Odhalení neveřejných, ale funkčních cest

Na [Previsi](/previse) je web na první pohled jen přihlašovací formulář. `dirsearch` ale rychle ukáže, že aplikace má další cesty:

```bash
./dirsearch/dirsearch.py -u http://$IP -e php -x 403 -r
```

```text
/accounts.php
/config.php
/download.php
/files.php
```

To je prakticky zásadní rozdíl proti situaci, kdy by se útočník zastavil u login stránky. Najednou je zřejmé, že aplikace má:

- účetní logiku,
- práci se soubory,
- a pravděpodobně i interní konfiguraci.

Právě to z ní dělá mnohem bohatší cíl, než naznačuje samotné UI.

### Odhalení vývojových artefaktů nebo vedlejšího hostu

Na [Cerealu](/cereal) samotný hlavní web nevrací skoro nic, ale vedlejší host `source.cereal.htb` už ano. `dirsearch` tam najde mimo jiné `/.git/logs/HEAD`:

```bash
./dirsearch/dirsearch.py -u https://cereal.htb/ -e php -x 403 -r
./dirsearch/dirsearch.py -u https://source.cereal.htb/ -e php -x 403 -r
```

To je typický okamžik, kdy se enumerace mění v rozhodnutí:

- hlavní web nevypadá slibně,
- vedlejší host vrací vývojový artefakt,
- a další čas má smysl investovat právě tam.

Tento pattern úzce souvisí i s článkem [Otevřený `.git`, zálohy a vývojové artefakty jako první foothold](/techniky/otevreny-git-zalohy-a-vyvojove-artefakty-jako-prvni-foothold).

### Objevování vedlejších administrativních a servisních cest

Na [Admireru](/admirer), [Jarvisi](/jarvis), [Bashed](/bashed), [BountyHunteru](/bountyhunter) nebo [Buckettu](/bucket) `dirsearch` opakovaně slouží ke stejnému typu práce:

- objevit admin adresář,
- narazit na `phpMyAdmin`,
- najít zálohovou nebo upload cestu,
- nebo potvrdit, že vedle hlavního webu existuje další technický povrch.

Právě v tom je jeho praktická síla. Neříká "tady je exploit", ale rychle označí, která větev webu má pro útočníka největší návratnost.

## Kdy je `dirsearch` nejlepší volba

Největší smysl dává tehdy, když:

- už znáte správný host nebo vhost,
- chcete najít neveřejné cesty a soubory,
- a další analýza závisí na tom, zda existuje login, upload, backup nebo admin rozhraní.

V těchto situacích je rychlý a velmi levný. Hlavně ale dává strukturu dalšímu webovému průzkumu.

## Co `dirsearch` neumí vyřešit za vás

`dirsearch`:

- sám nepozná obchodní logiku aplikace,
- neřekne, která nalezená cesta je skutečně zneužitelná,
- a nevyřeší host discovery ani technologickou identifikaci.

Na [Magicu](/magic) je užitečný proto, že `upload.php` okamžitě mění pracovní hypotézu. Na [Previsi](/previse) proto, že odhalí neveřejné cesty. Na [Cerealu](/cereal) proto, že najde vývojový artefakt. Bez čtení kontextu by byl jeho výstup jen seznam URL.

## Nejčastější chyby při použití

### Záměna za úplnou webovou enumeraci

`dirsearch` řeší obsahové cesty. Neřeší automaticky:

- správné vhosty,
- technologii aplikace,
- auth logiku,
- ani to, zda je nalezený endpoint opravdu důležitý.

### Mechanické čtení výstupu bez prioritizace

V praxi má větší hodnotu jeden:

- `upload.php`,
- `/.git/`,
- `phpMyAdmin`,
- nebo `download.php`

než desítky statických assetů. Úspěch tedy neleží v počtu nalezených odpovědí, ale ve správném výběru.

### Použití proti špatnému cíli

Pokud ještě není jisté, který vhost je relevantní, bývá důležitější nejdřív vyřešit host discovery nebo fingerprinting. `dirsearch` má největší cenu až ve chvíli, kdy je zřejmé, že prochází správný povrch.

## Nejčastější praktické scénáře

### Potřebuji najít login, upload nebo admin cestu

To je [Magic](/magic), [Jarvis](/jarvis) nebo [Bashed](/bashed).

### Potřebuji zjistit, zda aplikace skrývá další neveřejné cesty

To je [Previse](/previse).

### Potřebuji ověřit, zda vedlejší host neobsahuje vývojové artefakty

To je [Cereal](/cereal).

## Související články v projektu

- Rozbory: [Magic](/magic), [Previse](/previse), [Cereal](/cereal), [Admirer](/admirer), [Bucket](/bucket), [Jarvis](/jarvis)
- Techniky: [Otevřený `.git`, zálohy a vývojové artefakty jako první foothold](/techniky/otevreny-git-zalohy-a-vyvojove-artefakty-jako-prvni-foothold), [Nebezpečné uploady: polygloty, WAR deploy a plugin upload](/techniky/nebezpecne-uploady-polygloty-war-deploy-a-plugin-upload), [Exponovaná debug, maintenance a administrační rozhraní](/techniky/exponovana-debug-maintenance-a-administracni-rozhrani)

## Co si odnést do praxe

- `dirsearch` je nejpraktičtější jako nástroj na rychlé odhalení neveřejných webových cest, které mění další hypotézu útoku.
- Největší hodnotu má tehdy, když už víte, který host nebo vhost má smysl procházet.
- Samotný výstup nestačí. Skutečný přínos vzniká až tím, že z nalezených cest vyberete ty, které nesou technologii, citlivá data nebo další vstupní bod.
