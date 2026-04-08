---
layout: post
author: Tomáš Matějíček
title: "Searchsploit"
date: 2020-01-04
tags: nastroje exploits enumeration vulnerability-research
---
## Úvod a kontext

`searchsploit` je v praxi nejužitečnější ve chvíli, kdy už existuje rozumná hypotéza o technologii nebo verzi služby a je potřeba rychle ověřit, jestli pro ni existuje veřejně známá a lokálně dostupná exploit reference. Není to "nástroj na hacknutí cíle". Je to most mezi enumerací a rozhodnutím, jakou exploitační cestu má smysl dál ověřovat.

Na [OpenAdminu](/openadmin) ukáže odpovídající command injection pro OpenNetAdmin, na [Nibbles](/nibbles) pomůže vybrat správný exploit z více kandidátů a na [Horizontallu](/horizontall) dává smysl až po port-forwardu, protože teprve potom je interní Laravel skutečně dosažitelný. Právě to je praktická hodnota `searchsploitu`: neříká jen "něco existuje", ale pomáhá rozhodnout, zda daný exploit odpovídá konkrétnímu povrchu, verzi a podmínkám.

## Co tenhle nástroj v praxi řeší

Když už víte:

- jaká technologie běží,
- jaká verze nebo rodina produktu je pravděpodobná,
- a jaký typ chyby dává smysl,

pak je `searchsploit` rychlý způsob, jak zjistit, zda už podobnou cestu někdo zpracoval v Exploit-DB.

Prakticky řeší hlavně tři situace:

- rychlé vyhledání kandidátních exploitů podle produktu nebo verze,
- otevření lokální kopie exploitu k přečtení a úpravě,
- a rozhodnutí, zda konkrétní veřejná reference odpovídá realitě cíle.

## Nejčastější scénáře využití v tomto projektu

### Banner nebo identifikace technologie ukáže pravděpodobný produkt

Na [OpenAdminu](/openadmin) nejdřív `whatweb` identifikuje OpenNetAdmin. Teprve potom dává smysl ověřit, zda pro něj existuje známá RCE cesta:

```bash
searchsploit -w opennetadmin
```

Výsledek vrátí konkrétní command injection pro `18.1.1`, což je přesně ten typ hypotézy, který stojí za další ověření. Důležitý není samotný počet výsledků. Důležité je, že enumerace a produktová identifikace už předem zúžily prostor na realistického kandidáta.

### Je potřeba vybrat správný exploit z více možností

Na [Nibbles](/nibbles) `searchsploit` nevrátí jen jeden výsledek:

```bash
searchsploit nibbleblog
```

Vrací více položek, ale prakticky důležitá je otázka, která z nich odpovídá tomu, co už o aplikaci víte. Tady se ukáže, že upload exploit pro `Nibbleblog 4.0.3` dává mnohem větší smysl než starší SQL injection reference.

To je přesně situace, kde `searchsploit` nepoužíváte jako "spustím první výsledek", ale jako filtr kandidátů.

### Exploit existuje, ale rozhodující je dostupnost a kontext

Na [Horizontallu](/horizontall) je důležitý jiný detail. `searchsploit laravel` má smysl až ve chvíli, kdy už je interní Laravel dostupný přes port-forward. Bez tohoto kroku by exploit sice existoval, ale nebylo by ho kam aplikovat.

```bash
searchsploit laravel
searchsploit -p 49424
```

Tady je dobře vidět, že `searchsploit` nepomáhá jen hledat "jestli existuje CVE". Pomáhá ověřit, zda exploitační hypotéza sedí na reálné umístění služby v daném řetězci.

### Staré služby a rychlé ověření historické hypotézy

Na [Lame](/lame), [Armageddonu](/armageddon) nebo [Beepu](/beep) má `searchsploit` ještě jinou roli. Když banner nebo produkt už silně naznačuje známý problém, `searchsploit` slouží jako rychlá lokální reference:

- jaký exploit pro danou verzi existuje,
- jestli je veřejně dostupný,
- a jaké má předpoklady.

Právě tady se potkává s článkem [Legacy infrastruktura, kde banner prakticky rozhodne exploit](/techniky/legacy-infrastruktura-kde-banner-prakticky-rozhodne-exploit). `Searchsploit` sice není ten, kdo banner přečte správně, ale velmi rychle ověří, jaké realistické cesty pro takový banner existují.

## Jak `searchsploit` používat prakticky

Nejužitečnější pracovní sled bývá:

### 1. Nejdřív identifikace, až potom hledání

Nejprve musí přijít:

- `nmap`,
- `whatweb`,
- nalezená verze v admin panelu,
- nebo jiné technické potvrzení technologie.

Až potom má smysl:

```bash
searchsploit produkt-nebo-verze
```

### 2. Kandidáty číst, ne jen spouštět

Prakticky je často cennější podívat se na lokální kopii exploitu než ho hned spustit:

```bash
searchsploit -p 49424
```

Lokální reference rychle ukáže:

- jaké jsou předpoklady exploitu,
- jestli jde o autentizovanou nebo neautentizovanou cestu,
- jaký endpoint používá,
- a zda odpovídá vašemu kontextu.

### 3. Rozhodnout, jestli exploit sedí na skutečný povrch

To je nejdůležitější praktická otázka. Na [Horizontallu](/horizontall) exploit dává smysl až po port-forwardu. Na [Nibbles](/nibbles) dává smysl až po zjištění správné aplikace a cesty k admin rozhraní. Na [OpenAdminu](/openadmin) zase až po potvrzení produktu ONA.

## Co `searchsploit` neumí vyřešit za vás

`Searchsploit` sám nerozhodne:

- jestli je verze skutečně zranitelná,
- jestli bannery nelžou,
- jestli aplikace není patchovaná jinak, než vypadá,
- ani jestli máte splněné podmínky pro použití exploitu.

Právě proto je nebezpečné používat ho mechanicky. Výstup typu:

```text
produkt X - RCE
```

není důkaz zranitelnosti. Je to jen kandidát, který je potřeba technicky zasadit do kontextu.

## Nejčastější chyby při použití

### Spuštění prvního nalezeného exploitu bez čtení podmínek

Na blogu je opakovaně vidět, že rozdíl mezi použitelným a nepoužitelným kandidátem leží v detailech:

- potřebuje login,
- funguje jen na určité minor verzi,
- očekává konkrétní endpoint,
- nebo cíl musí být skutečně dosažitelný.

### Záměna reference za hotový exploit chain

`Searchsploit` typicky pomůže najít vstupní bod, ne celý řetězec. Na [OpenAdminu](/openadmin) například pomohl k prvnímu command kontextu, ale skutečný dopad pak vznikl až přes reuse databázového hesla a SSH pivot.

### Ignorování výhod lokální kopie

Mnoho lidí používá `searchsploit` jen jako vyhledávač názvů. V praxi je ale velmi cenné, že exploit lze okamžitě otevřít, přečíst a upravit lokálně, bez přepínání do prohlížeče a bez slepého kopírování cizího payloadu.

## Nejčastější praktické scénáře

### Po identifikaci produktu chci rychle ověřit, zda existuje realistická veřejná cesta

To je [OpenAdmin](/openadmin), [Armageddon](/armageddon) nebo [Beep](/beep).

### Mám více výsledků a potřebuji vybrat ten, který sedí na konkrétní kontext

To je [Nibbles](/nibbles) a částečně [Lame](/lame).

### Exploit existuje, ale je potřeba ho zasadit do řetězce s tunelem nebo lokální dostupností

To je [Horizontall](/horizontall).

## Související články v projektu

- Rozbory: [OpenAdmin](/openadmin), [Nibbles](/nibbles), [Horizontall](/horizontall), [Lame](/lame), [Armageddon](/armageddon), [Beep](/beep)
- Techniky: [Legacy infrastruktura, kde banner prakticky rozhodne exploit](/techniky/legacy-infrastruktura-kde-banner-prakticky-rozhodne-exploit), [Lokálně dostupné služby po footholdu: localhost není boundary](/techniky/lokalne-dostupne-sluzby-po-footholdu-localhost-neni-boundary), [Nebezpečné uploady: polygloty, WAR deploy a plugin upload](/techniky/nebezpecne-uploady-polygloty-war-deploy-a-plugin-upload)

## Co si odnést do praxe

- `Searchsploit` je nejpraktičtější jako rychlý filtr exploitačních kandidátů po dobré enumeraci, ne jako náhrada za analýzu cíle.
- Největší hodnotu má ve chvíli, kdy už znáte produkt, verzi nebo aspoň velmi úzkou technologickou rodinu.
- Výsledek z něj je nejlepší chápat jako technickou hypotézu k ověření, ne jako důkaz, že cíl je automaticky zranitelný.
