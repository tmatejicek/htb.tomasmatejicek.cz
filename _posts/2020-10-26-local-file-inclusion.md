---
layout: post
author: Tomáš Matějíček
title: "LFI - Local File Inclusion"
date: 2020-10-26
tags: LFI PHP
---
## Úvod a kontext

Local File Inclusion (LFI) je chyba v aplikaci, která dovolí ovlivnit, jaký lokální soubor server načte do zpracování. Sama o sobě neznamená automaticky vzdálené spuštění kódu, ale může vést k úniku citlivých dat, k obejití očekávaného toku aplikace a v některých kombinacích i k dalšímu zneužití.

Rozhodující je vždy kontext: jiný dopad bude mít prosté čtení souborů, jiný načítání souboru do interpretu a jiný situace, kdy útočník dokáže ovlivnit i obsah načítaného souboru. Níže proto rozlišuji dva základní scénáře.

## Scénář č. 1

Zranitelný kód:
```php
<?php include($_GET['stranka']); ?>
```
Vypsání obsahu passwd
```
?stranka=/etc/passwd
```

## Scénář č. 2

Zranitelný kód:
```php
<?php include($_GET['stranka'].".php"); ?>
```

**Vypsání obsahu passwd díky zranitelnosti Path Truncation (byla opravena ve verzi PHP 5.3.0)**
Zranitelnosti Path Truncation využívá toho, že starší verze PHP mají limit na délku cesty omezen na 4096 bytů a to co je navíc jednoduše oříznou.
```
?stranka=../../../../../../  [.....]  /../../../../../etc/passwd
```

**Vypsání obsahu passwd díky zranitelnosti Null Byte Injection (byla opravena ve verzi PHP 5.3.4)**
Zranitelnosti Null Byte Injection využívá toho, že starší verze PHP umožňují ukončit řetězec řídícím znakem null a to co je navíc jednoduše oříznou.
```
?stranka=/etc/passwd%00
```

## Shrnutí klíčových poznatků

- Z hlediska rozhodování bylo nejdůležitější správně přečíst vazbu mezi local file inclusion a webová aplikace v PHP.
- K uživatelskému kontextu vedl konkrétní a ověřitelný krok: stabilní uživatelský přístup.
- Poslední část ukazuje, že po získání shellu rozhoduje hlavně to, jakou roli hraje lokální enumerace po získání shellu.

## Co si odnést do praxe

- Pokud se zanedbá oblast local file inclusion a webová aplikace v PHP, vznikne stejný typ vstupu jako tady. LFI je potřeba brát jako únik citlivých souborů, ne jen jako čtení textu; konfigurace, klíče a šablony často stačí k plnohodnotnému přístupu.
- Jakmile útočník ověří stabilní uživatelský přístup, je potřeba počítat s dlouhodobým přístupem. Jakmile se v prostředí objeví použitelný klíč, heslo nebo token, je potřeba předpokládat okamžitý pivot na stabilní shell; obrana proto stojí na segmentaci a oddělení přístupů mezi službami.
- Stejně důležitá je i obrana proti mechanice lokální enumerace po získání shellu. Root/admin část obvykle nepadá na nové CVE, ale na lokální delegaci práv, reuse tajemství nebo pomocném skriptu; právě tyto mechaniky je potřeba po footholdu auditovat nejdřív.
