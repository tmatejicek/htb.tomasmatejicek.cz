---
layout: post
author: Tomáš Matějíček
title: "Impacket pro AD enumeraci a první identity"
date: 2020-11-26
tags: nastroje active-directory windows kerberos ldap smb enumeration
---
## Úvod a kontext

Část utilit z `Impacketu` dává smysl v úplně rané AD fázi, kdy ještě nemáte shell ani stabilní účet, ale už potřebujete převést první doménové indicie do konkrétní identity hypotézy. Právě sem patří hlavně `GetADUsers.py`, `GetNPUsers.py` a `lookupsid.py`. Neřeší vzdálené vykonávání ani dump tajemství. Řeší první otázku: kdo v doméně existuje a který účet stojí za další pokus.

Na [Forestu](/forest) právě tahle rodina utilit otevřela celý řetězec. Na [Heistu](/heist) pomohl `lookupsid.py` převést jediné nalezené heslo do seznamu skutečných účtů. Na [Monteverde](/monteverde) nebo [Sauně](/sauna) se stejný princip opakuje v jiné podobě: první úspěch nevzniká z exploitu služby, ale z chytré práce s identitami.

## Co tyhle utility v praxi řeší

Tahle rodina Impacket utilit pomáhá zodpovědět tři základní otázky:

- kdo v doméně nebo na hostu vůbec existuje,
- který účet má vypnutý Kerberos pre-auth,
- a jak vypadá SID prostor nebo naming okolí cíle.

Právě tyhle odpovědi pak rozhodují, zda půjdete do AS-REP roastu, password spraye, reuse nebo další AD enumerace.

## Nejčastější scénáře využití v tomto projektu

### `GetADUsers.py`: získání prostoru účtů

Na [Forestu](/forest) bylo první důležité zjištění prosté: doména vrací seznam účtů i bez plnohodnotného footholdu.

```bash
GetADUsers.py -all -dc-ip 10.10.10.161 htb.local/
```

Tento krok je prakticky cenný proto, že z neurčitého "nějaká doména běží" udělá konkrétní seznam kandidátů. Teprve s nimi dává smysl dál pracovat.

Na [Monteverde](/monteverde) je ten samý princip vidět i bez roastu. Získání seznamu uživatelů dává kontext pro úzký spray a pozdější SMB/LDAP práci.

### `GetNPUsers.py`: ověření AS-REP roast kandidátů

Jakmile máte prostor uživatelů, přichází otázka, jestli některý z nich nemá vypnutý Kerberos pre-auth.

```bash
GetNPUsers.py -format hashcat -usersfile users.txt -outputfile hashes.asreproast -dc-ip $IP htb.local/
```

Na [Forestu](/forest) tím vznikl hash účtu `svc-alfresco`. Na [Sauně](/sauna) se přes stejnou logiku podařilo potvrdit `FSmith`. Tenhle nástroj je praktický hlavně tím, že rychle oddělí obecný seznam jmen od účtů, které už dávají offline crack materiál.

Širší logiku téhle fáze rozebírám i v článku [Kerberos útoky z nulového přístupu: AS-REP roast a práce s kandidátními uživateli](/techniky/kerberos-utoky-z-nuloveho-pristupu-as-rep-roast-a-prace-s-kandidatnimi-uzivateli).

### `lookupsid.py`: převod jednoho hesla na seznam skutečných účtů

Na [Heistu](/heist) problém nebyl v nedostatku hesel, ale v nedostatku uživatelských jmen. `lookupsid.py` tu dává přesně ten typ odpovědi, který změní směr práce:

```bash
python lookupsid.py hazard:"stealth1agent"@$IP
```

Najednou máte `Hazard`, `support`, `Chase` a další účty, proti nimž lze hesla vyzkoušet systematicky. Praktická hodnota je tedy vysoká i v momentu, kdy ještě nejdete po Kerberosu, ale jen potřebujete převést jednu credential stopu do doménového kontextu.

Stejný pattern je vidět i na [ServMonu](/servmon), kde `lookupsid.py` potvrdil, že jména z uniklého souboru odpovídají reálným účtům.

## Kdy jsou tyto utility nejlepší volba

Tahle skupina nástrojů dává největší smysl tehdy, když:

- jde o Windows nebo AD prostředí,
- ještě nemáte stabilní shell,
- potřebujete validní uživatelský prostor,
- nebo chcete ověřit, zda existuje AS-REP roast nebo další identity cesta.

Naopak nedávají smysl jako náhrada za hlubší LDAP audit nebo privilegovanou AD analýzu. Jakmile už máte foothold a řešíte ACL, relay nebo gMSA, navazují jiné utility a jiné články.

## Nejčastější chyby v praxi

### Příliš rychlý přechod ke crackingu

Nejdřív je potřeba vědět, které účty existují a který z nich je relevantní. Když přeskočíte tuhle fázi, budete crackovat nebo sprayovat naslepo.

### Míchání tří různých cílů dohromady

`GetADUsers.py`, `GetNPUsers.py` a `lookupsid.py` vypadají příbuzně, ale prakticky řeší různé otázky:

- seznam účtů,
- roast kandidáty,
- a SID/identitní prostor.

Nejlépe fungují, když víte, na kterou z těchto otázek právě odpovídáte.

### Zaměnění identity enumerace za foothold

Výstup ještě není přístup. Je to materiál pro další krok: spray, roast, reuse nebo WinRM/SMB validaci.

## Související články v projektu

- Rozbory: [Forest](/forest), [Heist](/heist), [Monteverde](/monteverde), [Sauna](/sauna), [ServMon](/servmon)
- Techniky: [Kerberos útoky z nulového přístupu: AS-REP roast a práce s kandidátními uživateli](/techniky/kerberos-utoky-z-nuloveho-pristupu-as-rep-roast-a-prace-s-kandidatnimi-uzivateli), [Password reuse a rozpad hranic mezi aplikací, SSH, WinRM a admin nástroji](/techniky/password-reuse-a-rozpad-hranic-mezi-aplikaci-ssh-winrm-a-admin-nastroji)
- Nástroje: [Impacket](/nastroje/impacket), [Kerbrute](/nastroje/kerbrute), [ldapsearch](/nastroje/ldapsearch), [enum4linux](/nastroje/enum4linux)

## Co si odnést do praxe

- Tato skupina Impacket utilit je nejcennější na začátku AD řetězce, kdy ještě stavíte první identitní hypotézu.
- Prakticky řeší tři věci: kdo existuje, kdo je roastovatelný a jak vypadá SID prostor cíle.
- Největší hodnotu mají tehdy, když z prvního hesla nebo prvního jména udělají smysluplný další krok místo náhodného spraye.
