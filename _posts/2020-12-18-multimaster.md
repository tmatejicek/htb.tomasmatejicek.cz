---
layout: post
author: Tomáš Matějíček
title: "Multimaster"
date: 2020-12-18
tags: windows exploit privesc enumeration hackthebox
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

Další krok už není o nové webové chybě, ale o práci s nalezenými tajemstvími. Objevilo se heslo uložené v DLL nebo konfiguračním artefaktu, které bylo znovu použito pro další doménový účet. Právě tenhle reuse otevřel cestu k účtu, ze kterého se dalo pokračovat v AD enumeraci.

## Eskalace oprávnění

### Získání root flagu

Tento krok ukazuje, jak se nalezená slabina nebo chyba v delegaci oprávnění mění v privilegovaný přístup.

Rozhodující byla až oprávnění v Active Directory. Účet s právem `GenericWrite` nad účtem `jorden` šlo zneužít například vypnutím Kerberos pre-auth, následným AS-REP roastem a prolomením získaného hashe. Přihlášení jako `jorden` pak otevřelo cestu do skupiny s dostatečnými právy pro úpravu a spuštění služby běžící jako `SYSTEM`.

Z technického pohledu nejde o kernelovou eskalaci, ale o zneužití delegovaných práv v AD a servisního modelu Windows. To je přesně ten typ chyby, který v praxi často přežije i v prostředí bez známých RCE zranitelností.

## Shrnutí klíčových poznatků

- První skutečně užitečný závěr plynul z toho, jak do sebe zapadly Kerberos, Active Directory a SQL injection.
- User fáze se opírala o stabilní uživatelský přístup, takže přístup byl reprodukovatelný a ne jen jednorázový.
- Finální kontrolu nad systémem otevřela až mechanika typu lokální enumerace po získání shellu.

## Co si odnést do praxe

- Pokud se zanedbá oblast Kerberos, Active Directory a SQL injection, vznikne stejný typ vstupu jako tady. Vstupy do databázových dotazů musí být parametrizované a oddělené od aplikační logiky; SQL injection stále patří mezi nejrychlejší cesty k datům i dalšímu pivota.
- Jakmile útočník ověří stabilní uživatelský přístup, je potřeba počítat s dlouhodobým přístupem. Jakmile se v prostředí objeví použitelný klíč, heslo nebo token, je potřeba předpokládat okamžitý pivot na stabilní shell; obrana proto stojí na segmentaci a oddělení přístupů mezi službami.
- Stejně důležitá je i obrana proti mechanice lokální enumerace po získání shellu. Root/admin část obvykle nepadá na nové CVE, ale na lokální delegaci práv, reuse tajemství nebo pomocném skriptu; právě tyto mechaniky je potřeba po footholdu auditovat nejdřív.
