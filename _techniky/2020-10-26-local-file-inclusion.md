---
layout: post
author: Tomáš Matějíček
title: "LFI - Local File Inclusion"
date: 2020-10-26
tags: lfi php
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

- LFI samo o sobě neznamená totéž co RCE; zásadní je, zda aplikace soubor jen čte, nebo ho předává interpretru přes `include`.
- Oba uvedené scénáře ukazují, že dopad stejné chyby se mění podle toho, jak aplikace skládá výslednou cestu a jakou verzi PHP provozuje.
- `Path Truncation` i `Null Byte Injection` jsou historické techniky navázané na starší verze PHP, takže je potřeba je chápat v kontextu konkrétní platformy, ne jako obecně platný trik.

## Co si odnést do praxe

- Uživatelský vstup se nesmí přímo dostat do `include`, `require` ani podobných funkcí. Bezpečný přístup je whitelist logických identifikátorů a mapování na pevně definované soubory.
- LFI je potřeba hodnotit jako únik citlivých lokálních dat, ne jako „jen čtení textu“. Konfigurace, session soubory, logy nebo klíče často otevřou další fázi útoku i bez přímého RCE.
- Staré obchvaty jako `Null Byte Injection` nebo `Path Truncation` dnes slouží hlavně jako připomínka, že bezpečnostní vlastnosti jazyka se mění v čase. Obrana proto nesmí stát na domněnkách o chování dávno neudržované verze runtime.
