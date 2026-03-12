---
layout: post
author: Tomáš Matějíček
title: "Multimaster"
date: 2020-12-18
tags: linux exploit privesc enumeration hackthebox
---

## Úvod a kontext

Multimaster je stroj z Hack The Box založený na řetězení slabin ve webové aplikaci a v Active Directory. Z didaktického hlediska je cenný hlavně tím, že neukazuje jeden dominantní exploit, ale postupný laterální pohyb přes několik účtů a delegovaných práv.

## Počáteční průzkum

### Vyhledání otevřených portů

Nejprve mapuji veřejně dostupné služby, protože právě z otevřených portů odvodím, které protokoly a aplikace má smysl zkoumat detailněji.
```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep ^[0-9] | cut -d "/" -f 1 | tr "\n" "," | sed s/,$//);echo $ports;nmap -p $ports -A -sC -sV -v $IP
```

## Získání přístupu

### Získání user flagu

User flag zde slouží hlavně jako potvrzení, že už mám běžný uživatelský kontext a mohu pokračovat v lokální analýze systému.

Řetězec začínal SQL injection ve webové aplikaci, přes kterou bylo možné získat první příkazový kontext na serveru. Z lokální enumerace pak vyplynulo, že na hostu běží VS Code nebo podobná debug vrstva, kterou šlo zneužít k dalšímu pivotu a shellu jako jiný uživatel.

Další krok už není o nové webové chybě, ale o práci s nalezenými tajemstvími. V dostupných write-upech se objevuje heslo uložené v DLL nebo konfiguračním artefaktu, které bylo znovu použito pro další doménový účet. Právě tenhle reuse otevřel cestu k účtu, ze kterého se dalo pokračovat v AD enumeraci.

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.

Rozhodující byla až oprávnění v Active Directory. Účet s právem `GenericWrite` nad účtem `jorden` šlo zneužít například vypnutím Kerberos pre-auth, následným AS-REP roastem a prolomením získaného hashe. Přihlášení jako `jorden` pak otevřelo cestu do skupiny s dostatečnými právy pro úpravu a spuštění služby běžící jako `SYSTEM`.

Z technického pohledu nejde o kernelovou eskalaci, ale o zneužití delegovaných práv v AD a servisního modelu Windows. To je přesně ten typ chyby, který v praxi často přežije i v prostředí bez známých RCE zranitelností.

## Shrnutí klíčových poznatků

- Rekonstruovat lze hlavně enumeraci a potvrzené artefakty, které určily další směr postupu.
- Klíčové bylo správně vyhodnotit konfiguraci, přístupové údaje nebo chování služeb, ne mechanicky doplňovat chybějící kroky.
- Tam, kde chybí celý řetězec k uživatelskému nebo root kontextu, zůstávají v textu jen technicky podložené části postupu.

## Co si odnést do praxe

- Pravidla `sudo` a jiné privilegované cesty mají být co nejmenší a bez možnosti ovlivnit příkaz, vstup nebo prostředí z neprivilegovaného kontextu.
- Včasná inventura verzí, konfigurací a interních automatizací výrazně snižuje prostor pro řetězení více menších slabin.
- Stejné techniky mají smysl pouze v laboratorním nebo jinak autorizovaném testovacím prostředí.
