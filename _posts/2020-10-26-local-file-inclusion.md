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

- LFI je potřeba odlišovat od vzdáleného include nebo od přímého vzdáleného spuštění kódu; konkrétní dopad závisí na tom, jak aplikace s načítaným souborem dál pracuje.
- Starší techniky jako `null byte injection` nebo `path truncation` dávají smysl jen u historických verzí PHP; bez této podmínky by šlo o technicky nepřesný závěr.
- Už samotná možnost číst lokální soubory je závažná, protože často odhalí konfigurace, klíče nebo další tajemství potřebná pro navazující útok.

## Co si odnést do praxe

- Uživatelský vstup nesmí přímo určovat cestu k souboru, který server načítá; bezpečnější je práce s pevně definovaným seznamem povolených hodnot.
- Ochranu je potřeba stavět i na omezení přístupových práv procesu a na pečlivém oddělení dat od kódu, ne jen na filtrování řetězců v URL.
- Stejné techniky mají smysl pouze v laboratorním nebo jinak autorizovaném testovacím prostředí.
