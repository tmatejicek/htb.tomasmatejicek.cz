---
layout: post
author: Tomáš Matějíček
title: "WHOIS, DNS a AXFR jako reálná útočná plocha"
date: 2021-01-09
tags: dns whois axfr recon infrastructure
---
## Úvod a kontext

WHOIS a DNS se často berou jen jako recon kanál. Něco, co pomůže zmapovat jména hostů, ale samo o sobě nepředstavuje skutečnou bezpečnostní hranici. To je zkratka. Jakmile WHOIS dotaz končí v databázi, DNS server vrací víc dat, než má, nebo zónový přenos vydá interní hosty a slabší vedlejší služby, nejde už o pasivní mapování. Jde o samostatnou útokovou plochu.

Právě to je důležité držet v hlavě. Informace z WHOIS nebo DNS často nejsou cíl samy o sobě. Jejich hodnota je v tom, že z nich velmi rychle vznikne další kompromitovatelný systém: nový vhost, interní rozsah, zákaznický host nebo služba, která z veřejného webu vůbec není vidět.

## Kde se z „obyčejného reconu“ stává chyba

Bezpečnostně nejrizikovější jsou hlavně tyto situace:

- WHOIS endpoint zpracovává vstup nebezpečně a vydá víc dat, než měl,
- DNS zóna vrací hosty, které měly být dostupné jen interně,
- AXFR je povolený i pro nedůvěryhodné klienty,
- veřejná zóna nese technická jména, která útočníka navedou na slabší vrstvu,
- DNS nebo proxy konfigurace slouží zároveň jako dokumentace interní topologie.

To je důvod, proč je chybné mluvit o WHOIS nebo DNS jen jako o „pomocném zdroji indicií“. V řadě prostředí jsou to plnohodnotné vstupní body do architektury.

## Příklad z praxe: Scavenger

Scavenger je velmi čistý řetězec, kde WHOIS ani AXFR nejsou vedlejší detail, ale skutečný první zlom.

### WHOIS jako vstup do databázového backendu

Na portu `43` běžela WHOIS služba. Nevracela jen běžný text, ale i stopu po databázovém zpracování. Stačilo poslat rozbitý vstup a odpověď vydala seznam dalších hostovaných domén:

```bash
whois -h $IP "%') #"
```

```text
SUPERSECHOSTING.HTB
JUSTANOTHERBLOG.HTB
PWNHATS.HTB
RENTAHACKER.HTB
```

To je zásadní moment. Místo jednoho cíle vzniká celá sada dalších jmen a s nimi i šance najít slabší aplikaci na vedlejším hostu.

### AXFR jako most k dalšímu hostu

Následný zone transfer na `RENTAHACKER.HTB` vydal konkrétní virtuální host:

```bash
dig AXFR RENTAHACKER.HTB @SUPERSECHOSTING.HTB
```

Praktickou roli nízkoúrovňových DNS klientů pro podobné ověřování rozebírám i v článku [host a dig](/nastroje/host-a-dig).

```text
sec03.rentahacker.htb
```

Teprve tenhle host pak otevřel další řetězec: `shell.php`, přístup k poště, FTP, pcap a další účet. Bez WHOIS a AXFR by celá tahle větev zůstala mimo zorné pole.

Scavenger je tedy dobrá připomínka, že:

- WHOIS může být zranitelná aplikace,
- AXFR může být datový únik,
- oba kroky dohromady mohou vést ke kompromitaci úplně jiné služby.

## Příklad z praxe: Tentacle

Tentacle stojí víc na DNS a infrastrukturní konfiguraci než na klasickém WHOIS, ale princip je podobný. Veřejně viditelná vrstva sama o sobě nestačila. Teprve DNS data a následně `wpad.dat` vydaly další část vnitřní topologie. Síťovou návaznost WPADu, proxy a druhého rozsahu rozebírám detailněji i v článku [Síťová topologie jako leak: WPAD, Squid, FXP a interní mapování](/techniky/sitova-topologie-jako-leak-wpad-squid-fxp-a-interni-mapovani).

### DNS jako mapa interních jmen

Enumerace `realcorp.htb` ukázala hosty jako:

- `ns.realcorp.htb`
- `proxy.realcorp.htb`
- `wpad.realcorp.htb`

To samo ještě neotevře shell. Otevře to ale správnou hypotézu: prostředí má proxy vrstvu, interní DNS logiku a další segmenty, které zvenku nejdou vidět.

### WPAD a druhý interní rozsah

Po dosažení `wpad.dat` se ukázal další neveřejný rozsah:

```javascript
if (isInNet(dnsResolve(host), "10.241.251.0", "255.255.255.0"))
    return "DIRECT";
```

Tím se z „konfiguračního souboru pro browser“ stala síťová dokumentace. Následný scan druhého segmentu dovedl k internímu OpenSMTPD hostu a ten už vedl k footholdu.

Tentacle tak dobře ukazuje, že DNS a pomocné síťové konfigurace nejsou jen pasivní metadata. Často jsou to přesné instrukce, kudy pokračovat dál.

## Typické způsoby dopadu

V praxi se z WHOIS, DNS a AXFR nejčastěji stává:

### 1. Leak interních nebo slabších hostname

Útočník nemusí prolomit hlavní aplikaci. Stačí, že DNS nebo WHOIS prozradí vedlejší host, který je hůř chráněný.

### 2. Leak topologie a síťových zón

WPAD, proxy jména, interní resolver nebo záznamy v zóně často prozradí, že existuje další segment, další proxy nebo další management vrstva.

### 3. Přechod na jinou technologii

Z hlavního webu se útok může přesunout na:

- FTP,
- SMTP,
- interní admin vhost,
- zákaznickou aplikaci,
- monitorovací nebo integrační službu.

Právě tento přesun dělá z infrastrukturních dat skutečnou attack surface.

## Na co se při auditu zaměřit

### WHOIS vrstva

- je to jen statický server, nebo aplikace nad databází,
- filtruje a escapuje vstup,
- vrací jen veřejně určená data,
- nevydává interní jména, chybové hlášky nebo technické detaily hostingu.

### DNS vrstva

- které zóny jsou veřejné a které interní,
- kdo smí dělat AXFR,
- jaká jména se ve veřejné zóně objevují,
- neobsahují záznamy názvy, které zbytečně odhalují roli hostu.

### Propojené služby

- nevede DNS rovnou na administrační nebo zákaznický vhost,
- neukazují proxy nebo WPAD soubory na neveřejné rozsahy,
- nejsou interní názvy použitelné i z kompromitovaného veřejného hostu přes proxy nebo tunel.

## Obrana a hardening

### 1. Omezit AXFR jen na skutečné sekundární servery

Zone transfer je legitimní provozní mechanika, ale nesmí být otevřený komukoliv, kdo se zeptá.

### 2. Oddělit veřejná a interní jména

Veřejná zóna nemá sloužit jako katalog interní infrastruktury. Čím méně technických detailů v ní je, tím menší je šance, že navede útočníka na vedlejší, slabší vrstvu.

### 3. Auditovat WHOIS jako běžnou aplikaci

Pokud WHOIS běží nad databází nebo zákaznickým backendem, musí procházet stejným security review jako jakýkoliv jiný webový endpoint.

### 4. Revidovat WPAD a proxy konfigurace

Soubor, který má klientovi usnadnit síťový provoz, často zároveň prozrazuje interní rozsahy a pravidla přístupu. Nesmí fungovat jako neúmyslná dokumentace celé topologie.

### 5. Hodnotit infrastrukturní leaky podle dalšího kroku

Nejde jen o to, jestli unikl hostname. Jde o to, zda daný hostname vede k:

- novému vhostu,
- nové technologii,
- interní službě,
- nebo jiné zóně důvěry.

Právě to určuje skutečný dopad.

## Shrnutí klíčových poznatků

- WHOIS, DNS a AXFR nejsou jen pasivní recon. V mnoha prostředích tvoří samostatnou útokovou plochu.
- Největší škodu nepůsobí samotná informace, ale to, že navede útočníka k jinému, slabšímu systému nebo neveřejné síťové vrstvě.
- Scavenger ukazuje přímý řetězec WHOIS -> AXFR -> nový vhost -> další kompromitace.
- Tentacle ukazuje, že i DNS a WPAD mohou fungovat jako dokumentace vnitřní topologie, která dovede útočníka až k interním službám.

## Co si odnést do praxe

- Když hodnotíš WHOIS nebo DNS, neřeš jen „co jde vypsat“. Řeš hlavně, kam tato informace útočníka pošle dál.
- AXFR, veřejná proxy konfigurace a technicky popsané hostname bývají mnohem cennější, než na první pohled vypadají.
- Infrastrukturní metadata jsou součást útokové plochy. Pokud z nich útočník vyčte architekturu, zóny a vedlejší hosty, už nejsou jen recon kanálem.
